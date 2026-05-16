# FILE `plugins/agent-sdk-dev/README.md` 研究文档

## 场景与职责

`plugins/agent-sdk-dev/README.md` 是 `agent-sdk-dev` 插件的人类可读入口文档，负责把“命令驱动脚手架 + 双语言 verifier 验收”的能力说明成可执行的用户心智模型，而不是直接承载可执行逻辑。

其在上下文中的职责分层如下：

1. 插件定位与能力边界说明  
- 声明插件用于 Python/TypeScript 的 Agent SDK 项目创建与验证（`plugins/agent-sdk-dev/README.md:1-8`）。
- 明确核心组件是一个命令（`/new-sdk-app`）和两个 agent（`agent-sdk-verifier-py`、`agent-sdk-verifier-ts`）（`plugins/agent-sdk-dev/README.md:11-116`）。

2. 用户流程引导  
- 以“创建 -> 交互回答 -> 自动验证 -> 本地运行 -> 变更后复验”组织工作流示例（`plugins/agent-sdk-dev/README.md:117-148`）。

3. 运维与实践约束传达  
- 给出最佳实践（最新版 SDK、密钥安全、TS typecheck、部署前验证）（`plugins/agent-sdk-dev/README.md:157-164`）。
- 给出故障排查入口（TS 类型错误、Python 导入错误、Verifier warnings）（`plugins/agent-sdk-dev/README.md:173-200`）。

4. 在插件系统中的位置（调用方/被调用方）
- 上游调用方（发现链路）：  
  - 根仓库 README 把用户导向 `plugins/README.md`（`README.md:48-50`）。  
  - `plugins/README.md` 列出该插件与其命令/agent 能力摘要（`plugins/README.md:15`）。  
  - `.claude-plugin/marketplace.json` 注册该插件源目录 `./plugins/agent-sdk-dev`（`.claude-plugin/marketplace.json:12-16`）。
- 下游被调用方（由 README 所描述）：  
  - 命令协议：`plugins/agent-sdk-dev/commands/new-sdk-app.md:1-176`。  
  - 验证 agent 协议：`plugins/agent-sdk-dev/agents/agent-sdk-verifier-ts.md:1-145`、`plugins/agent-sdk-dev/agents/agent-sdk-verifier-py.md:1-140`。  
  - 外部文档与包源：docs.claude.com、npmjs.com、pypi.org（在 README 资源区与命令协议中均被引用）。

## 功能点目的

### 1) 命令 `/new-sdk-app` 的文档化目的

- 目标：把创建 Agent SDK 项目的复杂步骤收敛成统一入口，减少“语言/工具链/模板/版本”选择负担。  
- README 声明：交互提问、安装最新 SDK、生成配置、执行验证并自动调用 verifier（`plugins/agent-sdk-dev/README.md:15-47`）。  
- 实际承载文件：`commands/new-sdk-app.md` 对上述行为给出可执行提示词约束（`plugins/agent-sdk-dev/commands/new-sdk-app.md:27-136`）。

### 2) verifier agents 的文档化目的

- 目标：把“脚手架完成”升级为“可验收状态”并输出结构化报告。  
- README 声明了两类 verifier 的检查维度与输出格式（`plugins/agent-sdk-dev/README.md:50-116`）。  
- 实际承载文件进一步细化检查条目：  
  - TS：安装/tsconfig/SDK 用法/typecheck/安全/报告格式（`plugins/agent-sdk-dev/agents/agent-sdk-verifier-ts.md:13-145`）。  
  - Py：安装/依赖可复现/SDK 用法/导入与语法/安全/报告格式（`plugins/agent-sdk-dev/agents/agent-sdk-verifier-py.md:13-140`）。

### 3) 工作流示例与最佳实践的目的

- 目标：让首次使用者在不阅读命令源码的前提下也能跑通最短路径。  
- README 提供了可复制流程（`plugins/agent-sdk-dev/README.md:121-148`）与安全/质量清单（`plugins/agent-sdk-dev/README.md:157-164`），等价于“操作手册层”。

### 4) Troubleshooting 的目的

- 目标：把最常见失败映射到具体动作（例如 `npx tsc --noEmit`、`pip show claude-agent-sdk`）。  
- 这部分与命令/agent协议形成闭环：  
  - TS 校验动作与命令、TS verifier 一致（`plugins/agent-sdk-dev/README.md:180-182`，`plugins/agent-sdk-dev/commands/new-sdk-app.md:119,164`，`plugins/agent-sdk-dev/agents/agent-sdk-verifier-ts.md:39,103`）。  
  - Python 依赖验证动作与命令一致（`plugins/agent-sdk-dev/README.md:189-191`，`plugins/agent-sdk-dev/commands/new-sdk-app.md:84,87`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 关键流程（README 驱动的执行链）

1. 发现插件  
- 通过 marketplace 注册和插件目录总览可发现 `agent-sdk-dev`（`.claude-plugin/marketplace.json:12-16`，`plugins/README.md:15`）。

2. 用户入口  
- 用户依据 README 调用 `/new-sdk-app [project-name]`（`plugins/agent-sdk-dev/README.md:24-33`）。

3. 交互采集与计划  
- README 列出五个交互输入槽位（语言、项目名、agent 类型、起点、工具链）（`plugins/agent-sdk-dev/README.md:34-39`）。  
- 命令协议要求“一次只问一个问题”并按固定顺序执行（`plugins/agent-sdk-dev/commands/new-sdk-app.md:29-58,174-176`）。

4. 初始化与安装  
- README 宣称“安装最新版本 + 创建配置 + 创建示例 + 环境文件”（`plugins/agent-sdk-dev/README.md:16-22`）。  
- 命令协议将其落地为具体命令和文件操作（`plugins/agent-sdk-dev/commands/new-sdk-app.md:64-105`）。

5. 代码可用性检查与语言分支验收  
- README 宣称自动调用对应 verifier（`plugins/agent-sdk-dev/README.md:22,69,103,134`）。  
- 命令协议在 verification 阶段明确分流到 TS/Py verifier（`plugins/agent-sdk-dev/commands/new-sdk-app.md:128-135`）。

6. 结果消费  
- README 预期 verifier 返回 `PASS | PASS WITH WARNINGS | FAIL` 结构化报告（`plugins/agent-sdk-dev/README.md:77,111`）。  
- agent 协议定义同样状态枚举与报告字段（TS `:115-143`，Py `:110-138`）。

### B. 数据结构与协议模型

1. 文档协议（README）  
- 类型：叙述性 Markdown（非 frontmatter 协议），用于用户教育与能力声明。  
- 结构：Overview -> Features（Command/Agents）-> Workflow -> Installation -> Best Practices -> Resources -> Troubleshooting -> Author/Version。

2. 命令协议（被 README 引用）  
- 文件：`commands/new-sdk-app.md`，frontmatter 字段 `description`、`argument-hint`（`plugins/agent-sdk-dev/commands/new-sdk-app.md:1-4`）。  
- 协议重点：文档先行、最新版优先、逐问交互、验证前不可完成（`plugins/agent-sdk-dev/commands/new-sdk-app.md:10-25,29,117-127,162-176`）。

3. Agent 协议（被 README 引用）  
- 文件：`agents/*.md`，frontmatter 字段 `name`、`description`、`model`（TS `:1-5`，Py `:1-5`）。  
- 共同输出协议：`Overall Status` + `Summary/Critical Issues/Warnings/Passed Checks/Recommendations`。

4. 元数据配置  
- 插件元信息：`plugins/agent-sdk-dev/.claude-plugin/plugin.json`（`name/description/version/author`）（`plugins/agent-sdk-dev/.claude-plugin/plugin.json:1-9`）。  
- README 的 `Author/Version` 与 plugin.json 一致（README `:202-208` 对齐 plugin.json `:4-8`）。

### C. 关键命令与外部协议点

README 间接指向的关键命令（在 `new-sdk-app.md` 中明确）：

- TypeScript：`npm init -y`、`npm install @anthropic-ai/claude-agent-sdk@latest`、`npm list @anthropic-ai/claude-agent-sdk`、`npx tsc --noEmit`（`plugins/agent-sdk-dev/commands/new-sdk-app.md:68,83,86,119`）。  
- Python：`pip install claude-agent-sdk`、`pip show claude-agent-sdk`（`plugins/agent-sdk-dev/commands/new-sdk-app.md:84,87`）。

README 明示的外部协议入口：
- Agent SDK 文档：overview/typescript/python/examples（`plugins/agent-sdk-dev/README.md:168-171`）。  
- API key 获取：在命令协议中指向 `https://console.anthropic.com/`（`plugins/agent-sdk-dev/commands/new-sdk-app.md:100`）。

## 关键代码路径与文件引用

### 目标对象
- `plugins/agent-sdk-dev/README.md:1-208`

### 与目标对象直接耦合的插件内文件
- `plugins/agent-sdk-dev/.claude-plugin/plugin.json:1-9`（元数据）  
- `plugins/agent-sdk-dev/commands/new-sdk-app.md:1-176`（README 对应命令实现协议）  
- `plugins/agent-sdk-dev/agents/agent-sdk-verifier-ts.md:1-145`（TS 验证协议）  
- `plugins/agent-sdk-dev/agents/agent-sdk-verifier-py.md:1-140`（Py 验证协议）

### 上游入口与索引
- `README.md:48-50`（仓库级 plugins 入口）  
- `plugins/README.md:11-27`（插件目录能力索引，含 `agent-sdk-dev` 行）  
- `.claude-plugin/marketplace.json:10-16`（插件源注册）

### 研究流程相关脚本/文档（本次任务上下文）
- `Docs/researches/blueprint_checklist.md`（研究勾选源）  
- `.ops/generate_daily_research_todo.sh:1-42`（由 checklist 生成每日待办）

### 测试与脚本现状（围绕目标文件）
- `plugins/agent-sdk-dev` 目录没有测试代码与自动化脚本；README 描述的是“流程协议能力”，实际执行依赖 Claude 按命令/agent 文档实施。

## 依赖与外部交互

### 1) 内部依赖
- 对命令协议的依赖：README 的 `/new-sdk-app` 能力陈述必须与 `commands/new-sdk-app.md` 保持一致。  
- 对 agent 协议的依赖：README 的 verifier 描述必须与 `agents/*.md` 的检查范围和报告格式一致。  
- 对插件索引的依赖：`plugins/README.md` 与 marketplace 条目影响该 README 的可发现性。

### 2) 外部交互
- 网络文档站点：`docs.claude.com`（SDK 规范对齐）。  
- 包生态站点：`npmjs.com` 与 `pypi.org`（最新版本查询与安装依据）。  
- 本地命令工具：Node/npm/npx、Python/pip、TypeScript 编译链。  
- 环境与安全交互：`.env.example` 与 `.gitignore`（避免泄露 API key）。

### 3) 与调用方/被调用方的交互模式
- 调用方（用户/命令入口）通过 README 获取语义入口与操作路径。  
- 被调用方（命令与 verifier agent）由 README 承诺触发，但实际触发逻辑在命令协议层。  
- 这意味着 README 是“契约说明层”，不是“执行层”。

### 4) 配置、测试、脚本依赖结论
- 配置：`plugin.json` + marketplace 是发现与元数据基础。  
- 测试：本插件无自带测试，质量更多依赖运行时 typecheck/语法检查与 verifier。  
- 脚本：插件内无脚本，研究流程脚本位于 `.ops/`，只影响文档治理，不影响插件运行。

## 风险、边界与改进建议

### 风险

1. 文档承诺与实现漂移风险  
- README 承诺“自动验证/最新版本/完整检查”，但这些是提示词约束，不是仓库内强制执行器；执行一致性依赖运行时模型遵循度。

2. Python 校验表述相对抽象  
- README 写“syntax validation”，而命令协议仅要求“导入和基础语法检查”未固定命令模板（`plugins/agent-sdk-dev/commands/new-sdk-app.md:123-125`），可重复性弱于 TS 分支。

3. 版本与兼容性信息不完整  
- README 强调“latest”，但未给出 Node/Python 版本建议矩阵，首次搭建时容易在环境版本上踩坑。

4. 自动化回归缺口  
- 无脚本化 smoke test 校验“README 描述与命令/agent协议一致性”，改文档或改 prompt 后不易发现偏差。

### 边界

1. README 不包含执行代码  
- 仅负责说明与引导，真正的执行逻辑在 `commands/` 与 `agents/`。

2. README 不负责插件发现机制实现  
- 发现能力由 marketplace 与插件系统提供，README 只承接说明。

3. README 不负责测试保障  
- 当前缺少插件内自动化测试，文档质量更多依赖人工维护和使用反馈。

### 改进建议

1. 增加“README 承诺一致性”检查  
- 在 CI 或研究脚本中加入规则：检查 README 中声明的命令/agent 名称是否存在、关键能力是否在协议文件可追溯。

2. 补充环境兼容矩阵  
- 在 README 增加推荐 Node/Python 版本与常见失败对照（例如 `npx tsc --noEmit` 失败常见原因）。

3. 固化 Python 校验命令模板  
- 在 `new-sdk-app.md` 与 `agent-sdk-verifier-py.md` 加入可重复执行的语法/导入命令建议，提升一致性。

4. 为“自动验证”补充可观测输出约定  
- 在 README 指明自动验证完成时应输出哪些关键字段（状态、关键问题数、建议数），便于用户快速判断是否可继续开发。
