---
title: "Harness Engineering"
summary: "设计让 AI 编程 Agent 可靠运作的环境、约束和反馈循环的工程实践，以及 OpenClaw 如何围绕这一理念构建"
read_when:
  - 理解仓库为何以现在的方式组织
  - 为新模块添加 AGENTS.md、Linter 或边界指南
  - 入门 Agent 优先的开发工作流
  - 设计一个将由 Agent 维护的新子系统
---

# Harness Engineering

2026 年 2 月，OpenAI 发表了文章 ["Harness engineering: leveraging Codex in an agent-first world"](https://openai.com/index/harness-engineering/)，记录了一个三人团队在五个月内用零行手写代码构建出百万行内部产品的经历。他们将这一实践命名为 **Harness Engineering**：设计让 AI 编程 Agent 在规模化场景下可靠运作的环境、约束和反馈循环。

这个名字源自马具（harness）。马具不是用来限制马的力量的，而是把马的力量引导到有生产力的工作上。AI Agent 同理：harness 不是笼子，而是导轨，把原始的生成能力转化为聚焦的、可维护的输出。

OpenClaw 正是围绕这一理念组织的。本文档解释 harness engineering 的三大支柱，并将每个支柱映射到本仓库中的具体机制。

---

## 三大支柱

OpenAI 的框架描述了三个相互强化的组成部分：

| 支柱 | 作用 |
|---|---|
| **Context Engineering（上下文工程）** | 给 Agent 提供正确行动所需的知识 |
| **Architectural Constraints（架构约束）** | 从机制上阻止 Agent 生成无效或不可维护的代码 |
| **Entropy Management（熵管理）** | 定期修复偏差，防止其累积扩大 |

---

## 支柱一：Context Engineering

> "任何 Agent 在上下文中无法访问到的东西，对它来说都不存在。"
> — OpenAI Harness Engineering 文章

上下文工程是将 Agent 所需知识直接嵌入仓库的实践——不是放在外部 Wiki 或口头交接中，而是放在 Agent 每次会话开始时自动读取的文件里。

### CLAUDE.md 与 AGENTS.md

根目录的 `CLAUDE.md` 是本仓库的主要上下文文档，涵盖：

- 项目结构与模块组织
- 架构边界规则及其背后的设计理由
- 命名规范、语言选择与拼写标准
- 构建、测试和开发命令
- 编码风格与护栏（禁止 `@ts-nocheck`、禁止原型变异、禁止 `any`）
- 发布、安全和多 Agent 协作安全规程

每个主要子系统也有各自的局部边界指南：

```
extensions/AGENTS.md
src/plugin-sdk/AGENTS.md
src/channels/AGENTS.md
src/plugins/AGENTS.md
src/gateway/protocol/AGENTS.md
```

这些文件实现了**渐进式披露**：一个在 `src/channels/` 工作的 Agent 读取频道边界指南，获得该表面精确的规则，无需浏览完整的仓库上下文。

仓库约定：每当新增一个 `AGENTS.md`，都要在同级目录创建一个 `CLAUDE.md` 符号链接，使 Claude Code 和其他 Agent 运行时都能读到同一份文件。

### 文档即结构化上下文

`docs/` 目录树不只是面向用户的帮助文档——它是 Agent 在生成或修改面向用户的内容时所参考的结构化上下文层。Mintlify 规范（根相对链接、不加 `.md` 扩展名、标题中不用破折号、使用美式拼写）的存在，部分原因正是：一致的结构让文档成为可靠的 Agent 上下文。

插件公共合约文档位于 `docs/plugins/`：

```
docs/plugins/building-plugins.md
docs/plugins/architecture.md
docs/plugins/sdk-overview.md
docs/plugins/manifest.md
docs/plugins/sdk-channel-plugins.md
docs/plugins/sdk-provider-plugins.md
```

这个文档集是任何 Agent 生成或修改插件代码时的权威上下文。

### 动态上下文：可观测性与工具

静态文档告诉 Agent 关于结构的信息；动态上下文告诉 Agent 关于状态的信息。OpenClaw 通过以下方式暴露运行时状态：

- `openclaw channels status --probe` — 实时频道健康状态
- `openclaw doctor` — 配置与迁移诊断
- 通过 `scripts/clawlog.sh` 查询 Gateway 日志 — 按子系统、分类或时间段过滤的统一日志查询

当 Agent 调试频道或 Gateway 问题时，它直接读取这些输出，而不依赖可能过时的文档。

### 反馈循环：Agent 遇困时更新上下文

OpenAI 文章明确阐述了核心 harness engineering 循环：

> "当 Agent 遇到困难时，我们将其视为信号：找出缺少什么——工具、护栏、文档——然后反馈回去。"

在本仓库中，这个循环通过以下方式落地：

- 当 Agent 触发导入边界违规时，错误信息会引用相关 `AGENTS.md` 章节
- 当文档中新增技术术语或页面标题时，必须先在 i18n 词汇表（`docs/.i18n/glossary.zh-CN.json`）中注册，才能重新运行翻译流水线——这强制形成了文档优先的习惯
- 当配置 schema 或公共插件 SDK 表面发生变化时，必须同步更新 `docs/.generated/` 中对应的基准文件，由 `pnpm config:docs:check` 和 `pnpm plugin-sdk:api:check` 检查

---

## 支柱二：Architectural Constraints

> "严格的边界和可预测的结构，成倍放大了 Agent 的效率。"
> — OpenAI Harness Engineering 文章

反直觉地，让 Agent 更高效的方法是**缩小**其解空间，而不是扩大它。在无约束的代码库中，Agent 对有效路径和无效路径的探索是等价的。在受约束的代码库中，Agent 可以可靠地遵循模式，生成能通过审查的输出。

OpenClaw 在四个层面执行约束。

### 第一层：格式化工具与 Linter（确定性）

`pnpm check` 在每次编辑时运行 Oxlint 和 Oxfmt。这些工具是确定性的：无论是谁（或什么）生成了代码，它们产生的结果相同。关键约束包括：

- 禁止 `@ts-nocheck` 或未经说明的内联 lint 压制
- 不允许 `no-explicit-any` 例外——优先使用 `unknown` 或窄化的适配器
- 禁止原型变异（`applyPrototypeMixins`、对 `.prototype` 的 `Object.defineProperty`）
- 格式化不容协商：pre-commit hook 在 `pnpm check` 之前运行 `pnpm format`

确定性 Linter 承担一个特定的 harness 功能：其**错误信息同时充当修复指令**。一个生成了违反 Oxlint 规则代码的 Agent，会收到它能直接采取行动的错误信息，无需人工介入。

### 第二层：导入边界规则（结构性）

模块边界系统是本代码库中最重要的约束层之一。规则如下：

- 扩展生产代码只能从 `openclaw/plugin-sdk/*` 和本地 `api.ts`/`runtime-api.ts` 桶文件导入——**绝不**从 `src/**` 或其他扩展的 `src/**` 导入
- 核心代码和测试不得深度导入打包插件的内部实现（`extensions/<id>/src/**`）
- 扩展不得使用解析到自身包根目录之外的相对导入
- 生产代码路径中，同一模块不能同时使用 `await import("x")` 和静态 `import ... from "x"`

这些规则以书面政策的形式存在于 `CLAUDE.md`，以 TypeScript 路径别名的形式存在于 `vitest.config.ts`，并作为架构测试在 `check-additional` CI 门控中强制执行。

### 第三层：CI 门控（分层验证）

验证系统有三个明确作用域的层级：

| 门控 | 命令 | 作用域 |
|---|---|---|
| 本地开发 | `pnpm check` | 格式 + lint + 类型检查；快速循环 |
| 落地基准 | `pnpm check && pnpm test` | 推送前全套检查 |
| 硬性门控 | `pnpm build` | 当改动影响构建产物、打包、发布表面时必须通过 |
| CI 架构 | `check-additional` workflow | 导入边界、架构政策守卫 |

这种分离是刻意为之的。架构政策守卫被排除在默认本地循环（`pnpm check`）之外，以避免拖慢编辑节奏，但在每次 CI 推送时运行。这意味着 Agent 可以在本地快速迭代，而 CI 门控会在结构性违规落地之前将其捕获。

### 第四层：合约测试与偏差检测

已发布的表面——插件 SDK API 和配置 schema——由基准文件和偏差检查命令保护：

```
docs/.generated/               # 基准文件
pnpm config:docs:gen           # 重新生成配置 schema 文档
pnpm config:docs:check         # 检测 schema 偏差
pnpm plugin-sdk:api:gen        # 重新生成插件 SDK API 基准
pnpm plugin-sdk:api:check      # 检测 SDK 表面偏差
```

任何未反映在基准文件中的表面变更都会导致 CI 失败。这个约束防止 Agent 静默地扩大或缩小已发布的合约。

仓库不变式测试还强制执行插件命名一致性：`openclaw.plugin.json:id`、`extensions/<id>`、`openclaw.install.npmSpec` 和 `openclaw.channel.id` 必须保持对齐。

### 逃生舱口：`FAST_COMMIT`

严格的约束需要一个显式的逃生舱口，用于等效验证已在其他地方完成的场景。`FAST_COMMIT=1` 环境变量会跳过 pre-commit hook 中全仓库范围的格式化和检查。关键设计原则：这个逃生舱口是具名的、显式的、有作用域的——它不降低 CI 的验证基准，使用时预期配合有针对性的本地验证。

---

## 支柱三：Entropy Management

> "我们定期运行 Agent，检测文档不一致、命名违规和架构约束偏差——这是主动的垃圾回收。"
> — OpenAI Harness Engineering 文章

在 Agent 优先的代码库中，熵以可预测的模式累积：文档落后于代码、命名约定在边缘偏移、架构边界被局部侵蚀。熵管理是定期运行检查来检测并修复这种偏差的实践，在其累积扩大之前就加以处理。

### 自动偏差检测

OpenClaw 有几个自动化的熵管理机制：

**配置 schema 与插件 SDK 偏差检测：**
```bash
pnpm config:docs:check
pnpm plugin-sdk:api:check
```
这些在 CI 中运行，如果生成的文档与实时 schema 或 SDK 表面不一致则失败。

**i18n 词汇表强制：**
```bash
pnpm docs:check-i18n-glossary
```
强制要求变更文件中新出现的英文文档标题或短标签，在重新运行 i18n 流水线之前必须有对应的翻译词条。防止文档翻译静默偏移。

**Pre-commit hook：**
由 `prek install` 安装的 hook 在每次提交时运行 `pnpm format && pnpm check`。在产生的时间点就捕获格式和 lint 方面的熵，而不是让其累积。

### 人在循环中的熵管理

某些熵类别需要自动化工具无法提供的判断力。OpenAI 框架将工程师的角色描述为"架构守门人，而非逐行审查者"——聚焦于高层结构和长期可维护性。

在 OpenClaw 中，这体现在：

- **Changelog 纪律**：仅记录面向用户的变更；不记录内部/元信息；新条目追加到章节末尾（不插入到顶部）；不重复贡献者署名
- **依赖约束**：禁止更新 Carbon；已打补丁的依赖使用精确版本；新补丁需要明确批准
- **PR 合并纪律**：推送前 rebase，`main` 上禁止合并提交，批量关闭操作超过 5 个 PR 需明确确认
- **安全表面保护**：`CODEOWNERS` 覆盖的路径受到限制；变更需要列出的负责人参与

### 核心信号：Agent 遇困时更新 harness

最重要的熵管理实践是将 Agent 的失败视为 harness 信号。如果你发现自己反复纠正同一个模式——在审查中、在 CI 中、或在 Agent 输出里——正确的响应不是修正输出，而是改进 harness：

- 补充或加强相关 `AGENTS.md` 章节
- 强化 Linter 规则
- 添加合约测试
- 改善 CI 失败的错误信息

这个反馈循环是 harness engineering 区别于传统代码审查的地方：**投入是复利的**。每一次 harness 改进都让未来所有 Agent 运行更加可靠。

---

## 三大支柱的协同

三大支柱相互强化。上下文工程给 Agent 提供正确行动的信息；架构约束在上下文不足时从机制上兜底；熵管理随着代码库演进持续保持上下文和约束的有效性。

下表将 OpenAI 框架中的概念映射到本仓库的具体文件和命令：

| 概念 | OpenClaw 实现 |
|---|---|
| AGENTS.md 上下文文件 | `CLAUDE.md`、`extensions/AGENTS.md`、`src/plugin-sdk/AGENTS.md`、`src/channels/AGENTS.md`、`src/plugins/AGENTS.md`、`src/gateway/protocol/AGENTS.md` |
| 公共合约文档 | `docs/plugins/`（SDK、manifest、频道插件、提供商插件） |
| 动态上下文/可观测性 | `openclaw channels status --probe`、`openclaw doctor`、`scripts/clawlog.sh` |
| 确定性 Linter | Oxlint + Oxfmt，通过 `pnpm check` 运行 |
| 结构性测试/边界执行 | `check-additional` CI workflow，TypeScript 路径别名 |
| Pre-commit hook | `prek install` → 运行 `pnpm format && pnpm check` |
| 逃生舱口 | `FAST_COMMIT=1` |
| 合约偏差检测 | `pnpm config:docs:check`、`pnpm plugin-sdk:api:check` |
| i18n 熵检测 | `pnpm docs:check-i18n-glossary` |
| 插件命名不变式 | 通过 `check-additional` 的仓库不变式测试 |
| 反馈循环文档化 | 每次 Agent 失败模式出现后，在 `CLAUDE.md` 和子系统 `AGENTS.md` 中添加条目 |

---

## Harness 贡献者最佳实践

### 先写上下文，再写代码

在实现一个新模块之前，先写它的 `AGENTS.md` 边界指南。描述该模块拥有什么、不得导入什么，以及当 Agent 需要新接缝时应该怎么做。在第一个 Agent 接触代码之前写好的上下文，能预防最常见的失败模式。

### 让约束违规的错误信息自解释

产生"意外导入"的 Linter 规则是一个约束；产生"扩展必须从 `openclaw/plugin-sdk/*` 导入——参见 `src/plugin-sdk/AGENTS.md`"的 Linter 规则才是一个 harness。在错误信息质量上投入。

### 把架构门控与编辑循环分开

保持默认本地开发门控的快速（`pnpm check`）。把架构政策执行留给 CI 门控（`check-additional`）。慢速的本地门控会让开发者和 Agent 都倾向于绕过它。

### 把反复出现的 Agent 错误视为 harness 缺口

如果你发现自己纠正同一个模式超过两次——在审查中、在 CI 中或在 Agent 输出里——停下来修复 harness。添加规则、加强文档、增加合约测试。这个投入会回报到每一次未来的运行中。

### 保持生成文件与代码库共同版本化

生成的基准文件（`docs/.generated/`）存在于仓库中并纳入版本控制。这使偏差在 diff 中立即可见，并保持 harness 的诚实性：未反映在基准文件中的表面变更会导致 CI 失败，而不是静默回归。

---

## 延伸阅读

- [测试](/help/testing) — 测试套件、命令和 Docker 运行器（支撑验证步骤的测试基础设施）
- [插件架构](/plugins/architecture) — 架构约束所执行的边界模型
- [构建插件](/plugins/building-plugins) — 如何为新插件表面添加上下文和约束
- [Gateway 协议](/gateway/protocol) — 协议约束所保护的合约边界
