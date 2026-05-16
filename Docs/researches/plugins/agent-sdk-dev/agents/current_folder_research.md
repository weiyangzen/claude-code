# plugins/agent-sdk-dev/agents 目录研究（DIR）

## 场景与职责

`plugins/agent-sdk-dev/agents` 是 `agent-sdk-dev` 插件中的“语言分支验收层”，以两个 verifier agent（TS/Python）承接 `/new-sdk-app` 在项目初始化后的质量把关。

该目录仅包含两个 agent 定义文件：
- `plugins/agent-sdk-dev/agents/agent-sdk-verifier-ts.md`
- `plugins/agent-sdk-dev/agents/agent-sdk-verifier-py.md`

在插件整体链路中的位置：
1. 用户通过 `/new-sdk-app` 交互式创建 Agent SDK 项目。
2. 命令在“Verification”阶段按语言分支启动对应 verifier agent。参考 `plugins/agent-sdk-dev/commands/new-sdk-app.md:128-135`。
3. verifier 输出统一结构化报告（PASS/PASS WITH WARNINGS/FAIL + 分项），用于决定是否可继续交付。参考：
   - `plugins/agent-sdk-dev/agents/agent-sdk-verifier-ts.md:111-145`
   - `plugins/agent-sdk-dev/agents/agent-sdk-verifier-py.md:106-140`

职责边界：
- 负责 SDK 使用正确性、配置完整性、安全与可运行性核查。
- 明确不负责通用代码风格偏好（TS/Python 各自都给出 non-goals）。参考：
  - TS non-goals: `agent-sdk-verifier-ts.md:78-83`
  - Python non-goals: `agent-sdk-verifier-py.md:73-78`

## 功能点目的

### 1) `agent-sdk-verifier-ts` 的目的

核心目的是让 TypeScript Agent SDK 项目在“可编译 + SDK 用法合规 + 运行前安全检查”三条线上达到可发布状态。

关键检查面：
- SDK 安装与版本新旧、`package.json` 模块类型、Node 版本约束。`agent-sdk-verifier-ts.md:13-19`
- `tsconfig.json` 与 ESM/模块解析设置。`agent-sdk-verifier-ts.md:20-25`
- SDK 调用模式（初始化、参数、响应处理、权限、MCP）。`agent-sdk-verifier-ts.md:27-35`
- 强制类型检查动作：`npx tsc --noEmit`。`agent-sdk-verifier-ts.md:37-43,101-104`
- 环境与安全：`.env.example`、`.gitignore`、禁止硬编码 API Key。`agent-sdk-verifier-ts.md:50-55`

### 2) `agent-sdk-verifier-py` 的目的

核心目的是让 Python Agent SDK 项目在“环境可复现 + SDK 导入/调用正确 + 安全/文档可交接”上通过验收。

关键检查面：
- SDK 安装与版本、Python 版本要求、虚拟环境建议。`agent-sdk-verifier-py.md:13-19`
- 依赖声明与环境可复现（`requirements.txt`/`pyproject.toml`）。`agent-sdk-verifier-py.md:20-26`
- SDK 用法、权限、MCP 与会话模式检查。`agent-sdk-verifier-py.md:27-35,51-58`
- 导入与语法层校验（提示词层要求）。`agent-sdk-verifier-py.md:95-100`
- 环境安全与文档完整性检查。`agent-sdk-verifier-py.md:44-49,67-71`

### 3) 双 agent 共同目的

- 把 `/new-sdk-app` 产物从“已生成”提升到“可验收”，避免脚手架完成即结束。
- 固化统一报告接口，便于命令调用方和用户快速定位问题优先级。
- 把“对齐官方文档”纳入执行流程，而不是可选项。参考：
  - TS 文档对齐：`agent-sdk-verifier-ts.md:95-99`
  - Python 文档对齐：`agent-sdk-verifier-py.md:89-93`

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 关键流程（调用方 -> 目标目录 -> 被调用对象）

1. 上游调用方：`/new-sdk-app`
- 命令定义了在项目生成后进入验证阶段，并按语言启动对应 verifier。`plugins/agent-sdk-dev/commands/new-sdk-app.md:128-135`

2. 目标目录执行单元：`agents/*.md`
- 每个文件通过 frontmatter 声明 `name/description/model`，由 Claude Code 作为可调用 agent 载入。参考：
  - TS: `agent-sdk-verifier-ts.md:1-5`
  - Py: `agent-sdk-verifier-py.md:1-5`

3. 下游交互对象：
- 目标项目文件系统（读取 package/tsconfig/requirements 等）。
- 终端命令（TS 至少要求执行 `npx tsc --noEmit`）。`agent-sdk-verifier-ts.md:39,103`
- 官方文档站（WebFetch 对照官方 SDK 文档）。
  - TS: `agent-sdk-verifier-ts.md:97`
  - Py: `agent-sdk-verifier-py.md:91`

### B. 协议与数据结构

1. Agent 声明协议（YAML frontmatter）
- 必要字段（当前目录实际使用）：`name`, `description`, `model`。
- 两个 agent 的 `model` 均设为 `sonnet`。`agent-sdk-verifier-ts.md:4`, `agent-sdk-verifier-py.md:4`

2. 过程协议（Prompt Contract）
- 结构固定为：`Verification Focus` -> `What NOT to Focus On` -> `Verification Process` -> `Verification Report Format`。
- 这使 agent 行为具有较高可读性和可维护性，便于在插件内做语言间对齐。

3. 输出协议（报告模型）
- 状态枚举：`PASS | PASS WITH WARNINGS | FAIL`。
- 标准字段：`Summary`、`Critical Issues`、`Warnings`、`Passed Checks`、`Recommendations`。
- 参考：
  - TS: `agent-sdk-verifier-ts.md:115-143`
  - Py: `agent-sdk-verifier-py.md:110-138`

### C. 关键命令与检查项

- TypeScript verifier 要求执行：`npx tsc --noEmit`（硬性验证动作）。`agent-sdk-verifier-ts.md:39,103`
- Python verifier 要求做导入和基础语法检查（目前为策略要求，未指定固定命令模板）。`agent-sdk-verifier-py.md:95-99`
- `/new-sdk-app` 在安装阶段与验证阶段涉及的命令模板：
  - TS: `npm init -y`、`npm install @anthropic-ai/claude-agent-sdk@latest`、`npm list @anthropic-ai/claude-agent-sdk`、`npx tsc --noEmit`
  - Py: `pip install claude-agent-sdk`、`pip show claude-agent-sdk`
  - 参考 `plugins/agent-sdk-dev/commands/new-sdk-app.md:67-87,117-126,132-133`

## 关键代码路径与文件引用

### 目录内（被研究对象）
- `plugins/agent-sdk-dev/agents/agent-sdk-verifier-ts.md:1-145`
- `plugins/agent-sdk-dev/agents/agent-sdk-verifier-py.md:1-140`

### 上游调用方/配置
- 命令调用链入口：`plugins/agent-sdk-dev/commands/new-sdk-app.md:128-135`
- 插件自身元数据：`plugins/agent-sdk-dev/.claude-plugin/plugin.json:1-9`
- 插件说明（声明“自动调用 verifier”）：`plugins/agent-sdk-dev/README.md:11-23,69-73,103-107,133-135`

### 插件发现与仓库级上下文
- 仓库根 README 指向插件体系：`README.md:48-50`
- 插件总览声明该插件含两个 verifier agent：`plugins/README.md:15`
- marketplace 对插件路径注册：`.claude-plugin/marketplace.json:12-16`

### 测试/脚本/文档上下文
- `plugins/agent-sdk-dev/agents` 目录内无测试文件、无脚本文件，仅 agent 协议文档。
- 实际“验收执行”依赖命令提示词和运行时工具能力，而非本目录中的程序代码。

## 依赖与外部交互

### 1) 内部依赖
- 强依赖 `commands/new-sdk-app.md` 的语言分支触发约定；若命令不触发，本目录 agent 不会自动运行。
- 与 `plugins/agent-sdk-dev/README.md` 的用户预期耦合（README 明确承诺自动验证）。

### 2) 外部依赖
- 文档依赖：`docs.claude.com`（TS/Python SDK 页面）用于对齐官方模式。
- 工具链依赖：
  - TS: Node/npm/npx + TypeScript 编译器。
  - Py: Python/pip（以及项目环境中的依赖管理方式）。
- 生态依赖：npm 与 PyPI 的包版本信息（由上游 `/new-sdk-app` 安装流程触发）。

### 3) 文件系统交互对象（间接）
agent 验证目标通常是用户项目目录中的：
- TS: `package.json`, `tsconfig.json`, `index.ts`/`src/*`, `.env.example`, `.gitignore`
- Py: `requirements.txt`/`pyproject.toml`, `main.py`/`app.py`/`src/*`, `.env.example`, `.gitignore`

## 风险、边界与改进建议

### 风险

1. 提示词约束强，机器强制弱
- 当前校验逻辑是自然语言约束，不是可执行规则引擎；输出质量受模型执行一致性影响。

2. Python 校验动作不够可操作化
- 相比 TS 明确 `npx tsc --noEmit`，Python 仅写“检查导入和语法”，缺少标准命令模板（如 `python -m py_compile` 或静态检查命令），导致结果可重复性偏弱。参考：`agent-sdk-verifier-py.md:95-100`

3. 模型版本固定为 `sonnet`
- 两个 agent 统一写死 `model: sonnet`，当平台推荐模型更新时，可能出现能力/成本与组织策略不一致的问题。参考：`agent-sdk-verifier-ts.md:4`, `agent-sdk-verifier-py.md:4`

4. 外网依赖造成脆弱点
- 两个 verifier 都要求 WebFetch 官方文档；在离线或受限网络环境下，会削弱“文档对齐”能力。参考：TS `:97`，Py `:91`

### 边界

- 本目录只定义“怎么审查”，不定义“怎么创建项目”（创建逻辑在 `commands/new-sdk-app.md`）。
- 本目录不包含脚本、测试、模板资产，无法独立完成端到端初始化或回归验证。

### 改进建议

1. 为 Python verifier 补充标准化命令清单
- 在提示词中显式加入建议命令（例如语法编译/依赖检查），与 TS 的 `npx tsc --noEmit` 对齐。

2. 引入机器可读报告格式
- 在保留人类可读段落的同时，增加 JSON 结构模板（状态、问题数组、建议数组），方便自动化流水线消费。

3. 增加最小回归脚本或黄金样例
- 在 `plugins/agent-sdk-dev` 增设 smoke test，固定输入下检查 verifier 报告是否覆盖关键字段。

4. 模型策略改为“可配置优先”
- 将 `model` 固定值升级为可由插件策略注入，减少未来模型迭代时的维护成本。

5. 补充“失败分级处理”规则
- 约定哪些检查项属于阻断发布（Critical）与可延期（Warning），减少不同会话下判定口径波动。
