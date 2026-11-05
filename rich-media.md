# 富媒体组件返回

适用：要在数字人大屏/前端返回图片、链接、地图、表单、文档时，先抄第二节的格式串，再按第三到七节落节点与代码。

---

数字人大屏、智能前台、AGV 前台、智能座舱场景，一次回答要同时给出数字人能念的台词和前端能渲染的组件。本文给出可交付的信令写法与拼装代码。

## 一、核心机制

### 前提：Dify 不做渲染

Dify 只吐文本。图片框、地图卡片、表单弹窗全部由大屏前端（Vue / React / 安卓 / Webview）解析文本后自己渲染。

| 角色 | 职责 |
|---|---|
| Dify | 把结构化数据拼成约定格式的纯文本信令，经 SSE 原样下发 |
| 前端 TTS 层 | 正则剥掉信令，只把信令外的文字送进数字人语音合成 |
| 前端 UI 层 | 从信令里取字段，渲染原生组件 |

推论：**预览区里看到的永远是代码，不会是组件**，因为 Dify 原生聊天框不认识这套协议。判断标准是「研发的正则能不能匹配到」，不是「Dify 有没有渲染出图」。

### 双通道输出

```text
[数字人口播台词，纯文本，无 Markdown，无括号动作描写]
widget_xxx_start:{...}:widget_xxx_end
```

研发文档原文要求：信令「此部分最好单独一行」。

### 三条硬约束

1. **信令单独成行**：台词在上，信令在下，中间一个换行，不挤同一行。
2. **标记逐字符一致**：`widget_pic_start:` / `:widget_pic_end` 的冒号、下划线、大小写全部照抄。
3. **信令内禁止 Markdown**：不加粗、不换行、不加注释、不包代码块反引号。

### 必须改用对话流的情况

- 要输出富媒体信令 → 用**对话流（Chatflow）**，由【直接回复】节点直出。
- 不走 Agent 模式：Agent 会把信令当普通文本再加工一轮，模型自作主张加粗、换行、补解释、改标记名，前端正则匹配不到，最终显示成代码或不弹窗。
- 不在信令下游再挂 LLM 做「润色」，拼好的信令一经模型，格式即失控。
- 富媒体链路只做三件事：提取（LLM）→ 拼装（代码执行）→ 直出（直接回复）。

## 二、各组件标准写法

### 通用规则

- 起始标记 `widget_<组件>_start:{`，结束标记 `}:widget_<组件>_end`。
- 参数名双下划线开头：`__image_url`、`__link_title`、`__start_text`、`__title`、`__label`、`__value`。
- 参数名与值之间用**英文半角冒号**，不用中文冒号。
- 分隔符按组件区分：链接用**空格**，地图和表单用**英文逗号**。这是最高频出错点。

### 图片

```text
为你找到以下内容：
widget_pic_start:{__image_url:https://maas-watermark-...png?...}:widget_pic_end
```

- 起始 `widget_pic_start:{`，结束 `}:widget_pic_end`；参数 `__image_url` 为图片公网 URL，以 `https://` 开头，单参数无分隔符。
- 变量插法：`widget_pic_start:{__image_url:{{代码执行.image_url}}}:widget_pic_end`，`{__image_url:` 紧贴变量气泡，气泡后紧贴 `}:widget_pic_end`。

### 链接

研发标准模板：

```text
为你找到以下内容：
widget_link_start:{__link_title:{{value}} __link_url:{{value}}}:widget_link_end
```

- 起始 `widget_link_start:{`，结束 `}:widget_link_end`；参数 `__link_title`（标题）、`__link_url`（网址），顺序不可颠倒。
- 分隔符：两参数之间**有且仅有一个英文半角空格**。

固定链接可写死，不必上代码节点：`widget_link_start:{__link_title:魔珐科技官方网站 __link_url:https://xmov.ai}:widget_link_end`。

### 地图

研发最完整标准示例：

```text
为你找到以下内容：
widget_map_start:{__start_text:{{起点}},__end_text:{{终点}},__type:{{交通方式}}}:widget_map_end
```

| 参数 | 含义 | 取值 |
|---|---|---|
| `__start_text` | 起点地名 | 中文，建议带城市前缀 |
| `__end_text` | 终点地名 | 中文，建议带城市前缀 |
| `__type` | 出行方式 | `driving` / `walking` / `riding` / `transfer` |
| 分隔符 | 参数之间用英文半角逗号 `,` | |

研发示例原文（`driving` 后带一个空格）：

```text
widget_map_start:{__start_text:上海桂林路,__end_text:上海外滩,__type:driving }:widget_map_end
```

自己录入和代码拼接时不保留这个空格，写成 `__type:driving}:widget_map_end`。老前端若仍用 `:::AMAP_ROUTE_CARD` JSON 卡片协议，先与研发确认当前实现，两套不混用。

### 结构化表单

研发模板：`widget_form_start:{__title:...,__label:...,__value:...}:widget_form_end`，扁平 key-value 串接，前端按 `__label` / `__value` 成对循环渲染。

```text
widget_form_start:{__title:魔珐科技访客实名登记核对单,__label:访客姓名,__value:张三,__label:手机号码,__value:13812345678,__label:身份证号,__value:310101199001011234}:widget_form_end
```

- 起始 `widget_form_start:{`，结束 `}:widget_form_end`；`__title` 必须第一位，`__label` / `__value` 成对出现，缺失填 `未提供`。
- 分隔符为英文半角逗号，全部键值对平铺，无嵌套。

### 文档 / 文件

Markdown 下载链接（全端通用，零对接）：

```markdown
[点击下载：《嘉兴市妇幼保健院就医指南与科室分布图.pdf》（大小：2.4MB）](https://your-oss-domain.com/files/guide.pdf)
```

文档卡片信令（大屏专用，带扫码带走）：

```text
好的，已为您调出《魔珐科技产品手册》，您可以在屏幕上预览，或微信扫码保存至手机。

:::DOCUMENT_CARD
{
  "file_name": "魔珐全栈具身智能产品画册.pdf",
  "file_type": "pdf",
  "file_size": "8.5MB",
  "file_url": "https://oss.xmov.ai/docs/brochure.pdf",
  "qr_code_url": "https://api.qrserver.com/v1/create-qr-code/?data=https://oss.xmov.ai/docs/brochure.pdf"
}
:::
```

起始标记 `:::DOCUMENT_CARD` 独占一行，结束标记 `:::`。字段含义：`file_name` 展示名，`file_type` 扩展名（前端据此选预览器），`file_size` 展示大小，`file_url` 直链，`qr_code_url` 二维码图片地址（内容为 `file_url`）。动态生成文件（现场造 Excel / 报告）用【代码执行】拼 CSV 或 HTML 模板；需要 PDF 时用 Marketplace 的 `Markdown to PDF`、`CSV / Excel Export` 插件。

## 三、代码执行节点的拼装职责

### 职责边界

只做三件事：结构归一（字符串 → 字典，列表 → 首元素）、逐字段取值兜底（缺失填 `未提供`，不让 `None` 流进信令）、拼成单行信令作为唯一输出变量。不在这里下载文件、请求外部接口、做二次模型调用。可选参数（如 `__type`、`<transition>`）缺失时整段省略，连参数名带值一起去掉，不留 `__type:` 这类空值，否则前端取到空串。

### 结构归一

```python
if isinstance(raw_mcp, str):
    raw_mcp = json.loads(raw_mcp)
if isinstance(raw_mcp, list) and len(raw_mcp) > 0:
    data = raw_mcp[0]
elif isinstance(raw_mcp, dict):
    data = raw_mcp
else:
    data = {}

paths_list = data.get("paths", [])          # 高德 MCP 的 paths 是列表
path_info = paths_list[0] if isinstance(paths_list, list) and paths_list else (
    paths_list if isinstance(paths_list, dict) else {})
distance_meters = int(path_info.get("distance", 15000))
duration_seconds = int(path_info.get("duration", 1500))
```

列表直接 `.get()` 报 `'list' object has no attribute 'get'`。模型输出 `[ALL_COMPLETE]\n{"visitor_name": "张三", ...}` 时，整段 `json.loads` 抛 `Expecting value: line 1 column 1`，先 `re.search(r'(\{[\s\S]*\})', text)` 切片再 `json.loads`，写法见第五节。

### 字段缺失兜底

```python
name = data.get("visitor_name") or data.get("name") or "未提供"
phone = data.get("visitor_phone") or data.get("phone") or "未提供"
company = data.get("company") or data.get("所属企业") or data.get("企业名称") or "未提供"
target = data.get("target_person") or "相关老师"
```

模型输出中文客套话而非 JSON 时，再补一层正则兜底：

```python
if name == "未提供":
    m = re.search(r'姓名[^\u4e00-\u9fa5]*[:：\s*]*([\u4e00-\u9fa5]{2,4})', text) or re.search(r'([一-龥]{2,4})(?:先生|女士)', text)
    if m: name = m.group(1)
if phone == "未提供":
    m = re.search(r'(1[3-9]\d{9})', text)
    if m: phone = m.group(1)
```

### URL 校验

一个多余字母就能让弹窗全白，曾出现 `"page_url": "hhttps://www.shairport.com/..."`。拼 URL 前统一过前缀校验：

```python
page_url = "https://www.shairport.com"
if not page_url.startswith(("http://", "https://")):
    page_url = "https://" + page_url.lstrip("/")
```

### 表单拼装函数

```python
def build_widget_form(payload):
    parts = [f"__title:{payload['title']}"]
    for field in payload["fields"]:
        parts.append(f"__label:{field['label']}")
        parts.append(f"__value:{field['value']}")
    return "widget_form_start:{" + ",".join(parts) + "}:widget_form_end"

final_output = f"{speech_text}\n{form_json_str}"
return {"result": final_output}
```

### 输出变量登记

- 只输出一个 `result`（String），内容是「台词 + 换行 + 信令」。研发代码 `return {"result": final_output}`，界面变量名必须与之一致，否则【直接回复】选不到；下游【直接回复】也不手写文字，直接插 `{{代码执行.result}}`，避免模板引擎拆断信令。
- 需要【直接回复】分别插两个变量时（如 `page_title` / `page_url`），在代码节点里登记两项独立变量，并在界面点 `+` 手动登记，不能只写在 `return` 里。

## 四、地图组件专项

### 三条实现路线

| 路线 | 实现方式 | 前端工作量 | 适用 |
|---|---|---|---|
| A 信令协议 | 输出 `widget_map_start:{...}:widget_map_end`，前端用高德 JS SDK 渲染 | 需渲染组件 | 正式交付，UI 可控 |
| B Web URI 嵌入 | 拼 `https://uri.amap.com/navigation?...`，前端塞 Webview / iframe | 只切一个窗口 | 快速上线，可交互 |
| C 静态地图图片 | 拼静态地图 URL，输出 Markdown 图片 | 零改动 | 现场 Demo、MVP |

路线 B 官方 Web URI：

```text
https://uri.amap.com/navigation?from=起点经纬度,起点名&to=终点经纬度,终点名&mode=car&callnative=0
```

生产版在末尾补 `&policy=1&src=mypage&coordinate=gaode`。中文地名必须 `urllib.parse.quote()` 编码，否则乱码。`mode` 取 `car` 自驾 / `ride` 骑行 / `walk` 步行 / `bus` 公交；`callnative=0` **必须为 0**，强制网页内打开，否则会尝试唤起手机 App。

路线 C 静态地图：

```text
https://restapi.amap.com/v3/staticmap?zoom=12&size=750*400&markers=large,,A:121.429,31.153|large,,B:121.505,31.238&key=高德KEY
```

`markers` 格式为 `大小,颜色,文字:经度,纬度`，多点用 `|` 分隔。高德 Key 必须是**服务平台 = Web 服务**类型，不是 Web 端 JS API 的 Key。

### 起终点用名称还是经纬度

前端 `IWidgetMap` 里 `start_lng_lat?: string`、`end_lng_lat?: string` 带 `?`，是选填项。含义：不传经纬度时，前端高德 SDK 用设备默认定位城市，在同城结果里挑最热门的匹配项。

| 方案 | 优点 | 风险 |
|---|---|---|
| 只传名称 | 对话流极简，3 个节点，无需 MCP | 重名会跑偏 |
| 传经纬度 | 定位精确 | 需地理编码工具，链路变长；模型自估经纬度会幻觉 |

重名是真实风险：「人民广场」全国几乎每个城市都有，上海还有「奉贤人民广场」；「万达广场」仅上海就有 8 座（五角场、江桥、青浦……）。

结论：**默认走名称，用城市前缀消歧，不用经纬度。** 在提取节点的 SYSTEM 提示词里下死命令：

```text
1. start_text: 起点。必须自动补齐所在城市前缀（如默认补齐"上海市"，防止全国重名歧义）。
2. end_text: 终点。必须自动补齐所在城市前缀（如默认补齐"上海市"）。
3. travel_type: 出行方式，严格映射为 driving / walking / riding / transfer 之一。

【范例】
输入："我想骑车去外滩，现在在桂林路"
输出：{"start_text": "上海市桂林路", "end_text": "上海市外滩", "travel_type": "riding"}
```

只有前端已具备经纬度渲染能力、业务要求精确到门牌时，才引入高德 MCP 拿官方纠偏坐标。让模型自己估经纬度会定位到太平洋。

### 出行方式参数

| 枚举 | 用户口语 |
|---|---|
| `driving` | 自驾 / 开车 / 打车 / 出租车 / **未明说时的默认值** |
| `walking` | 步行 / 走路 / 散步过去 |
| `riding` | 骑行 / 骑车 / 扫共享单车 |
| `transfer` | 公共交通 / 坐地铁 / 乘公交车 |

代码里对 `__type` 再做一次白名单校验，非法值回落 `driving`，避免前端拿到中文「骑车」直接报错。

## 五、表单组件专项

### 字段定义

前端接口数据驱动，字段没有写死：

```typescript
interface IWidgetForm {
    title: string
    fields: Array<{
        label: string
        value: string
    }>
}
```

传什么 `__label`，屏幕左边显示什么标签；传什么 `__value`，输入框就填什么内容；字段数量由 `fields` 长度决定，可自由增删。

```python
form_payload = {
    "title": "魔珐科技访客实名登记核对单",
    "fields": [
        {"label": "访客姓名", "key": "visitor_name", "value": name},
        {"label": "手机号码", "key": "mobile_phone", "value": masked_phone},
        {"label": "所属企业", "key": "company_name", "value": company},
        {"label": "被访人员", "key": "target_person", "value": target}
    ]
}
masked_phone = phone[:3] + "****" + phone[-4:] if len(phone) == 11 else phone
```

`key` 是业务系统入库标识，`label` 是屏幕显示文字，`value` 是预填值。信令只带 `__title` / `__label` / `__value`，`key` 不上行，需要时由前端按顺序或自有映射表还原。敏感字段先脱敏再进信令。

### 前端提交回传的格式

用户点「确认提交」后，前端把表单内容以一段中文文本回传：

```text
【我已确认以下信息】访客姓名:未提供，手机号码:未提供，所属企业:未提供，被访人员:相关老师
```

特征：固定前缀 `【我已确认以下信息】`，字段间中文逗号 `，`，字段名与值之间英文冒号 `:`，字段顺序与 `fields` 一致。这段文本作为下一轮用户输入进入对话流，**下游所有依赖表单数据的节点都必须能解析它**，包括发飞书消息、写库、条件判断分支。

### 回传解析的两种方式

**方式一：整段 `json.loads`（不可用）**。模型或前端给的是「前缀标记 + JSON」或中文句式时，整段解析必然抛 `Expecting value: line 1 column 1`，代码落进 `except` 静默分支，字段全部回落「未提供」，表面看是提取不到数据，实际是解析整段就崩了。

**方式二：括号切片后再解析（推荐）**。

```python
json_match = re.search(r'(\{[\s\S]*\})', text)
if json_match:
    data = json.loads(json_match.group(1))
```

`[\s\S]` 覆盖换行，能吃掉跨行 JSON；贪婪匹配到最后一个 `}`，能容忍尾部多余字符。

| 场景 | 整段 `json.loads` | 括号切片 |
|---|---|---|
| 模型加了 `[ALL_COMPLETE]` 前缀 | 崩 | 正常 |
| 模型在 JSON 前后加客套话 / 包了代码块 | 崩 | 正常 |
| 前端回传中文句式 | 崩 | 走字段级正则兜底 |

中文句式回传用字段级正则，如 `re.search(r'(1[3-9]\d{9})', text)` 取手机号。`try/except` 只包解析动作，不包业务拼装，否则出错时拿不到任何线索。

### 字段名变更时需要同步改的地方

以「身份证号 → 所属企业」为例，固定四处，缺一处出错：

| 序号 | 位置 | 改什么 |
|---|---|---|
| 1 | LLM 节点 SYSTEM 提示词 | 输出 JSON 的键名与说明，如 `id_card` 换成 `company` |
| 2 | 代码节点取值行 | `data.get("company") or data.get("所属企业") or "未提供"` |
| 3 | `form_payload.fields` | `{"label": "所属企业", "key": "company_name", "value": company}` |
| 4 | 正则兜底规则 | 新字段补对应正则，旧字段正则删掉 |

不用改：【直接回复】节点（只插 `{{代码执行.result}}`）；`build_widget_form` 函数（按 `fields` 循环，天然支持任意字段数）。需与前端确认：前端若按 `key` 入库或校验，`key` 变更必须同步告知，`label` 变更一般不用。

## 六、图片专项

### 只返回 URL，不下载字节流

错误做法：Python 里 `urllib.request` 把图片下载成 bytes，再拼成 `{'type': 'image', 'transfer_method': 'local_file'...}` 塞进 Markdown 括号。Dify 代码节点跑在精简 Docker 沙箱，缺根证书，下载 `ufileos.com` 图片直接崩：`urllib.error.URLError: <urlopen error [SSL: CERTIFICATE_VERIFY_FAILED] certificate verify failed...>`。即使不报错，`![图片]({...})` 里的值不是 URL，屏幕上是乱码，图片不显示。

正确做法：只把 URL 拎出来。生图插件（CogView 等）返回的已是可直接访问的公网 URL，带签名参数，不需要二次处理。

```python
def main(**kwargs):
    data = kwargs.get("data") or kwargs.get("arg1") or []
    image_url = ""
    try:
        if isinstance(data, list) and len(data) > 0:
            image_url = data[0].get("url", "")
        elif isinstance(data, dict):
            image_url = data.get("data", [{}])[0].get("url", "")
    except Exception:
        image_url = ""
    return {"image_url": image_url}
```

### 预览区显示代码是正常现象

带签名的图片 URL 常有 200+ 字符，预览窗口里折成好几行，`}:widget_pic_end` 被挤到最底部，容易误判「尾巴丢了」；Dify 前端会把紧贴 URL 的 `}:widget_pic_end` 一起染成蓝色链接，视觉上和 URL 参数混在一起；但后端纯文本里这几个字符一个没少，研发用正则从 `widget_pic_start:` 匹配到 `:widget_pic_end` 能正常取到。验证方式：复制预览区文本，检查 `:widget_pic_end` 是否存在，不靠肉眼判断。

变量插错是高频错误：把 `page_title`（网页标题）插进 `__image_url`，前端拿到文字不是 URL，图片加载不出来。图片组件只认图片 URL 变量。

### 生成类任务的耗时与中断

生图模型单次耗时 4~8 秒。数字人客户端有保活心跳，长时间无语音输出会被判定任务中断，甚至死机。两个必做动作：拆双轨（见第七节），让文本先出、图片后到；不在前端等图，图片信令晚到正常，收到就弹，收不到不弹，不做同步阻塞。

## 七、音画双轨与 SSML

### 长任务先出文本

```text
               ┌── 轨道 A（声音轨·0.3 秒极速出声） → 【LLM 绘本故事创作】 → 【直接回复 1：播报故事】
【用户输入】 ──┤
               └── 轨道 B（视觉轨·后台异步生图） → 【COGVIEW 绘图】 → 【代码执行】 → 【直接回复 2：弹图片信令】
```

轨道 A：模型选首字延迟 200~300ms 的；输出纯文本，禁 Markdown、禁括号动作描写，80~120 字，朗读 10~15 秒；直接插 `{{LLM 绘本故事创作.text}}`，不等图片。

轨道 B：保留完整生图链路，末端【直接回复 2】只放信令：

```text
widget_pic_start:{__image_url:{{代码执行 / image_url}}}:widget_pic_end
```

时序效果：第 0.3 秒数字人开口，第 4~5 秒图片信令到达，屏幕淡入插画，声音不断。同一思路适用于任何慢接口：先出一句固定文本，再走慢链路，实现方式是在分支节点前加一个并行的【直接回复】节点先吐固定文本，不靠模型生成垫话。

### SSML 跳转事件

魔珐 SDK 把 SSML 里的 `<ue4event>` 当作时间轴同步事件抛给前端。

```xml
<speak>
    好的，正在为您打开商品选购页面，请在屏幕上挑选您喜欢的商品。
    <break time="1000ms"/>
    <ue4event>
        <type>page_jump</type>
        <data>
            <target_page>shopping_page</target_page>
            <transition>fade_in</transition>
        </data>
    </ue4event>
</speak>
```

| 标签 | 含义 |
|---|---|
| `<type>page_jump</type>` | 事件类型，前端按此分发 |
| `<target_page>shopping_page</target_page>` | 目标页面，可填路由名如 `/shop` |
| `<transition>fade_in</transition>` | 转场动画，选填 |
| `<break time="1000ms"/>` | 静音停顿，位置在事件之前 |

### break 时序

SSML 标签严格挂在音频时间轴上：第 0 秒数字人开始念台词；台词念完遇到 `<break time="1000ms"/>`，静音 1 秒，留给访客反应时间；静音结束，时间轴走到 `<ue4event>`，SDK 抛出事件；前端收到事件执行切页。

`<break>` 必须放在 `<ue4event>` **之前**。放错位置会导致台词念到一半屏幕跳走、声音被掐断。

需要先做动作再切页时，在 `<break>` 前插一个 `<ue4event>`，`<type>ka</type>`，`<data><action_semantic>point_screen</action_semantic></data>`，再跟 `<break>` 和 `page_jump` 事件。`point_screen` 需替换为动作库里实际存在的预置动作名，如 `guide_right`、`point_forward`。

### 转场动画参数

| 取值 | 效果 | 适用 |
|---|---|---|
| `fade_in` | 淡入淡出 | 默认推荐，稳重不晃眼 |
| `slide_left` | 从右往左滑入 | 窄屏、手机形态 |
| `slide_up` | 从底部向上推出 | 类手机弹窗 |
| `none` / `instant` | 瞬间切换 | 追求极致响应 |

不加此参数不影响跳转逻辑，前端未实现动画时可整行删除。商品名等动态内容用代码节点拼 SSML，在 `<data>` 里追加 `<keyword>{goods_name}</keyword>`。SSML 由【直接回复】节点直出时，Dify 原样经 SSE 下发，不做转义，零篡改。
