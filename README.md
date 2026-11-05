# dify-chatflow-builder

一套来自真实交付项目的 Dify 对话流 / 工作流搭建方法论与踩坑库。

不是官方文档的复述，是从几十个实际项目里提炼出来的：什么配置会生效、什么写法看着对但一定出问题、报错原文对应哪个根因。

## 内容来源

来自四类真实项目的完整搭建与排障记录：

| 项目类型 | 覆盖能力 |
|---|---|
| 电信营业厅销售陪练 | 状态机闭环、角色扮演、判卷打分、薄弱点沉淀 |
| 医院导诊挂号 | 知识库检索、HIS 接口对接、科室映射、兜底话术 |
| 企业前台接待 | 多场景分诊、飞书通知、结构化表单、通讯录穿透 |
| 机场航班查询 / 数字人大屏 | Swagger 接入、JSONPath 抽取、富媒体信令、音画双轨 |

## 目录结构

```
dify-chatflow-builder/
├── SKILL.md                      # 主入口：原则、路由、高频坑速查
├── README.md
├── references/
│   ├── architecture.md           # 形态选型、状态机、多场景分诊
│   ├── variables.md              # 变量体系与生命周期
│   ├── nodes.md                  # 节点配置要点
│   ├── prompt-design.md          # 节点级提示词设计
│   ├── knowledge.md              # 知识库与检索
│   ├── intent-router.md          # 意图分类与兜底
│   ├── integration.md            # HTTP / Swagger / MCP / 飞书
│   ├── rich-media.md             # 富媒体组件信令协议
│   ├── latency-ux.md             # 延迟与等待体验
│   ├── deploy-ops.md             # 私有化部署排障
│   ├── delivery.md               # 命名、注释、交付规范
│   ├── errors.md                 # 错误百科索引（152 条）
│   └── errors/                   # 错误百科分册 ×7
│       ├── 01-architecture.md
│       ├── 02-variables.md
│       ├── 03-prompt.md
│       ├── 04-knowledge-intent.md
│       ├── 05-integration.md
│       ├── 06-rich-media.md
│       └── 07-deploy-delivery.md
└── examples/
    └── minimal-chatflow.md       # 最小可用骨架，从零跑通
```

## 怎么用

### 方式一：支持 Agent Skills 标准的工具（推荐）

这类工具直接读 `SKILL.md`，不需要额外转换：

- **Claude Code**：放到 `.claude/skills/dify-chatflow-builder/`
- **Cursor**：放到 `.cursor/skills/` 或 `.agents/skills/`
- **Codex / Gemini CLI / Copilot**：放到 `.agents/skills/` 或各自的 skills 目录
- **豆包工作**：客户端 → 插件·技能·伙伴 → 技能 → `+ 添加` → 上传技能（选文件夹）
- **WorkBuddy**：Skills 管理 → 导入

### 方式二：平台内共享

- **豆包工作**：把技能挂到「工作伙伴」上，设为团队使用，成员 @ 即可调用，不用各自安装
- **Coze**：保存为技能草稿 → 发布到「我的技能」

### 方式三：不支持 Skill 的工具

把 `SKILL.md` 正文压成一段系统提示词贴进人设框，`references/` 内容按需作为知识库上传。能力会打折，但核心判断逻辑保留。

## 维护约定

- 新踩的坑统一补进 `references/errors.md`，按「现象 / 根因 / 解法 / 验证」四段写，报错原文一字不改，方便检索匹配
- 平台升级导致写法变化时，在对应 reference 文件里标注适用版本
- 修改后提交到 Git，各平台重新导入或挂载即可，不要各自留副本

## 兼容性说明

- 统一使用标准命名 `SKILL.md`。若目标平台要求小写 `skill.md`，复制一份改名即可（GitHub 不允许同目录下两个仅大小写不同的文件）
- frontmatter 只保留 `name` 和 `description`，不使用任何平台专属字段
- 所有内容为纯 Markdown，无脚本依赖，任何工具都能读取
