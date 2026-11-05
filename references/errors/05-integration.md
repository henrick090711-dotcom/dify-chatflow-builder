# 外部集成

覆盖症状：401/403/404/405、data 传参、飞书授权、MCP、Swagger

条目 70–104（共 35 条）

---

## 代码执行节点

### 70. `TypeError: main() got an unexpected keyword argument 'keyword'`

- **现象**：`TypeError: main() got an unexpected keyword argument 'keyword'`，代码节点直接失败。
- **原因**：函数签名 `def main(keyword):` 的形参名与左侧【输入变量】名不一致，Dify 按输入变量名注入关键字参数。
- **解法**：改形参名与输入变量名完全一致；不确定时直接 `def main(**kwargs):`，从 `kwargs` 里按变量名取。
- **验证**：节点由红转绿，右侧【输出】出现值；日志里能看到 `输入变量` 与实参同名。

### 71. `TypeError: main() got an unexpected keyword argument 'arg1'`

- **现象**：上一个报错改完后，同一节点又报 `unexpected keyword argument 'arg1'`。
- **原因**：平台版本差异——有的版本把全部输入包成 `arg1` 字典传入，有的直接按变量名传入。
- **解法**：用 `**kwargs` 兼容两种形态：`if "keyword" in kwargs` → `elif isinstance(kwargs.get("arg1"), dict)` → `else` 遍历取第一个字符串值。
- **验证**：同一段代码在不同工作区/不同版本下均能跑通。

### 72. `Output result is missing`

- **现象**：`Output result is missing`，节点红色失败，但代码本身没抛异常。
- **原因**：`return` 的键名与右侧【输出变量】声明的名字不一致（声明 `result`，返回 `data` 或 `{"text": ...}`）。
- **解法**：把 `return` 的键改成与输出变量声明逐字相同；下游气泡引用名也要跟着一致。
- **验证**：节点变绿，下游节点气泡能选中该变量且运行时非空。

### 73. `Expecting value: line 1 column 1 (char 0)`

- **现象**：`json.decoder.JSONDecodeError: Expecting value: line 1 column 1 (char 0)`。
- **原因**：`json.loads` 拿到的是空字符串、HTML 报错页或纯文本，不是 JSON（HTTP 失败、返回 `null`、WAF 拦截页都会这样）。
- **解法**：先判空 `if not raw: return {"result": "接口无返回"}`；`print(raw)` 打印原始 body 定位来源；对 HTML 用 `if raw.strip().startswith("<"): return {"result": "接口返回异常页面"}`。
- **验证**：日志能看到被打印的原始 body，兜底话术正常返回给用户而不是整链报错。

### 74. `TypeError: '<' not supported between instances of 'str' and 'int'`

- **现象**：号源判断处抛类型错误，或「明明有排班却回复未排班」。
- **原因**：接口把数字给成字符串（`"-1"`、`"0"`），代码里直接与整数比较。
- **解法**：先 `t = int(t)`（外层包 `try/except`），再按哨兵值翻译：`< 0` 不显示、`== 0` 已约满、`> 0` 有号；禁止把 `-1` 原样交给模型。
- **验证**：对 `-1`/`0`/`5` 三种返回各测一次，播报话术分别为不显示、已约满、有号。

### 75. `json.loads` 报错但返回体肉眼看是 JSON

- **现象**：解析失败，返回体带 ``` 围栏或开头多一个 `json`。
- **原因**：LLM 输出了 Markdown 代码块或 `json` 前缀。
- **解法**：`clean_str = s.strip('` \n')`；`if clean_str.lower().startswith('json'): clean_str = clean_str[4:].strip()`；再 `json.loads`。
- **验证**：把带围栏与不带围栏两种 LLM 输出都喂进代码节点，均解析成功。

### 76. 嵌套 `data` 解析后仍是字符串

- **现象**：外层 `json.loads` 成功，取 `data` 后 `for item in data` 报 `string indices must be integers`。
- **原因**：客户接口把 `data` 作为「字符串化的 JSON」返回，需要二次解析。
- **解法**：对内层再 `json.loads`：`inner = json.loads(response_dict.get("data") or "null")`，解析前判空串与 `"null"`。
- **验证**：`type(inner)` 为 `dict`，`inner.get("List")` 可取到数组。

## HTTP 请求节点

### 77. `Failed to parse JSON`（双引号套双引号）

- **现象**：HTTP 节点报 `Failed to parse JSON`，或对方返回格式错误。
- **原因**：Body 写成 `"data": "{"keyWord": "眼科"}"`，气泡值自带双引号，与外层引号冲突成非法 JSON。
- **解法**：在打包代码节点里 `stringified = json.dumps(inner_data, ensure_ascii=False)` 后 `safe = stringified.replace('"', '\\"')`，HTTP 节点只引用 `{{代码1.result}}` 一个变量，禁止手写反斜杠。
- **验证**：日志 request 里 `data` 值形如 `{\"keyWord\":\"眼科\"}`，对方返回 `code` 为 1。

### 78. `data` 首尾多空格导致接口报格式错误

- **现象**：Apifox 同样参数通、Dify 不通，对方返回参数格式错误。
- **原因**：Body 里写成 `"data": "  </>代码执行/result  "`，双引号与气泡之间多敲了空格，传统系统解析器不裁剪。
- **解法**：把气泡紧贴双引号：`"data": "</>代码执行/result"`；代码侧同时 `str(raw).strip()`。
- **验证**：日志 request 的 `data` 值首尾无空格。

### 79. 变量没被替换，日志里出现 `{{START.hotel_id}}`

- **现象**：`数据处理 / request` 日志显示 `{\n \"hotel_id\": \"{{START.hotel_id}}\"`，接口报缺参。
- **原因**：手工敲入 `{{变量}}`，Dify 当作纯文本原样发出，不做模板替换。
- **解法**：点输入框右上角 `{x}` 插入带底色的变量气泡；最终正文应为「双引号内包蓝色气泡」，不是白色英文字母。
- **验证**：日志 request 中该字段是真实值而非花括号。

### 80. `405 Method Not Allowed`（IIS 报错页、中文乱码成菱形问号）

- **现象**：返回 405，页面是 IIS 错误页，中文全是 `????`。
- **原因**：方法错配——节点默认 `GET`，接口要 `POST`；或服务端 IIS + WCF（`.svc`）网关拒绝该方法；页面默认 GB2312 编码故乱码。
- **解法**：先把请求方法下拉切到 `POST`；仍 405 再把 `https` 改 `http` 试；都失败则截图（含状态码）交对方 IT 核查网关/WAF 是否拦截外网 POST。
- **验证**：Apifox 用同方法同协议能通即配置已对；若 Apifox 也 405 则是接口方限制。

### 81. `http` 与 `https` 只有一种放行

- **现象**：Apifox 用 `https` 通，Dify 报 405 或连接异常（或反之）。
- **原因**：公网映射只放行了其中一种协议。
- **解法**：在 URL 上互换协议各试一次，确定放行的那一种后固定。
- **验证**：两种协议在 Apifox 各跑一次，记录可用组合。

### 82. `403 Forbidden`（本地 Apifox 通、云端 Dify 报）

- **现象**：`403 Forbidden`，有时附 HTML 页面。
- **原因**：云端出口 IP 不在白名单；或请求 `User-Agent` 为 `Go-http-client/1.1` 被 WAF 判定为脚本；深信服/绿盟类 WAF 特征拦截。
- **解法**：`HEADERS` 加 `User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36` 重试；仍 403 再抓出口 IP 找对方加白。
- **验证**：加 UA 后返回 200 即确认是特征拦截；加白后返回 200 即确认是 IP 拦截。

### 83. 对方说已加白名单，仍 403

- **现象**：对方确认已加白，请求依旧 403。
- **原因**：加的是平台入口 IP，实际出口是另一段地址（IP 货不对版）。
- **解法**：新建临时 HTTP 节点 `GET http://pv.sohu.com/cityjson?ie=utf-8`（返回 `var returnCitySN = {"cip": "120.25.x.x", ...}`）或 `GET http://ip.cip.cc`（返回纯文本 IP），取 `cip` 交对方加白；测完删除临时节点。
- **验证**：用该 IP 加白后，原接口返回 200。

### 84. `Connection refused`

- **现象**：`Connection refused` 或请求直接失败。
- **原因**：目标域名被网络策略拦截（如访问国外 `api.ipify.org`）。
- **解法**：换国内查 IP 接口 `http://ip.cip.cc`；内网服务确认端口与协议。
- **验证**：换成国内接口后正常返回纯文本 IP。

### 85. 接口 200 但 `data` 内容为空

- **现象**：Apifox 用英文/拼音能取到数据，传中文关键词（如「眼科」）返回空列表。
- **原因**：客户系统只认内部代码，不认中文名。
- **解法**：向客户索要《名称与代码对照表》Excel，在代码节点内置映射字典，按最长词优先匹配（`sorted(mapping.items(), key=lambda x: len(x[0]), reverse=True)`），并保留兜底代码。
- **验证**：中文问法与代码问法返回同一批数据。

### 86. `入参节点科室代码[KeShiDM]为空`

- **现象**：文档标注「非必须」的字段，实际不传时接口返回该字段为空错误。
- **原因**：文档与实现不一致，该字段是强校验。
- **解法**：映射未命中时也填兜底代码（如 `"2040000"`），同时把中文原文塞进 `keyWord`。
- **验证**：未命中科室的问法返回兜底科室数据，不再报代码为空。

## Swagger / OpenAPI

### 87. YAML 粘进去后【可用工具】表格不弹条目

- **现象**：保存后工具列表为空，或节点里选不到工具。
- **原因**：`operationId` 用了中文、`paths` 与 `servers.url` 拼接重复、YAML 缩进错误、缺 `responses`。
- **解法**：`operationId` 改纯英文字母+下划线；`servers.url` 只写到域名/版本前缀，路径写 `paths`；缩进用两个空格；补 `'200'` responses。
- **验证**：表格出现条目（名称、描述、方法、路径、测试按钮），点【测试】能返回真实数据。

### 88. 工具调用命中率低、模型不主动用

- **现象**：用户明确要求查询，模型却不调工具，或参数抽取错。
- **原因**：工具 `description` 只写了功能，没写触发条件；参数 `description` 缺格式约束。
- **解法**：工具描述写「当用户询问 X、Y、Z 时必须调用此工具」；参数描述写明「11 位手机号码」「机场三字码」这类约束；必填项标 `required: true`。
- **验证**：用三种不同口语问法各测一次，均触发调用且参数正确。

### 89. 工具传中文参数调用失败

- **现象**：传「眼科」「国航」「浦东」报参数非法或返回空。
- **原因**：对方系统只接受代码（科室代码、航司代码、机场三字码）。
- **解法**：调用前置转换——在上游 LLM 提示词里要求直接输出代码，或在代码节点用映射字典转换后传给工具；枚举值写进 `description`。
- **验证**：工具入参日志里出现 `PVG`、`CA`、`2201000` 这类代码值。

## MCP

### 90. 添加 MCP 服务报 `500 Internal Server Error`

- **现象**：填完 URL 点添加，返回 `500 Internal Server Error`。
- **原因**：目标 MCP 服务没在运行，或地址不可达。
- **解法**：确认运行服务的终端窗口仍开着、无报错退出；用浏览器直接访问 `<URL>`（带 `/sse`）看是否响应；重启服务后再添加。
- **验证**：MCP 卡片出现「N 个工具」与绿点。

### 91. MCP 服务明明在跑，Docker 里的 Dify 连不上

- **现象**：宿主机 `curl 127.0.0.1:8000/sse` 正常，Dify 添加仍 500。
- **原因**：Dify 跑在容器里，容器内的 `127.0.0.1` 指向容器自身，找不到宿主机端口。
- **解法**：URL 改为 `http://host.docker.internal:8000/sse`；Dify 在局域网服务器则用 `http://<宿主机内网IP>:8000/sse`（`ipconfig` 查 `192.168.x.x` / `172.16.x.x`）。
- **验证**：添加成功，卡片显示工具数。

### 92. MCP 添加弹窗强制要求填客户端 ID 与密钥

- **现象**：弹窗出现「客户端 ID」「客户端密钥」必填项，不填无法保存。
- **原因**：【使用动态客户端注册】开关处于开启状态，这是企业级 OAuth2 场景才需要的。
- **解法**：关闭该开关；认证方式选 `无`；服务器标识符填英文（如 `five-health-calc`）。
- **验证**：弹窗只剩 URL 等基础字段，保存后卡片绿点。

### 93. MCP URL 填了却一直「未配置服务」

- **现象**：卡片红字或显示「未配置服务」。
- **原因**：URL 末尾缺 `/sse`；路径仍是 `/******` 之类占位符；端口与启动端口不一致。
- **解法**：补全 `http://host:8000/sse`；核对 `mcp.run(transport="sse")` 的端口；托管服务形如 `https://mcp.amap.com/sse?key=<你的Key>`。
- **验证**：卡片显示工具数量与绿点；点开可看到每个 `@mcp.tool()` 对应的工具。

## 飞书

### 94. `code: 230006, msg: 'Bot ability is not activated.'`

- **现象**：发消息节点报 `230006 Bot ability is not activated`。
- **原因**：自建应用没有添加机器人能力。
- **解法**：飞书开放平台 →【添加应用能力】→【机器人】→ 添加；然后【版本管理与发布】→ 创建版本 → 申请发布。
- **验证**：再次发送返回 `code: 0`，目标用户收到消息。

### 95. `code: 230001 ... invalid receive_id`

- **现象**：`230001 invalid receive_id`，或给非开发者发消息失败。
- **原因**：可用人员范围未放开（默认只允许创建者/开发者）；或接收者 ID 类型与值不匹配（用邮箱却选了 `open_id`）。
- **解法**：把可用人员范围设为「所有员工」并发布新版本；`Receive ID Type` 与值类型严格对齐（邮箱配 `email`）。
- **验证**：给非开发者账号发一条测试消息成功。

### 96. 飞书节点显示「需要授权」且右侧面板空白

- **现象**：节点上有 `• 需要授权`，右侧只有空白/无法配置 `Receive ID`。
- **原因**：节点未绑定凭据；或前端缓存了未授权的死锁状态。
- **解法**：点节点里 `工作区默认 ∨` 下拉，选中刚创建的凭据；仍不刷新则 `Delete` 节点 → `Ctrl + F5` 强刷 → 重新从【工具】拖入并连线；仍报错回飞书后台确认机器人能力与权限已发布。
- **验证**：右侧出现 `Receive ID Type`、`Receive ID`、`Msg Type`、`Content` 输入框。

### 97. 改完权限仍不生效

- **现象**：权限、机器人能力、人员范围都改了，消息还是发不出去。
- **原因**：飞书配置改动只在发布新版本后生效。
- **解法**：【版本管理与发布】→【创建版本】（版本号递增，如 `1.0.0`→`1.0.1`）→【申请发布】。
- **验证**：发布状态为「已发布」后重测，报错消失。

### 98. 群机器人拿不到 Webhook 地址

- **现象**：在开放平台建的应用里找不到 webhook URL。
- **原因**：自建应用机器人走 App ID/Secret，本身没有 Webhook；Webhook 属于群自定义机器人。
- **解法**：飞书群 → 右上角 `···` →【群机器人】→【添加机器人】→ 选系统列表里的【自定义机器人】→ 命名 → 安全设置（测试期不勾，或勾「自定义关键词」填「提醒」）→ 完成 → 复制 `https://open.feishu.cn/open-apis/bot/v2/hook/xxxx`。
- **验证**：`POST` 该 webhook 返回 `{"StatusCode":0,"StatusMessage":"success"}`，群里出现消息。

### 99. 群消息发送成功但内容格式不对

- **现象**：群里出现原始 JSON 文本，或消息被拒。
- **原因**：用了应用消息的 `content` 结构去发 webhook，或 `msg_type` 与实际结构不匹配。
- **解法**：webhook 直发用 `{"msg_type": "text", "content": {"text": "..."}}`；@ 人写在 `text` 内：`<at user_id="all">所有人</at>`、`<at user_id="ou_xxx">姓名</at>`；换行用 `\n`。
- **验证**：群消息渲染为正常文本，@ 生效。

### 100. 通讯录查不到人导致发送失败

- **现象**：报 `invalid receive_id`，或消息发给了错误的人。
- **原因**：姓名匹配到多个/零个邮箱，直接把空值或错误值传给发送节点。
- **解法**：知识库方案（Excel 四列：真实姓名/花名外号/所属部门/飞书企业邮箱）在提取邮箱的 LLM 提示词里规定「找不到人只输出 `NONE`」，后接条件分支判断不含 `NONE` 才发；或改用官方通讯录 API `POST https://open.feishu.cn/open-apis/search/v1/user`（需 `contact:user.employee_id:readonly`）。
- **验证**：分别用真实姓名、外号、不存在的人名测试，前两者发送成功，后者走「查无此人」话术。

## Mock 与联调

### 101. Mock 地址突然 404

- **现象**：昨天通的 Mock 接口今天返回 404。
- **原因**：`run.mocky.io/v3/xxx` 公共免费链接被平台定期清理。
- **解法**：浏览器直接打开该 URL 确认；换 `beeceptor.com`（填端点名 → Create Endpoint → Mocking Rules → Create New Rule：Method `POST`、Path `/`、Status `200`、粘 Response Body → Save Rule，地址形如 `https://demo-hotel-pms.free.beeceptor.com`）。
- **验证**：浏览器打开新地址返回预期 JSON；Dify 节点由红转绿。

### 102. 无法判断是接口方问题还是自己配错

- **现象**：报错笼统，对方说「我们接口没问题」。
- **原因**：缺少独立于 Dify 的复现证据。
- **解法**：在 Apifox/Postman 一比一还原方法、URL（协议/端口/路径/大小写）、Headers（`Content-Type: application/json`）、Body（粘死值版本）跑一次，按结果分叉。
- **验证**：Apifox 通 + Dify 不通 → 查变量气泡、Body 格式、出口 IP；两边都不通 → 截图与完整请求交对方 IT，问出口 IP 加白、路径、是否强制 http、必填字段。

### 103. 模板转换节点无法通过校验

- **现象**：模板节点报未绑定变量，无法保存。
- **原因**：节点强制要求至少一个输入变量。
- **解法**：随便绑定一个已存在的变量，正文手写固定常量。
- **验证**：节点可保存并运行，输出为手写文案。

### 104. 播报 LLM 把思考过程输出给用户

- **现象**：用户看到「好的，让我分析一下数据……」「首先我需要……」之类的内心戏。
- **原因**：提示词未禁止输出推理过程。
- **解法**：在 SYSTEM 里明确「只输出面向用户的最终话术，禁止输出思考过程、JSON、英文字段名」；仍出现则换模型。
- **验证**：连续三轮不同问法测试，输出均为直接可读话术。
