# interactive-commands.md 研究

## 场景与职责

`plugins/plugin-dev/skills/command-development/references/interactive-commands.md` 定义了 command-development 中“交互式命令协议层”，用于处理纯参数无法表达的复杂决策场景（多选、条件分支、解释型选择、渐进配置）。

其在体系中的职责：

1. 规定何时用 `AskUserQuestion`，何时继续用参数（`15-31`）。
2. 固化问答结构协议（`questions/question/header/options/multiSelect`，`34-60`）。
3. 提供从简单确认到多阶段向导的完整模式库（`269-920`）。
4. 为交互结果落地到配置文件（如 `.claude/*.local.md`）给出流程。

调用方与上下文：

1. `plugins/plugin-dev/skills/command-development/README.md:102` 把该文档列为进阶 references。
2. `plugins/plugin-dev/commands/create-plugin.md:4,13-18,85-99` 的引导式工作流依赖问答思路。
3. 与 `plugin-settings` 的 `.local.md` 模式联动：`plugins/plugin-dev/skills/plugin-settings/SKILL.md:11-19`。

## 功能点目的

1. 决策边界清晰化
- 把“可脚本化输入”与“需要解释和权衡的输入”分开（`17-31`）。

2. 交互协议标准化
- 用统一字段结构降低命令作者随意发挥带来的 UX 不一致。

3. 复杂流程模板化
- 提供 5 类核心模式（确认、多问题、条件、迭代、依赖选择）与高级模式（验证循环、增量构建、上下文感知）。

4. 交互与自动化兼容
- 明确“参数 + 问答”混合模式，保留自动化入口同时支持复杂配置（`871-899`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) AskUserQuestion 数据结构协议

核心结构（`36-60`）：

1. `questions[]`：单次调用可包含多问题。
2. `question`：完整问题文本。
3. `header`：短标签（建议 <=12 字符，`41,467`）。
4. `options[]`：`label + description`。
5. `multiSelect`：是否允许多选。

约束建议（`63-67,235-240,469`）：

1. 每题 2-4 选项。
2. 每次调用 1-4 问题。
3. 自动提供 `Other` 自定义入口。

### 2) 交互流程实现模板

基础流程（`70-147`）：

1. 发问采集。
2. 处理回答。
3. 生成配置（示例写入 `.claude/plugin-name.local.md`）。
4. 确认并引导下一步。

多阶段流程（`149-197`）：

1. Stage 1 采集基础信息。
2. Stage 2 条件分支提问。
3. Stage 3 汇总确认（可修改/重来）。
4. Stage 4 执行落地。

### 3) 模式库实现

1. 确认型：破坏性操作前 yes/no（`271-299`）。
2. 多问题批量采集：一次调用多问题（`300-337`）。
3. 条件分支：按复杂度追问（`339-380`）。
4. 迭代采集：先采集数量，再循环采集明细（`381-425`）。
5. 多选依赖：`multiSelect: true` 生成组合配置（`427-460`）。

高级模式（`541-666`）：

1. 验证循环：配置无效时二次交互修复。
2. 增量构建：阶段性保存 partial 配置。
3. 上下文感知：先 Bash 检测，再动态生成问题。

## 关键代码路径与文件引用

核心文件：

1. `plugins/plugin-dev/skills/command-development/references/interactive-commands.md:1-920`

关键关联路径：

1. `plugins/plugin-dev/skills/command-development/README.md:94-103`
2. `plugins/plugin-dev/commands/create-plugin.md:3-5,24-99`
3. `plugins/plugin-dev/skills/plugin-settings/SKILL.md:11-19,60-171`
4. `plugins/plugin-dev/skills/command-development/references/frontmatter-reference.md:196-323`（`argument-hint` 与 `disable-model-invocation` 配合交互命令设计）

实战样例来源：

1. 多 agent swarm 引导例（`interactive-commands.md:668-778`）

## 依赖与外部交互

1. 强依赖 AskUserQuestion 工具能力与返回结构稳定性。
2. 常与 `Read/Write/Bash` 组合：
- `Write`：写入 `.claude/*.local.md`
- `Read`：读取已有配置
- `Bash`：先做上下文探测（语言、框架、工具存在性）
3. 与外部文件协议交互：
- `.claude/plugin-name.local.md`
- `.daisy/swarm/tasks.md`

## 风险、边界与改进建议

### 风险

1. 文档强调问答结构，但缺少“回答结果标准化映射”规范（例如稳定 key/id 设计），复杂流程下易出现解析歧义。
2. `Other` 自由输入虽提升灵活性，但会破坏后续严格分支判断。
3. 交互链过长（特别是迭代模式）容易造成用户疲劳和中途放弃。

### 边界

1. 本文件只定义交互模式，不实现任何真实解析器或状态机。
2. 不覆盖并发会话冲突与跨会话恢复；这些属于 `advanced-workflows.md` 范畴。

### 改进建议

1. 补充“答案规范化层”模板：将 label/other 输入映射到稳定枚举值。
2. 增加“取消/超时/回退”标准流程段，避免流程悬空。
3. 增加交互回合预算建议（例如 >6 回合必须进入“快速模式”）。
4. 补充与 `disable-model-invocation: true` 的联动建议，防止敏感交互被程序化调用。
