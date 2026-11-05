# 私有化部署与运维排障（Deploy & Ops）

适用：客户内网私有化 Dify（CentOS / RedHat / 统信 UOS / 麒麟 + Docker Compose）。
原则：先判定故障属于"平台层"还是"DSL 业务层"。报错落在容器、网络、代理上的，一律交运维；报错落在变量、节点连线、提示词上的，才改 DSL。

## 1. 故障定位分流

| 报错 / 表现 | 归属层 | 首查对象 |
|---|---|---|
| `[Errno 113] No route to host` | 平台层 | sandbox 容器、`.env`、firewalld |
| 节点一直转圈 `running` | 平台层（多数） | sandbox 是否可达 |
| 本地上传正常，粘贴链接报错 | 平台层 | ssrf_proxy 白名单 |
| LLM 输出为空 / 提示缺变量 | DSL 层 | 上下文与 USER 消息是否绑变量 |
| HTTP 请求头报错 | DSL 层 | 变量是否误选 `text` |
| 同一 DSL 在另一环境行为不同 | 版本层 | 平台版本能力差异 |

## 2. 沙箱（Sandbox）不可达

现象原文：

```text
Failed to execute code... check if the sandbox service is running.
( Error: [Errno 113] No route to host )
```

原因：`代码执行（Python/JS）` 与 `文档提取器` 都在独立 Sandbox 容器内运行。Dify 主程序找不到沙箱地址，后台任务死等响应，前端持续 `running` 直到超时。

### 2.1 检查沙箱容器是否存活

```bash
docker ps | grep sandbox
```

判断与处理：
- 无任何输出 → 沙箱未启动。进 Dify 的 `docker` 目录，确认 `docker-compose.yaml` 含 sandbox 服务，然后：
  ```bash
  docker compose up -d sandbox
  ```
- 状态为 `Restarting (xx)` 或异常退出 → 查日志，多为端口冲突或内存不足：
  ```bash
  docker logs docker-sandbox-1
  ```

### 2.2 检查 `.env` 网络地址（最高频错因）

打开部署目录下 `.env`，搜索 `SANDBOX`，核对：

```env
# 正确：指向 docker-compose 内部服务名
SANDBOX_HOST=sandbox
SANDBOX_PORT=8190
```

错误写法（均导致 `No route to host`）：
- `SANDBOX_HOST=127.0.0.1`：容器内的 127.0.0.1 找不到宿主机上的沙箱。
- `SANDBOX_HOST=localhost`：同上。
- `SANDBOX_HOST=<宿主机内网 IP>`：容器网络隔离，同样不可达。

改完后重启：

```bash
docker compose restart api worker
```

### 2.3 检查 firewalld 是否拦截 Docker 网桥

`[Errno 113]` 在企业级 Linux 上常由 `firewalld` 丢弃容器间转发导致。

```bash
# 临时关闭验证
sudo systemctl stop firewalld

# 或放通 docker 网桥
sudo firewall-cmd --zone=trusted --add-interface=docker0 --permanent
sudo firewall-cmd --reload
```

判定：关闭防火墙后工作流立即跑通，即为防火墙切断 Dify 与 Sandbox 通信。验证后按合规要求恢复防火墙并改用网桥放通规则。

## 3. SSRF 拦截：本地上传正常、粘贴链接报错

现象：文档提取器用 `local_file` 正常，改用 `remote_url` 报错。请求体形如：

```json
{
  "type": "document",
  "transfer_method": "remote_url",
  "remote_url": "http://192.168.10.161:9000/files/1b998650-.../file-preview?timestamp=...&sign=..."
}
```

根因：
- `local_file` 由浏览器直传内部存储桶（MinIO/S3），提取时走容器卷或存储 SDK，不经外部网络，畅通。
- `remote_url` 要求 `worker` / `api` 容器作为客户端主动 HTTP 下载。目标 `192.168.x.x` 属 RFC 1918 私网段，Dify 内置的 `ssrf_proxy`（基于 Squid）默认拒绝访问 `192.168.x.x`、`10.x.x.x`、`172.16.x.x`，直接掐断连接（403 或连接拒绝），文档提取器拿不到文件。

### 3.1 方案一：`.env` 关闭 SSRF 代理或加白名单（最快）

```env
# 默认配置
SSRF_PROXY_HTTP_URL=http://ssrf_proxy:3128
SSRF_PROXY_HTTPS_URL=http://ssrf_proxy:3128

# 内网环境：注释掉或留空
# SSRF_PROXY_HTTP_URL=
# SSRF_PROXY_HTTPS_URL=

# 或保留代理但放行指定内网地址
NO_PROXY=192.168.10.161,127.0.0.1,localhost
```

生效：

```bash
docker compose down
docker compose up -d
```

### 3.2 方案二：改 `ssrf_proxy` 白名单（保留外网防护）

编辑 `docker/volumes/ssrf_proxy/squid.conf`，找到 `http_access deny to_localnet`，在其正上方插入放行：

```conf
acl my_internal_server dst 192.168.10.161
http_access allow my_internal_server

http_access deny to_localnet
```

重启代理：

```bash
docker compose restart ssrf_proxy
```

顺序要求：`allow` 规则必须在 `deny to_localnet` 之前，否则不生效。

### 3.3 改完仍失败：验证容器到宿主机端口连通性

```bash
docker compose exec worker curl -I "http://192.168.10.161:9000/files/1b998650-e3d1-4923-9249-68b1fd8a1635/file-preview?..."
```

| 返回 | 结论 | 处理 |
|---|---|---|
| `HTTP/1.1 200 OK` | 网络已通 | 文档提取器可正常解析 |
| `Connection refused` | 宿主机防火墙拦了 9000 端口在 Docker 内的访问 | 放通 9000 端口或关闭 `firewalld` |

### 3.4 规避做法

演示与交付场景优先用"文件上传（File Upload）"：文件以二进制流进容器，100% 绕开网络拦截与 SSRF 判定。不要用粘贴 MinIO 内网链接的方式。

## 4. DSL 跨环境导入后的异常

### 4.1 节点卡 `running`

排查顺序：
1. 导入后的环境是否有沙箱容器且可达（见第 2 节）。
2. 报错是否落在代码执行 / 文档提取器节点上——这两个节点强依赖沙箱。
3. 确认与 DSL 逻辑无关：同一 DSL 在原环境正常、仅新环境卡住，即为环境层问题。

### 4.2 LLM 节点报错或输出为空

排查顺序：
1. 检查 LLM 节点顶部 `上下文` 是否仍为灰色 `{x} 设置变量值`（空）。空上下文即无输入。
2. 检查是否配置了 `USER` 消息框。只配 SYSTEM（规则）而无 USER（待处理文本），模型看得到规则、看不到内容。
3. 检查变量气泡是否在导入后失效：需用 `{x}` 重新插入 `文档提取器.text` 等上游输出，不要手打变量名。
4. 检查是否有中文/换行混入 HTTP 请求头变量（见 4.3）。

### 4.3 HTTP 请求头报错

现象：下游 HTTP 节点用 `X-JSON-Path` 之类的请求头传值时报错。

根因：上游 LLM 开启了结构化输出，若在变量下拉里选了第一项 `text`，传给请求头的是带花括号和回车换行的完整 JSON，如 `{\n "json_path": "$.flights[...]"\n}`。请求头遇换行与外层花括号即崩。

解法：
1. 下拉菜单不要选 `{x} text`。
2. 选第三行 `{x} structured_output`，点右侧小箭头 `>` 展开子级。
3. 选中子字段（如 `{x} json_path (String)`）。输入框显示为 `LLM: 条件转 JSONPATH / structured_output.json_path` 即为正确。

附带检查：接口定义了 `apiKey in: header`（如 `X-Access-Key`）时，到【集成 → Swagger API 作为工具 → 对应工具 → 鉴权方法】填有效密钥，否则返回 `401` / `403`。

### 4.4 节点连线顺序错乱

导入或改造后必查因果顺序，典型错误：

```text
错误：【文档切分】→【文档提取器】→【LLM】
正确：【开始（上传 Docx）】→【文档提取器】→【文本切分】→【LLM】
```

判定：先提取出文本，才可能切分。顺序颠倒必然失败。

## 5. 版本差异导致的行为不一致

同一 DSL 在客户环境与自有环境表现不同时，先对齐版本，不要改 DSL。

| 能力 | 1.0.3 / 1.3.1 时代 | 1.17.1 |
|---|---|---|
| 会话变量 | 基本没有，多轮跨轮状态难传递 | 支持会话变量 + 【变量赋值】节点（如 `current_mode`、`weakness_log`） |
| 逻辑节点 | 线性流转为主 | 【迭代/循环】、【参数提取器】、多级 `ELIF`、`AND/OR`、包含/正则/为空判断 |
| 节点容错 | 节点失败整条线死机 | 【失败时重试（最大重试次数/重试间隔）】+【异常处理】分支 |
| 工具接入 | 写死在底层代码，加工具需改后端重打包 | Marketplace 插件化、沙箱化，在线安装（博查、百度 OCR、安思派联网等） |
| 代码沙箱 | 隔离与超时控制弱，易全局内存泄漏 | 沙箱隔离、超时控制、内网私有包调用更稳 |
| 知识检索 | 以纯向量检索为主，专业术语/工号/科室名易搜不到 | 向量 + BM25 混合检索，原生 Rerank（BGE-Reranker、Cohere 等），先粗筛 20 条再重排 |
| 分段解析 | 基础分段 | Q&A 分段（CSV/Excel 直接拆问答对）、父子分块（Parent-Child Chunking） |
| 多模态 | 音频只能在客户端转写，画布无音频节点 | 【Audio / Speech To Text】工具；开始节点可上传文档/图片/音频；LLM 节点可塞图片变量（GPT-4o、Qwen-VL） |
| 可观测性 | 日志粗 | 运行日志显示每节点耗时（如 `50.171 ms`）与 Token（如 `54 tokens`） |

处置规则：
- 客户环境为 1.0.3 / 1.3.1 → 不要在 DSL 里使用会话变量、迭代节点、插件工具、音频节点，否则导入即失效。
- 客户环境为 1.0.3 且必须用文档提取器 → 强制走本地上传，不依赖 `remote_url`。
- 需要定位 token 消耗或节点耗时 → 需 1.17.1 的日志面板，低版本无此能力。

## 6. 文档提取器的限制与替代

原生限制：
- 依赖 Sandbox 容器，沙箱不通即整节点失败。
- `remote_url` 受 SSRF 代理限制。
- 单次提取文本量受模型上下文限制，实践上单份 2~3 万字内可被 `qwen-3.5`、`deepseek` 单次完整吃下；超长文档必须切分。

超长文档的正确链路（不能只有切分，必须配迭代）：

```text
【开始】→【文档提取器】→【文本切分（Text Splitter）】→【迭代节点】
        └ 内部放【LLM】（对每段跑同一套审查/提取提示词）
        →【代码节点（汇总）】→【结束】
```

替代插件（在 Marketplace / 工具中安装后，删掉原生文档提取器，拖入插件节点，入参仍传 `开始 / file_doc`，输出 `text` 接下游 LLM）：
- `MinerU`：面向 PDF/Word 复杂排版，表格、公式、段落无损提取为干净 Markdown。
- `Apache Tika`：企业级通用解析，支持 1000+ 格式，容错率高。
- `Document Extractor Pro`：插件市场自带增强版提取器。

选择判断：
- 纯文本型 Word、篇幅可控 → 原生文档提取器即可。
- 复杂排版 PDF、含表格公式 → MinerU。
- 格式杂、数量多、容错优先 → Apache Tika。

## 7. 交付给运维的话术模板

```text
老师您好，该报错来自私有化环境的 Dify 沙箱服务网络不可达
（报错信息：[Errno 113] No route to host），导致依赖沙箱的
代码执行 / 文档提取无法运行，前端一直挂起等待。该问题属宿主机
Docker 容器间网络通信或配置问题，与 DSL 业务流本身无关。

请协助排查三项：
1. docker-sandbox 容器是否正常运行在 8190 端口；
2. .env 中 SANDBOX_HOST 是否配置为内部服务名 sandbox；
3. 系统 firewalld 是否拦截 Docker 容器间转发（可临时关闭验证）。

沙箱恢复后流程即恢复正常。
```

## 8. 排障检查清单

- [ ] `docker ps | grep sandbox` 有输出且状态非 `Restarting`。
- [ ] `.env` 中 `SANDBOX_HOST=sandbox`、`SANDBOX_PORT=8190`。
- [ ] `firewalld` 已放通 `docker0` 或已临时关闭验证。
- [ ] 内网 `remote_url` 已加入 `NO_PROXY` 或 squid 白名单，且 `allow` 在 `deny to_localnet` 之前。
- [ ] `docker compose exec worker curl -I <目标 URL>` 返回 `200 OK`。
- [ ] LLM 节点上下文与 USER 消息均已绑定变量气泡。
- [ ] HTTP 请求头变量取自 `structured_output.xxx` 而非 `text`。
- [ ] 节点因果顺序为 提取 → 切分 → 迭代 → 汇总。
- [ ] 客户环境版本已确认，DSL 未使用该版本不具备的能力。
