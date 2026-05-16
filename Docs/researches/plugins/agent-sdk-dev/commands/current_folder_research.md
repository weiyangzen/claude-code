# plugins/agent-sdk-dev/commands 目录研究（DIR）

## 场景与职责

`plugins/agent-sdk-dev/commands` 是 `agent-sdk-dev` 插件的命令入口层，当前仅包含一个命令定义文件 `new-sdk-app.md`，负责把“新建 Claude Agent SDK 项目”的端到端流程编排为可交互执行的协议。

目录清单（无子目录扩展、无脚本）：
- `plugins/agent-sdk-dev/commands/new-sdk-app.md`

在整体插件链路中的职责位置：
1. 插件由 marketplace 注册发现：`.claude-plugin/marketplace.json:12-16`。
2. 插件能力在总览中声明为 `/new-sdk-app` + 双 verifier agent：`plugins/README.md:15`。
3. `commands/new-sdk-app.md` 作为用户主入口，负责需求采集、初始化、安装、验证编排：`plugins/agent-sdk-dev/commands/new-sdk-app.md:1-176`。
4. 命令在验证阶段分流到 `agent-sdk-verifier-ts` 或 `agent-sdk-verifier-py`：`plugins/agent-sdk-dev/commands/new-sdk-app.md:128-135`。

职责边界：
- 负责“创建与编排”而非“底层执行引擎实现”；该目录仅有提示词协议，没有可执行 TS/Python 代码。
- 负责过程约束（提问顺序、版本策略、验收门槛），不直接提供 SDK API 封装层。

## 功能点目的

### 1) 交互式需求采集（一次一问）
- 目的：降低用户输入负担并减少需求歧义，尤其是语言、项目名、用例类型、脚手架粒度与工具链偏好。
- 机制：明确要求按顺序逐个提问并等待回答，避免一次性多问：`new-sdk-app.md:27-58,174-176`。
- 价值：让后续计划可根据回答分支执行，避免“默认假设”导致错误初始化。

### 2) 文档优先与“最新版本优先”
- 目的：把官方文档对齐和最新包版本校验前置，减少过时模板与 API 漂移风险。
- 机制：先读 overview，再按语言读 TS/Python 文档，并按需要补读 permissions/MCP/subagents 等：`new-sdk-app.md:8-25`。
- 版本要求：安装前检查 npm/PyPI 最新版本并告知用户：`new-sdk-app.md:74-79,110,162-169`。

### 3) 脚手架 + 环境安全 + 可运行性闭环
- 目的：不仅“生成文件”，还要求“可验证运行”，将可用性纳入默认交付标准。
- 机制：
  - 项目初始化与配置：`new-sdk-app.md:64-73`
  - 依赖安装与版本确认：`new-sdk-app.md:81-87`
  - 样例入口与错误处理：`new-sdk-app.md:89-95,115-116`
  - 环境文件与密钥实践：`new-sdk-app.md:96-104`
  - TypeScript 强制 typecheck：`new-sdk-app.md:117-123,163-166`

### 4) 与 verifier agent 的验收衔接
- 目的：把“创建完成”升级为“已通过语言侧质量检查”。
- 机制：按语言触发对应 verifier agent：`new-sdk-app.md:128-135`。
- 对应被调方：
  - `plugins/agent-sdk-dev/agents/agent-sdk-verifier-ts.md:1-145`
  - `plugins/agent-sdk-dev/agents/agent-sdk-verifier-py.md:1-140`

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 关键流程（可视为命令状态机）

1. 入口解析阶段
- frontmatter 暴露命令说明与参数提示：`description`、`argument-hint`。证据：`new-sdk-app.md:1-4`。
- 触发方式：用户调用 `/new-sdk-app [project-name]`（README 示例）。证据：`plugins/agent-sdk-dev/README.md:26-33`。

2. 文档同步阶段
- 执行 WebFetch/WebSearch 指令，先读取官方 SDK 文档，再读取相关指南。证据：`new-sdk-app.md:10-25`。

3. 需求采集阶段（严格顺序）
- 五个问题顺序：语言 -> 项目名 -> agent 类型 -> 起步模板 -> 工具链偏好。证据：`new-sdk-app.md:31-57`。
- 条件分支：若有 `$ARGUMENTS` 则跳过项目名提问。证据：`new-sdk-app.md:37-40`。
- 约束：一次只问一个问题。证据：`new-sdk-app.md:29,174-176`。

4. Setup Plan 阶段
- 依据回答生成计划，计划项覆盖初始化、版本检查、安装、starter file、环境配置、可选 `.claude/` 结构。证据：`new-sdk-app.md:60-105`。

5. Implementation 阶段
- 先查最新版本，再执行安装与文件生成，并进行可运行验证。证据：`new-sdk-app.md:106-127`。

6. Verification 阶段
- TS 分支：调用 `agent-sdk-verifier-ts`。
- Python 分支：调用 `agent-sdk-verifier-py`。
- 证据：`new-sdk-app.md:128-135`。

7. 交付引导阶段
- 输出 next steps、文档链接、后续扩展方向（系统提示词、自定义工具、权限、subagents）。证据：`new-sdk-app.md:137-158`。

### B. 数据结构与协议

1. 命令元数据结构（YAML frontmatter）
- 字段：`description`、`argument-hint`。
- 作用：定义 slash command 可发现描述与参数提示。
- 证据：`new-sdk-app.md:1-4`。

2. 会话内隐式数据模型（由命令流程维护）
- 关键槽位：
  - `language`
  - `projectName`（可能来自 `$ARGUMENTS`）
  - `agentType`
  - `startingPoint`
  - `toolingChoice`
- 这些槽位虽未以 JSON 显式定义，但由提问顺序和分支条件确定。证据：`new-sdk-app.md:31-58`。

3. 执行协议（Prompt Contract）
- 关键协议条款：
  - 文档先行：`new-sdk-app.md:10-25`
  - 最新版本优先：`new-sdk-app.md:25,74-79,162-167`
  - 验证通过前不可视为完成：`new-sdk-app.md:117-127,163-166`
  - 逐问交互约束：`new-sdk-app.md:29,174-176`

4. 命令到 agent 的协议衔接
- 命令文本定义“按语言启动 verifier agent”，属于插件内跨组件调用约定。
- Claude Code 的命令开发参考说明：命令可通过 Task tool 触发 plugin agents。证据：`plugins/plugin-dev/skills/command-development/references/plugin-features-reference.md:334-356`。

### C. 关键命令（终端/网络）

TypeScript 分支：
- `npm init -y`
- `npm install @anthropic-ai/claude-agent-sdk@latest`
- `npm list @anthropic-ai/claude-agent-sdk`
- `npx tsc --noEmit`

Python 分支：
- `pip install claude-agent-sdk`
- `pip show claude-agent-sdk`

网络查询与文档来源：
- `https://docs.claude.com/en/api/agent-sdk/overview`
- `https://docs.claude.com/en/api/agent-sdk/typescript`
- `https://docs.claude.com/en/api/agent-sdk/python`
- `https://www.npmjs.com/package/@anthropic-ai/claude-agent-sdk`
- `https://pypi.org/project/claude-agent-sdk/`

命令证据：`new-sdk-app.md:68,76-78,83-87,110,119,132-133,150-151,164-166`。

## 关键代码路径与文件引用

### 目标目录（研究对象）
- `plugins/agent-sdk-dev/commands/new-sdk-app.md:1-176`

### 上游调用方与装配配置
- 插件市场注册：`.claude-plugin/marketplace.json:12-16`
- 插件总览入口：`plugins/README.md:15,49-61`
- 仓库级插件说明入口：`README.md:48-50`
- 插件自身元数据：`plugins/agent-sdk-dev/.claude-plugin/plugin.json:1-9`

### 被调用方（下游）
- TS verifier agent：`plugins/agent-sdk-dev/agents/agent-sdk-verifier-ts.md:1-145`
- Python verifier agent：`plugins/agent-sdk-dev/agents/agent-sdk-verifier-py.md:1-140`
- 命令-代理衔接点：`plugins/agent-sdk-dev/commands/new-sdk-app.md:128-135`

### 文档与使用者预期
- 插件 README 的功能承诺与示例：`plugins/agent-sdk-dev/README.md:11-46,69-73,103-107,123-136,159-180`

### 测试与脚本上下文
- `plugins/agent-sdk-dev/commands` 目录中无测试文件、无脚本文件，仅 Markdown 命令协议。
- `plugins/agent-sdk-dev` 全目录亦无 `.sh/.ts/.py` 实现文件，说明其运行逻辑完全依赖 Claude 执行命令协议而非仓库内可执行模块。

## 依赖与外部交互

### 1) 内部依赖
- 依赖插件系统对 `commands/` 的发现机制（由 marketplace source 指向插件目录）。
- 依赖 `agents/` 中两个 verifier 定义来完成验收闭环。
- 依赖 README 对用户交互预期进行声明（例如“自动验证”）。

### 2) 外部依赖
- 文档源：`docs.claude.com`（overview/TS/Python 与相关指南）。
- 包仓库：`npmjs.com`、`pypi.org`（版本查验与安装参考）。
- 本地工具链：`npm`/`npx`/`pip`/Python/Node 运行环境。

### 3) 文件系统与产物交互
命令会在用户项目目录创建或修改：
- TypeScript：`package.json`、`tsconfig.json`、`index.ts` 或 `src/index.ts`
- Python：`requirements.txt` 或 `pyproject.toml`、`main.py`
- 通用：`.env.example`、`.gitignore`（包含 `.env`）

证据：`new-sdk-app.md:64-105,89-100`。

### 4) 测试/脚本依赖现实
- 该目录没有自带自动化测试脚本，执行质量依赖：
  - 命令内部强制检查条款（TS typecheck / Python 基础检查）
  - verifier agent 的审查质量
- 这使其更像“流程协议层”，而非“可重复执行的脚手架程序层”。

## 风险、边界与改进建议

### 风险

1. 协议强约束但弱确定性
- 逻辑以自然语言提示词表达，缺少机器可验证流程图或固定执行器，同需求在不同会话可能存在产物差异。

2. Python 验证标准不如 TS 具象
- TS 明确 `npx tsc --noEmit`；Python仅要求“导入与语法检查”，缺少固定命令模板，验收重复性较弱。证据：`new-sdk-app.md:123-125`。

3. 对网络与外部服务依赖高
- 文档与版本校验都依赖在线访问；离线/受限网络会削弱命令可靠性。证据：`new-sdk-app.md:10-25,74-79`。

4. 命令-代理衔接描述偏语义化
- 文本写明“Launch verifier agent”，但无显式调用样板（例如固定 Task 调用片段），在实现一致性上存在弹性空间。

5. 自动化回归缺口
- 目录内无脚本化 smoke test，prompt 调整后难以快速验证行为是否回归。

### 边界

1. 本目录只定义 `/new-sdk-app` 行为，不负责 verifier 具体判定细节（在 `agents/`）。
2. 本目录不承载模板资产；生成文件结构依赖执行时模型与用户回答。
3. 本目录不处理插件安装/发现机制实现，仅消费既有插件系统约定。

### 改进建议

1. 增加“机器可执行”最小回归
- 在 `plugins/agent-sdk-dev` 增加 smoke-test 脚本，固定输入后校验关键产物文件与必含字段。

2. 为 Python 分支补齐硬性校验命令
- 在命令或 verifier 中补充可重复的语法/导入命令模板，缩小与 TS 分支验收强度差距。

3. 增强命令到 agent 的调用显式性
- 在 `new-sdk-app.md` 增加更明确的 Task 调用约定示例，减少执行歧义。

4. 增加离线降级策略
- 当 docs/npm/PyPI 不可达时，允许基于本地默认模板继续初始化，并在结果中明确“未完成在线最新版本校验”。

5. 增补兼容性矩阵
- 在插件 README 或命令中明确推荐 Node/Python 版本区间与常见失败排查，降低首次使用失败率。
