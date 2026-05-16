# plugins/agent-sdk-dev 目录研究（DIR）

## 场景与职责

`plugins/agent-sdk-dev` 是 Claude Code 插件体系里的“Agent SDK 项目初始化 + 质量验收”插件，定位是把“创建 SDK 项目”与“按官方文档做合规检查”串成一个可复用工作流。

- 在插件市场清单中的注册入口：`.claude-plugin/marketplace.json:12-16`
- 在插件总览中的能力声明：`plugins/README.md:15`
- 插件自身元数据：`plugins/agent-sdk-dev/.claude-plugin/plugin.json:1-9`

目录内只有 5 个文件，且全部为声明式 Markdown/JSON（无可执行源码）：
- `commands/new-sdk-app.md`：主工作流命令
- `agents/agent-sdk-verifier-ts.md`：TypeScript 验证代理提示词
- `agents/agent-sdk-verifier-py.md`：Python 验证代理提示词
- `README.md`：插件说明与使用示例
- `.claude-plugin/plugin.json`：插件基础元信息

职责拆分：
1. `/new-sdk-app` 负责交互式收集需求、规划初始化、安装 SDK、创建样例、触发验证。
2. `agent-sdk-verifier-ts/py` 负责按语言侧规范执行“部署前检查”。
3. README 负责把上述流程产品化为可理解的使用路径（创建 -> 自动验证 -> 继续开发）。

## 功能点目的

### 1) `/new-sdk-app` 的目的
- 把“从零搭建 Agent SDK 项目”的步骤标准化，降低用户首次上手成本。
- 强制引导“先读官方文档、再选语言、再确认工具链、再安装最新版本”，避免脚手架过时。
- 在流程末尾自动调用 verifier agent，减少“搭完不验”的遗漏。

证据：
- 文档优先策略：`plugins/agent-sdk-dev/commands/new-sdk-app.md:10-25`
- 逐问式需求收集：`.../new-sdk-app.md:27-58`
- 版本检查 + 安装 + 验证：`.../new-sdk-app.md:74-136`

### 2) `agent-sdk-verifier-ts` 的目的
- 在 TypeScript 侧聚焦 SDK 正确性，而非泛代码风格。
- 要求运行 `npx tsc --noEmit`，将“类型可编译”作为必要验收项。
- 输出结构化结果（PASS/PASS WITH WARNINGS/FAIL）以支持决策。

证据：
- 验证范围定义：`plugins/agent-sdk-dev/agents/agent-sdk-verifier-ts.md:13-77`
- 类型检查要求：`.../agent-sdk-verifier-ts.md:37-43,101-104`
- 报告格式：`.../agent-sdk-verifier-ts.md:111-145`

### 3) `agent-sdk-verifier-py` 的目的
- 在 Python 侧做安装、导入、配置与安全检查，强调可复现环境。
- 明确“遵循 SDK 文档模式”与“环境安全（.env/.gitignore）”两条主线。
- 同样采用结构化报告，便于对齐修复优先级。

证据：
- 验证范围定义：`plugins/agent-sdk-dev/agents/agent-sdk-verifier-py.md:13-72`
- 文档对齐要求：`.../agent-sdk-verifier-py.md:89-93`
- 报告格式：`.../agent-sdk-verifier-py.md:106-140`

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 关键流程（调用链）

1. 插件被发现/安装
- Claude Code 从市场清单读取 `agent-sdk-dev` 的 `source` 指向 `./plugins/agent-sdk-dev`。
- 入口：`.claude-plugin/marketplace.json:12-15`

2. 用户触发 `/new-sdk-app`
- 命令 frontmatter 定义：`description` + `argument-hint`。
- 入口：`plugins/agent-sdk-dev/commands/new-sdk-app.md:1-4`

3. 命令执行阶段（提示词驱动）
- 先 WebFetch 官方文档，按语言加载 TS/Python 参考文档。
- 再按“一次一个问题”收集需求。
- 生成 setup plan，执行初始化、安装、样例创建、环境文件创建。
- 要求验证实际可用性（TS: `npx tsc --noEmit`；Py: 导入/语法检查）。
- 证据：`.../new-sdk-app.md:10-176`

4. 语言分支验收
- TS 项目 -> 启动 `agent-sdk-verifier-ts`
- Python 项目 -> 启动 `agent-sdk-verifier-py`
- 证据：`.../new-sdk-app.md:128-135`

5. verifier 输出结构化报告
- 状态枚举：`PASS | PASS WITH WARNINGS | FAIL`
- 分段：Summary / Critical Issues / Warnings / Passed Checks / Recommendations
- 证据：`agents/agent-sdk-verifier-ts.md:111-145` 与 `agents/agent-sdk-verifier-py.md:106-140`

### B. 数据结构与配置模型

1. 插件元数据（JSON）
- 结构字段：`name`, `description`, `version`, `author`
- 文件：`plugins/agent-sdk-dev/.claude-plugin/plugin.json:1-9`

2. 命令协议（Markdown + YAML frontmatter）
- `description`: 命令说明
- `argument-hint`: 参数提示
- 命令正文是“行为规范/流程约束”，由 Claude 按指令执行
- 文件：`plugins/agent-sdk-dev/commands/new-sdk-app.md:1-176`

3. Agent 协议（Markdown + YAML frontmatter）
- `name`, `description`, `model`
- 正文定义验证边界、禁区、流程、报告模板
- 文件：`plugins/agent-sdk-dev/agents/*.md`

4. 产物侧配置要求（由命令生成/检查）
- TypeScript：`package.json`、`tsconfig.json`、脚本项
- Python：`requirements.txt`/`pyproject.toml`
- 安全：`.env.example`、`.gitignore` 中 `.env`
- 证据：`.../new-sdk-app.md:64-105`，`.../agent-sdk-verifier-ts.md:44-55`，`.../agent-sdk-verifier-py.md:44-49`

### C. 关键命令与协议要求

`/new-sdk-app` 要求使用的外部命令/检查点（通过提示词驱动执行）：
- TypeScript
  - 初始化：`npm init -y`
  - 安装：`npm install @anthropic-ai/claude-agent-sdk@latest`
  - 版本核验：`npm list @anthropic-ai/claude-agent-sdk`
  - 类型检查：`npx tsc --noEmit`
- Python
  - 安装：`pip install claude-agent-sdk`
  - 版本核验：`pip show claude-agent-sdk`

证据：`plugins/agent-sdk-dev/commands/new-sdk-app.md:68,83-87,119,164-166`

## 关键代码路径与文件引用

### 目录内核心路径
- 插件元信息：`plugins/agent-sdk-dev/.claude-plugin/plugin.json:1-9`
- 使用说明与定位：`plugins/agent-sdk-dev/README.md:1-208`
- 主命令：`plugins/agent-sdk-dev/commands/new-sdk-app.md:1-176`
- TS 验证 agent：`plugins/agent-sdk-dev/agents/agent-sdk-verifier-ts.md:1-145`
- Python 验证 agent：`plugins/agent-sdk-dev/agents/agent-sdk-verifier-py.md:1-140`

### 上游调用方（上下文依赖）
- 仓库级插件入口说明：`README.md:48-50`
- 插件目录总览（包含本插件）：`plugins/README.md:11-27`
- 市场注册（source 指向本目录）：`.claude-plugin/marketplace.json:10-16`

### 被调用方（本插件驱动的下游）
- 官方文档站（WebFetch/WebSearch）：
  - `https://docs.claude.com/en/api/agent-sdk/overview`
  - `https://docs.claude.com/en/api/agent-sdk/typescript`
  - `https://docs.claude.com/en/api/agent-sdk/python`
- 包管理与版本源：
  - npm 包页：`https://www.npmjs.com/package/@anthropic-ai/claude-agent-sdk`
  - PyPI 包页：`https://pypi.org/project/claude-agent-sdk/`
- 本地工具链：`npm`, `npx`, `pip` 等（由命令提示词要求触发）

### 测试与脚本上下文
- `plugins/agent-sdk-dev` 目录本身没有 `.sh/.py/.ts` 可执行脚本，也没有单元测试文件（仅文档化协议）。
- 因此其质量保障主要依赖：
  - 命令内要求的即时验证步骤（如 `npx tsc --noEmit`）
  - verifier agent 的结构化审查流程

## 依赖与外部交互

### 1) 运行时依赖
- Claude Code 插件机制（命令/agent 发现与触发）
- 网络可达性：docs.claude.com、npmjs.com、pypi.org
- 本地开发工具：Node/npm/npx 或 Python/pip

### 2) 文件系统交互
`/new-sdk-app` 指令会在用户项目目录创建/修改：
- TS：`package.json`, `tsconfig.json`, `index.ts`/`src/index.ts`
- Py：`requirements.txt` 或 `pyproject.toml`, `main.py`
- 通用：`.env.example`, `.gitignore`

相关约束证据：`plugins/agent-sdk-dev/commands/new-sdk-app.md:64-105`

### 3) 文档/协议耦合
- 本插件强耦合“官方 Agent SDK 文档最新状态”（明确要求先读 docs 再实施）。
- 若官方 API 或推荐模式变化，本插件行为会随执行时抓取的文档变化而变化。

## 风险、边界与改进建议

### 风险与边界

1. 文档驱动强，确定性弱
- 目录内无可执行脚手架代码，核心逻辑靠提示词约束，执行结果受模型行为与上下文影响。
- 影响：同一请求在不同会话下可能生成略有差异的项目结构。

2. 外部网络依赖重
- `new-sdk-app` 明确依赖 WebFetch/WebSearch 与 npm/PyPI 页面。
- 离线或受限网络环境会显著降低可用性。

3. 自动验证是“约定式”，不是强制编排代码
- 命令中写明“应启动 verifier agent”，但没有目录内脚本保证一定被调用。
- 需要执行代理严格遵循提示词。

4. 元数据一致性存在轻微不齐
- marketplace 对 `agent-sdk-dev` 仅给出 `name/description/source/category`，未像多数插件一样显式写 `version/author`（虽然插件自身 `plugin.json` 有）。
- 证据：`.claude-plugin/marketplace.json:12-16` 对比 `17-148` 其他插件条目。

5. 自动化测试缺口
- 目录内无回归脚本/样例快照测试，无法直接做“改 prompt 后行为稳定性”验证。

### 改进建议

1. 增加最小可执行回归脚本
- 在插件目录新增 `scripts/`，用固定输入驱动 `/new-sdk-app` 流程并校验产物关键文件（可先做 smoke test）。

2. 为命令与 agent frontmatter 增加 `allowed-tools`
- 将 WebFetch/WebSearch/Bash/Edit 等工具权限显式化，减少执行时漂移。

3. 固化验收清单为机器可读规则
- 把 verifier 的关键检查项抽到可复用 checklist（如 JSON/YAML），便于未来自动比对。

4. 增加离线降级路径
- 当外网不可达时，允许基于本地内置模板完成初始化，并在结果中标记“未做在线最新版校验”。

5. 补充“版本兼容矩阵”
- 在 README 里明确 Node/Python 推荐版本与最低版本，减少用户在安装阶段踩坑。

