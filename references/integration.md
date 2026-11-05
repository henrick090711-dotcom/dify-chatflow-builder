# 外部系统集成（HTTP / 代码清洗 / Swagger / MCP / 飞书）

适用：对话流或工作流需要读写客户既有业务系统（HIS、BOSS、CRM、SaaS），或需要向飞书/企微推送消息。
读取顺序：先读「一、三段式架构」定骨架，再按接入方式跳到「二」「五」「六」，报错查「三」，联调查「八」。

---

## 一、外部接口的三段式架构

### 1.1 标准骨架

```text
【参数提取 LLM】 → 【请求（HTTP 节点 / 工具节点）】 → 【数据清洗（代码执行）】 → 【播报 LLM】
```

| 段 | 节点类型 | 职责 | 输出 |
| :--- | :--- | :--- | :--- |
| 参数提取 | LLM | 把用户口语抽成接口要求的参数；只输出纯净 JSON 或纯文本 | `{"keyword":"眼科"}` |
| 请求 | HTTP 请求 / 工具节点 | 拼装 Header/Body，发起真实调用 | 原始 `body`（可能是嵌套字符串、HTML、null） |
| 数据清洗 | 代码执行（Python3） | 剥壳、字段映射、空值与异常兜底、裁掉无用字段 | 几十字的干净文本 |
| 播报 | LLM | 按用户当前问题挑字段，转自然语言 | 面向用户的话术 |

### 1.2 禁止让 LLM 直接吃原始返回，理由只有三条

| 理由 | 具体表现 |
| :--- | :--- |
| Token 成本 | 传统 HIS 单条记录几十个无用字段（`TingZhenBZ`、`ZhenLiaoFei` 等），全量塞给模型一次对话消耗数千 Token |
| 幻觉 | 模型算数差，会把「上午余号」和「下午余号」看串行，把余号 0 读成还有号 |
| 崩溃 | 接口偶发返回 `null`、超时、HTML 报错页，直接喂给模型会整链报错；代码里有 `try/except` 可以温柔兜底 |

### 1.3 什么时候可以省掉清洗节点

同时满足以下三条才可省：

1. 返回体是标准扁平 JSON，无「字符串套字符串」、无反斜杠转义、无 `-1` 之类的哨兵值。
2. 字段总量小（单条记录可读字段 ≤ 10 个），全量喂模型不吃 Token。
3. 数值字段不需要精确计算或阈值判断，模型读错也不影响业务。

任一条不满足，保留代码节点。省掉清洗节点时，把 `{{HTTP请求.body}}` 直接放进播报 LLM 的 SYSTEM，并在提示词里写明「code 不等于 1 或数据为空时回复兜底话术」「禁止输出 JSON 与英文字段名」。

### 1.4 入参侧也要一个代码节点（高频遗漏）

当接口要求 `data` 是「字符串化的 JSON」（文档写成 `{\"YuanQuID\":\"0\"}` 这种带反斜杠的格式）时，必须在 HTTP 节点**前面**加一个代码节点用 `json.dumps` 打包，禁止手写反斜杠或靠 LLM 拼串。链路变为：【提取 LLM】→【代码1 打包员】→【HTTP】→【代码2 清洗员】→【播报 LLM】。

---

## 二、HTTP 请求节点

### 2.1 配置位置

| 配置项 | 位置 | 常见错法 |
| :--- | :--- | :--- |
| 请求方法 | 节点顶部下拉 | 默认是 `GET`，POST 接口忘改，直接得到 405 |
| URL | 方法右侧输入框 | 照抄文档占位符 `http://ip:端口/...`，未换真实 IP 与端口 |
| Headers | `HEADERS` 表格区 | 漏 `Content-Type: application/json`；新增行找不到「+」 |
| Body | `BODY` 区，先勾 `JSON` | 勾了 JSON 但正文是表单格式；用纯文本手敲变量 |

新增 Header 行的正确做法：`HEADERS` 区域没有「+」按钮，它是一张自动增长的表格。点击 `Content-Type` 正下方那个提示「键入 '/' 键快速插入变量」的空白框直接打字输入键名，右侧框填值，写完后系统自动追加下一行空白行。

### 2.2 `data` 传字符串化 JSON 的三个致命点

**致命点 1：气泡前后的空格。** 双引号与气泡之间多敲空格，传统系统解析器直接报错。

```json
"data": "  </>代码执行/result  "   // 错误
"data": "</>代码执行/result"       // 正确
```

**致命点 2：双引号套双引号。** 气泡值本身含双引号，塞进外层引号后变成非法 JSON → `Failed to parse JSON`。

```json
"data": "{"keyWord": "眼科", "YuYueGH": "1"}"   // 错误
```

解法：在代码节点里对 `json.dumps` 的结果再做一次转义，HTTP 节点只引用这一个变量。

```python
stringified = json.dumps(inner_data, ensure_ascii=False)
safe_string = stringified.replace('"', '\\"')
return {"result": safe_string}
```

**致命点 3：变量引用写成纯文本。**

手敲 `{{START.hotel_id}}`、`{{CODE.guest_name}}`，Dify 不会替换，会把花括号原样当文本发出。判断依据：运行日志的 `数据处理 / request` 里能看到 `{\n \"hotel_id\": \"{{START.hotel_id}}\"`。

规则：所有变量必须点输入框右上角 `{x}` 插入带底色的气泡，最终正文长相是「双引号里包着蓝色药丸气泡」，不是白色英文字母。

### 2.3 鉴权方式

| 方式 | 配置位置 | 说明 |
| :--- | :--- | :--- |
| API Key | 自定义工具保存时的【鉴权方法】→ `API Key`，位置选 `Header`，Header 名填 `Authorization`，值填对方给的 Token | 适用于 Swagger 工具 |
| Header 自定义 | HTTP 节点 `HEADERS` 手动加行 | 适用于 HTTP 节点；`User-Agent` 伪装也走这里 |
| IP 白名单 | 不在 Dify 侧配置，由对方防火墙加白 | 必须提供平台**出口** IP，不是入口 IP，见 3.2 |
| 无 | 内网自建服务、公开测试接口 | MCP 弹窗里同时要关掉「使用动态客户端注册」 |

`User-Agent` 伪装（WAF 按特征拦截时用）：Key `User-Agent`，Value `Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36`。

### 2.4 超时与重试

- HTTP 节点失败会自动重试，日志里表现为「重试 3 次、耗时 12 秒」后才判失败；看到节点打绿勾写「重试成功」说明请求本身通了，问题在后续节点。
- 对方是传统内网系统时，先在 Apifox 里量出真实响应时间，再判断 Dify 侧超时是否够用。
- 重试对写操作（扣款、下单、实名核验）有副作用风险，这类接口不要依赖自动重试。

### 2.5 变量过滤器抽取（Dify 无原生 JSONPath）

从上游节点输出里取字段，用过滤器链，不要手写字符串切割：

```text
{{提取搜索词.text | parse_json | get:'keyword'}}
```

- `parse_json` 先把 LLM 输出的文本转成对象，`get:'key'` 取键。
- 返回是数组（如 `List<I_PaiBan>`）时，过滤器无法遍历，改用代码节点 `for item in schedule_list` 逐条取。
- 接口返回「字符串化的 JSON」时，`parse_json` 只能剥一层，第二层仍要代码节点再 `json.loads`。

---

## 三、错误码排查树

先做一次交叉验证再动手改配置：把同一个请求在 Apifox 里原样复现（见第八节）。Apifox 通、Dify 不通 → 配置或出口网络问题；两边都不通 → 接口方问题。

### 3.1 401 / 403

| 码 | 真实原因 | 验证方法 |
| :--- | :--- | :--- |
| 401 | Token 缺失、过期、放错位置（应在 Header 却放 query） | 在 Apifox 里带同一 Token 重放；换对方新发的 Token |
| 403（本地 Apifox 通、云端 Dify 报） | 云端出口 IP 被 WAF 拦；或 `User-Agent` 为 `Go-http-client/1.1` 被判定为脚本 | 加 Chrome UA 头重测；用查 IP 节点抓真实出口 IP |
| 403（对方已加白仍报） | 加白的 IP 货不对版：加的是平台入口 IP，实际出口是另一段 | 见 3.2 抓内鬼 |
| 403（附 HTML 页面） | WAF 特征拦截（深信服、绿盟等），不是应用层拒绝 | 换 UA 后 200 即确认 |

抓真实出口 IP：新建临时 HTTP 节点，`GET http://pv.sohu.com/cityjson?ie=utf-8`（返回 `var returnCitySN = {"cip": "120.25.x.x", ...}`），或 `GET http://ip.cip.cc`（返回纯文本 IP）。拿到的 `cip` 才是平台真正敲对方大门用的 IP。测完删掉临时节点。

### 3.2 404 / 405

| 码 | 真实原因 | 验证方法 |
| :--- | :--- | :--- |
| 404 | 公网 Mock 链接过期（`run.mocky.io/v3/xxx` 被平台清理） | 浏览器直接打开该 URL，看是否 404 |
| 404 | 路径大小写不符、少了 `/rest/` 等中间层 | 找对方确认完整路径 |
| 405 | 方法错配：文档写 POST，节点还是默认 GET | 切换 GET/POST 各试一次 |
| 405 | `http` 与 `https` 权限配置不同，公网映射只放行其中一种 | 把 `https` 改成 `http` 重试 |
| 405（IIS 报错页、中文乱码成菱形问号） | 服务端 IIS + WCF（`.svc`）网关拒绝该方法，页面默认 GB2312 编码 | 截图 + 状态码交对方核查网关/WAF 是否拦截外网 POST |

### 3.3 500 / 超时 / Connection refused

| 现象 | 真实原因 | 验证方法 |
| :--- | :--- | :--- |
| `500 Internal Server Error`（MCP 添加时） | 目标 MCP 服务没在跑，或地址不可达 | 看运行服务的终端窗口是否还开着、有无报错退出 |
| `Connection refused` | 目标域名被网络策略拦截（如访问国外 `api.ipify.org`） | 换成国内接口 `http://ip.cip.cc` |
| 超时 | 内网穿透链路慢、对方接口本身慢、Body 过大 | Apifox 量真实耗时；缩小请求体 |
| `Expecting value: line 1 column 1 (char 0)` | Python 去解析 JSON，但拿到的是空串或 HTML 报错页 | 打印原始 `body`；Apifox 看返回体类型 |

---

## 四、数据清洗（代码执行节点）

### 4.1 代码节点三条硬约束

1. 函数参数名必须与左侧【输入变量】名完全一致，否则报 `unexpected keyword argument`。
2. `return` 的键名必须与右侧【输出变量】声明名完全一致，否则报 `Output result is missing`。
3. 不同平台版本传参方式不一致（有时包成 `arg1` 字典，有时直接传变量名），用 `**kwargs` 兜住。

```python
import json

def main(**kwargs):
    raw_input = ""
    if "keyword" in kwargs:
        raw_input = kwargs["keyword"]
    elif "arg1" in kwargs and isinstance(kwargs["arg1"], dict):
        raw_input = kwargs["arg1"].get("keyword", "")
    else:
        for k, v in kwargs.items():
            if isinstance(v, str):
                raw_input = v
                break
```

### 4.2 全量返回 vs 精简返回

| 选择 | 适用 | 代价 |
| :--- | :--- | :--- |
| 全量喂 LLM | 返回字段少、用户问题覆盖面广（既问地点又问费用又问号源） | Token 高、易看错数字 |
| 精简成固定文本 | 用户问题单一（只查号源） | 后续新增问法要改代码 |
| 折中：代码里按字段打标签再拼文本 | 用户问题多样（推荐） | 需维护拼装模板 |

折中写法：代码把每条记录拼成带标签的多行文本，播报 LLM 按用户当前问题挑对应标签作答。

```python
line = f"【医生名片】{name} ({title})\n"
if expertise: line += f"【擅长】{expertise}\n"
line += f"【就诊指引】{campus} {dept} {location} | 诊疗费:{fee}元\n"
line += f"【排班状态】{date} 已停诊" if is_stop == 1 else f"【排班号源】{date} 上午:{parse_tickets(am_tickets)}, 下午:{parse_tickets(pm_tickets)}"
```

### 4.3 字段映射字典（客户系统代码 ↔ 业务名称）

客户系统只认内部数字代码，不认中文名。在代码节点里内置映射表，用最长词优先匹配，避免「眼科(儿童)」被「眼科」截胡。

```python
dept_mapping = {"眼科(儿童)": "2201000", "眼科(成人)": "2200000", "眼科": "2201000",
                "儿童近视防控": "14100", "儿保门诊": "6020000", "内科": "2020000", "儿科": "2040000"}
matched_code = ""
for dept_name, dept_code in sorted(dept_mapping.items(), key=lambda x: len(x[0]), reverse=True):
    if dept_name in actual_word or actual_word in dept_name:
        matched_code = dept_code
        break

if matched_code:
    inner_data["KeShiDM"] = matched_code
else:
    inner_data["KeShiDM"] = "2040000"   # 兜底，避免接口报「代码为空」
    inner_data["keyWord"] = actual_word
```

要点：
- 向客户索要《名称与代码对照表》Excel 再填表，不要凭猜。
- 必须有兜底代码，否则未命中时接口直接报「入参节点科室代码为空」。
- 同一套映射在 Swagger 方案里应前置成英文/数字参数（机场三字码、航司代码、科室代码），由上游 LLM 或代码节点转换后再传给工具。

### 4.4 JSON 解析失败的四个原因

| 原因 | 现象 | 处理 |
| :--- | :--- | :--- |
| LLM 输出了 Markdown 围栏或 `json` 前缀 | `json.loads` 抛异常 | `clean_str = s.strip('` \n')`；`if clean_str.lower().startswith('json'): clean_str = clean_str[4:].strip()` |
| 多余空格 | 首尾空格导致解析失败或接口报格式错 | `str(raw).strip()`；HTTP Body 里气泡两侧不留空格 |
| 类型不符 | `if t < 0` 报 `TypeError`，因为接口给的是字符串 | 先 `t = int(t)`，再判断 |
| 嵌套字符串 | 外层 `json.loads` 后 `data` 仍是字符串 | 对内层再做一次 `json.loads` |

### 4.5 哨兵值与空值兜底

- 号源约定：`< 0` 表示不显示，`0` 表示已约满，`> 0` 表示有号。代码里必须显式翻译，禁止把 `-1` 直接给模型。
- `data` 可能为 `"null"` 或空串：先判 `if not inner_data_str or inner_data_str == "null"` 再解析。
- 外层 `code` 判定用字符串比较 `str(response_dict.get("code")) != "1"`，避免 `1` 与 `"1"` 类型不等。

### 4.6 返回结构对齐前端协议

- 代码节点输出变量名 = HTTP 节点/播报节点引用的气泡名，三处必须同名（打包员 `result`、清洗员 `result`、HTTP 的 `data`、播报提示词里的 `{{代码2.result}}`）。
- 需要结构化下发时，代码直接 `return {"room_number": ..., "status": ...}` 多个键，让下游节点按字段引用，不要拼成一整段文本再让 LLM 二次拆。
- 模板转换节点（Template）强制要求绑定至少一个输入变量才能通过校验；只需固定文案时，随便绑一个存在的变量，正文手写常量即可。

---

## 五、Swagger / OpenAPI 接入

### 5.1 配置流程

1. 打开【工具】→【自定义工具】→ 创建，填名称（建议英文）、鉴权方法（`无` / `API Key`）、`Schema`（YAML）。
2. 粘贴 YAML 后观察下方【可用工具】表格是否自动弹出条目（名称、描述、方法、路径、右侧【测试】按钮）。弹出即解析成功。
3. 点【测试】跑一次真实调用，成功后再【保存】。保存后该工具进入工作区工具库。

### 5.2 YAML 规范要点

```yaml
openapi: 3.0.0
info:
  title: 医院门诊排队叫号查询系统
  description: 提供患者排队序号、前序等待人数、当前就诊号码的实时查询服务。
  version: 1.0.0
servers:
  - url: https://api.hospital.example.com/v1
paths:
  /outpatient/queue-status:
    get:
      summary: 查询患者门诊排队候诊进度
      description: 当患者询问"我还要等多久"、"现在叫到几号了"、"前面还有几个人"时，调用此接口。
      operationId: get_patient_queue_status
      parameters:
        - name: patient_id
          in: query
          description: 患者的就诊卡号或电子医保凭证ID
          required: true
          schema: {type: string, example: "KH20240508001"}
      responses:
        '200': {description: 成功获取排队信息, content: {application/json: {schema: {type: object}}}}
```

| 字段 | 规则 |
| :--- | :--- |
| `operationId` | 工具的身份证，只能纯英文字母 + 下划线（`query_subscriber_balance`），中文会失效 |
| `description` | 大模型的使用说明书。必须写清「当用户询问 X 时必须调用」，命中率由这句决定 |
| 参数 `description` | 决定抽取准不准，写明「11 位手机号码」这类格式约束 |
| `required` | 严格区分，必填参数缺失会直接调用失败 |
| `servers.url` | 只写到域名/版本前缀，路径写在 `paths` 里 |
| POST 体 | 用 `requestBody.content.application/json.schema`，参数写进 `properties` |

连通性自测用 httpbin（无需鉴权、响应快）：

```yaml
openapi: 3.0.0
info: {title: 网络连通与回显测试工具, description: 验证 Swagger 工具链能否正常解析并调用外部接口, version: 1.0.0}
servers:
  - url: https://httpbin.org
paths:
  /get:
    get:
      summary: 发送测试消息并原样回显
      operationId: test_get_echo
      parameters:
        - {name: test_message, in: query, description: 测试问候语, required: true, schema: {type: string, default: "hello_dify_test"}}
      responses:
        '200': {description: 测试成功返回, content: {application/json: {schema: {type: object}}}}
```

返回体含 `"args": {"test_message": "hello_dify_test"}` 即工具链可用。

### 5.3 与 HTTP 请求节点的区别

| 维度 | HTTP 请求节点 | Swagger 自定义工具 |
| :--- | :--- | :--- |
| 节点数量 | 一个接口要 5 个节点：分类 → 提取 LLM → 打包代码 → HTTP → 清洗代码 → 播报 LLM | 1 个 Agent / 支持工具调用的 LLM 节点勾选工具即可 |
| 触发方式 | 硬编码：连线指到就必须执行，用户说「算了不查了」也照发 | 模型自主判断：觉得有必要才调用；同一轮可先后调多个工具再整合回答 |
| 参数来源 | 你手动连线喂入 | 模型自己从用户话里抽取 |
| 复用范围 | 仅当前工作流，换项目要重新连线排查 | 工作区级公共资产，多个智能体勾选即用 |
| 适用 | 支付扣款、实名核验、状态机严格流转、入参畸形需 Python 深度纠偏 | 标准 RESTful 接口、天气/物流/CRM/飞书、需要跨项目复用 |

一句话选型：客户给了接口文档（URL + 入参）→ 优先包成 Swagger 工具；流程不允许模型自主发挥 → 用 HTTP 节点。

### 5.4 中文参数前置转换

- 工具的 `operationId`、参数名、枚举值全部用英文/代码；枚举型参数写进 `description` 并列出可选值，减少模型瞎填。
- 用户口中的中文（「眼科」「国航」「浦东」）必须在调用前转成对方系统认的代码：科室代码 `2201000`、航司代码 `CA`、机场三字码 `PVG`。
- 转换位置二选一：上游 LLM 提示词里直接要求输出代码；或代码节点用映射字典转换（推荐，可控可兜底）。

### 5.5 从工具返回里抽字段

- 工具节点输出是结构化对象，直接用变量引用取字段；需要二次加工时接代码节点。
- 需要从文本型返回里取单字段，用过滤器链 `{{节点.body | parse_json | get:'data'}}`。
- 返回是数组时不要试图用过滤器遍历，接代码节点循环。

---

## 六、MCP 接入

### 6.1 选型判断

| 条件 | 选型 |
| :--- | :--- |
| 客户已有成熟 HTTP API，只给一个 URL，安全隔离要求高，单次查询/提交 | Swagger / OpenAPI 或 HTTP 节点 |
| 需要直读数据库（Text-to-SQL）、本地文件系统、Git 仓库、浏览器/桌面控制 | MCP |
| 需要接入社区现成开源 MCP（Postgres、SQLite、GitHub、Notion、高德地图） | MCP |
| 需要双向长连接、持续监听系统状态 | MCP（JSON-RPC 2.0，stdio 或 SSE） |

本质差异：Swagger 是「一次 Request 一次 Response 就断开」的无状态 HTTP；MCP 是长连接，除 Tools 外还能把对方数据当 Resources 读。

### 6.2 本地 MCP 服务搭建要点

先装库：`pip install "mcp[cli]"`。

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("FiveHealthCalculator")

@mcp.tool()
def calculate_child_bmi(height_cm: float, weight_kg: float) -> str:
    """输入儿童的身高(厘米)和体重(公斤)，自动计算BMI并给出医学健康分级评估"""
    bmi = round(weight_kg / ((height_cm / 100) ** 2), 2)
    return f"身高 {height_cm}cm，体重 {weight_kg}kg，BMI {bmi}"

if __name__ == "__main__":
    mcp.run(transport="sse")
```

| 要点 | 说明 |
| :--- | :--- |
| 传输协议 | Dify 走 HTTP + SSE，必须 `mcp.run(transport="sse")` |
| 端口 | 示例 8000，启动日志出现 `Uvicorn running on http://127.0.0.1:8000` 才算成功 |
| 进程必须常驻 | 运行终端窗口一关，服务立刻死，Dify 侧报 500 |
| 函数 docstring | 就是工具 description，写清用途与参数含义，模型靠它决定调用 |
| 工具数量 | 一个 `@mcp.tool()` 对应 Dify 侧一个工具，添加成功会显示「N 个工具」 |

### 6.3 Docker 环境下的 127.0.0.1 隔离陷阱与 URL 规范

MCP 服务跑在宿主机、Dify 跑在 Docker 容器里时，容器内的 `127.0.0.1` 指向容器自己，找不到宿主机的 8000 端口，表现为添加时报 `500 Internal Server Error`。

| 部署形态 | 服务端点 URL |
| :--- | :--- |
| Dify 为本地 Docker | `http://host.docker.internal:8000/sse` |
| Dify 在公司局域网服务器 | `http://<宿主机内网IP>:8000/sse`（`ipconfig` 查 `192.168.x.x` / `172.16.x.x`） |
| Dify 与 MCP 同机同网络命名空间 | `http://127.0.0.1:8000/sse` |
| 内网部署 | `http://<内网IP>:<端口>/sse` |
| 托管类官方服务（带 Key） | `https://mcp.amap.com/sse?key=<你的Key>` |

规则：
- 末尾必须带 `/sse`；路径被写成 `/******` 之类占位符时必须替换。
- 名称填中文可读名，服务器标识符填英文（如 `five-health-calc`）。
- 认证选 `无`，并把【使用动态客户端注册】开关关闭。该开关用于企业级 OAuth2 动态注册，自建服务没有这套认证，开着会被强制要求填客户端 ID 与密钥，直接卡死无法添加。
- 卡片上出现「N 个工具」+ 绿点才算接入成功；显示「未配置服务」或红字说明没连通。
- 排查捷径：点开工作区里已连通的 MCP 卡片（绿点 + 工具数），照抄它的 URL 写法。

---

## 七、飞书集成

### 7.1 应用创建与凭证

1. 打开 `https://open.feishu.cn/`，进【开发者后台】→【创建企业自建应用】。
2. 进入【凭证与基础信息】，复制 `App ID`（形如 `cli_a1b2c3...`）与 `App Secret`。
3. 回 Dify 授权弹窗填这两项，凭据名称自取。

### 7.2 必须开启的能力与权限

| 项 | 位置 | 不配的后果 |
| :--- | :--- | :--- |
| 权限 `im:message` | 【权限管理】 | 无法收发消息 |
| 权限 `im:message.group_msg` | 【权限管理】 | 无法收发群组消息 |
| 机器人能力 | 【添加应用能力】→【机器人】→ 添加 | `code: 230006, msg: 'Bot ability is not activated.'` |
| 可用人员范围 = 所有员工 | 【版本管理与发布】/【权限管理】内 | 给非开发者发消息报 `code: 230001 ... invalid receive_id` |
| 发布新版本 | 【版本管理与发布】→【创建版本】→【申请发布】 | 任何配置改动都不生效 |

**改完任何配置必须发布新版本**，这是飞书铁律：权限、机器人能力、可用人员范围，改一项就要发布一次（版本号递增，如 `1.0.0`→`1.0.1`→`1.0.2`）。

### 7.3 文本消息内容格式

- `消息类型（Msg Type）` 选 `text` 时，`Content` 必须是 JSON：`{"text": "要发送的内容"}`，不能只填裸文本；换行用 `\n`。
- 接收者类型 `Receive ID Type` 可选 `open_id`、`email`、手机号；填邮箱最省事。
- 接收者 ID 支持动态气泡，接上游 LLM 提取出的邮箱即可实现「访客报谁就发给谁」。

```json
{"text": "【前台来访提醒】\n您好，前台检测到有访客找您！\n- 访客原话：{{sys.query}}\n请及时前往接待~"}
```

### 7.4 节点显示「需要授权」且右侧面板空白

按顺序做三步：

1. 点节点里那个 `工作区默认 ∨` 下拉框，选中你刚创建的凭据名称。选中后右侧立刻刷新出 `Receive ID Type`、`Receive ID`、`Msg Type`、`Content` 等输入框。
2. 仍不刷新：`Delete` 删掉节点 → `Ctrl + F5` 强制刷新页面 → 从【工具】重新拖入并连线（前端缓存了「未授权」死锁状态）。
3. 仍报需要授权：回飞书后台确认机器人能力已添加、权限已开通、版本已发布。

### 7.5 群机器人 Webhook

获取：飞书群 → 右上角 `···` →【群机器人】→【添加机器人】→ 选系统列表里那个朴素的【自定义机器人】（**不是**自己在开放平台建的自建应用机器人，后者走 App ID/Secret，没有 Webhook 地址）→ 命名 → 安全设置（测试阶段不勾选，或勾「自定义关键词」填「提醒」）→ 完成 → 复制 `https://open.feishu.cn/open-apis/bot/v2/hook/xxxxxxxx-xxxx...`。

HTTP 直发写法（不依赖插件）：`POST <webhook>`，`HEADERS: Content-Type: application/json`，Body：

```json
{"msg_type": "text", "content": {"text": "【前台来访提醒】\n前台检测到有新访客到访！\n来访信息：{{LLM.text}}\n请相关老师注意接待！"}}
```

@ 人写法（写在 `text` 里）：@所有人 `<at user_id="all">所有人</at>`；@某人 `<at user_id="ou_xxxxxx">西南</at>`。

Webhook 免审核、免可用人员白名单，群消息首选此路；私聊必须走自建应用。

### 7.6 通讯录匹配：知识库 vs 官方 API

| 方案 | 做法 | 优点 | 缺点 |
| :--- | :--- | :--- | :--- |
| 知识库挂载 | Excel 四列（真实姓名 / 花名外号 / 所属部门 / 飞书企业邮箱）→ 建知识库（高质量检索）→ 链路：提取姓名 LLM → 知识检索 → 提取邮箱 LLM → 条件分支 → 发飞书 | 零开发；支持「柴教授」「老王」这类外号模糊匹配 | 需手工维护；人员变动要更新表 |
| 官方通讯录 API | `POST https://open.feishu.cn/open-apis/search/v1/user`，入参 `query: "王瑞航"`，返回 `user_id` / `open_id` | 实时同步组织架构，新人入职即可搜到 | 需开通 `contact:user.employee_id:readonly`，通常要 IT 管理员审批 |

知识库方案必须加安全闸门：
- 提取邮箱的 LLM 提示词里规定「找不到人只输出 `NONE`」。
- 接条件分支判断 `{{LLM.text}}` 不包含 `NONE` 才发消息，ELSE 走「查无此人」话术，避免飞书报 `invalid receive_id`。

---

## 八、Mock 与自测

### 8.1 用 Apifox / Postman 复现 Dify 请求

一比一还原四项：方法（POST）、URL（含协议/端口/路径/大小写）、Headers（`Content-Type: application/json`，需要伪装再加 `User-Agent`）、Body（选 `json`，粘死值版本）：`{"hospitalID": "1005", "type": "TranYiYuanPBXX", "data": "{\"keyWord\":\"眼科\", \"YuYueGH\":\"1\"}"}`。

| 返回 | 结论 | 动作 |
| :--- | :--- | :--- |
| HTML 页面 / 403 / 404 | 接口方网络层拦截或地址错 | 截图交对方 IT，问是否 IP 白名单、WAF 拦截、是否强制 http |
| `{"code": -1, "msg": "参数错误"}` | 文档与实际不符 | 试去掉反斜杠改成普通 JSON 对象；试换 `hospitalID`；试换 GET |
| 200 且数据正常 | 接口没问题，问题在 Dify 配置 | 拿成功响应体回改清洗代码与变量引用 |

### 8.2 Mock 接口选型

| 工具 | 用法 | 注意 |
| :--- | :--- | :--- |
| `designer.mocky.io` | 填 Status `200`、Content-Type `application/json`、Body 粘 JSON → Generate | 生成 `run.mocky.io/v3/xxx`，公共免费链接会被定期清理，随时变 404 |
| `beeceptor.com` | 填端点名（如 `demo-hotel-pms`）→ Create Endpoint →【Mocking Rules】→ Create New Rule（Method `POST`、Path `/`、Status `200`、Response Body 粘 JSON）→ Save Rule | 无需注册，稳定性优于 mocky；地址形如 `https://demo-hotel-pms.free.beeceptor.com` |

Mock 返回 404 一定是链接失效，不是 Dify 配置问题：先在浏览器直接打开该 URL 验证。

### 8.3 判断是接口方问题还是配置问题

```text
Apifox 通？
├── 否 → 接口方问题（地址/白名单/WAF/方法/协议/参数强校验）
│         └── 带上截图与完整请求信息找对方 IT，问清出口 IP 加白、路径、是否强制 http、必填字段
└── 是 → 配置问题
          ├── 日志 request 里变量没被替换 → 手敲了 {{}}，改用 {x} 气泡
          ├── 日志 request 格式非法 → 反斜杠/空格/双引号套双引号
          ├── 状态码 403 且带 HTML → 加 User-Agent 伪装，再抓出口 IP 核对白名单
          └── 状态码 200 但下游报错 → 清洗代码的输出变量名、字段名、类型不匹配
```

### 8.4 上线前必查清单

- [ ] 节点方法与文档一致；URL 无 `ip`、`端口`、`******` 占位符
- [ ] Headers 含 `Content-Type: application/json`；Body 里变量都是 `{x}` 气泡且外侧无空格
- [ ] 打包代码输出名 = HTTP 引用名；清洗代码输出名 = 下游引用名
- [ ] 清洗代码含 `try/except`、空值与 `null` 兜底、哨兵值翻译
- [ ] 播报 LLM 提示词含「禁止输出 JSON/英文字段名」「禁止输出思考过程」；飞书侧权限 + 机器人能力 + 可用人员范围齐全且已发布新版本；Mock 链接已换真实接口
