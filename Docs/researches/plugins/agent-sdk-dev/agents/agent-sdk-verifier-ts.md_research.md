# FILE `plugins/agent-sdk-dev/agents/agent-sdk-verifier-ts.md` 研究文档

## 场景与职责

`agent-sdk-verifier-ts.md` 是 `agent-sdk-dev` 插件中负责 TypeScript 项目验收的代理协议文件。它在 frontmatter 中定义可调用标识 `agent-sdk-verifier-ts` 与执行模型 `sonnet`（`plugins/agent-sdk-dev/agents/agent-sdk-verifier-ts.md:1-5`），用于在项目创建/修改后执行 SDK 合规检查。

该文件的系统职责是“将 TypeScript Agent SDK 项目从可生成状态推进到可验证交付状态”。上游由 `/new-sdk-app` 在验证阶段触发（`plugins/agent-sdk-dev/commands/new-sdk-app.md:128-135`），README 在用户层声明其自动运行与输出格式（`plugins/agent-sdk-dev/README.md:83-116`）。

职责边界：
- 重点检查 SDK 配置、tsconfig、类型安全、安全项与文档对齐。
- 明确排除通用代码风格争议（`type` vs `interface`、命名偏好等）（`plugins/agent-sdk-dev/agents/agent-sdk-verifier-ts.md:78-83`）。

## 功能点目的

文件按 9 个 Focus 维度定义了 TypeScript 侧完整验收面（`plugins/agent-sdk-dev/agents/agent-sdk-verifier-ts.md:13-76`）：

1. SDK 安装与配置：检查 `@anthropic-ai/claude-agent-sdk`、版本新旧、`package.json` 的 ESM 类型与 Node 要求（`:15-19`）。
2. TS 配置：`tsconfig.json`、模块解析、target 与 SDK 导入兼容性（`:20-25`）。
3. SDK 调用模式：初始化、参数、响应模式、权限、MCP（`:27-35`）。
4. 类型安全与编译：强制 `npx tsc --noEmit`（`:37-43`）。
5. 脚本与构建：检查 `build/start/typecheck` 脚本及可运行性（`:44-48`）。
6. 环境与安全：`.env.example`、`.gitignore`、密钥治理、API 错误处理（`:50-55`）。
7. 最佳实践：系统提示词、模型选择、权限范围、工具与子代理、会话（`:57-64`）。
8. 功能验证：初始化与执行链路闭环、SDK 错误处理（`:66-71`）。
9. 文档：README 和关键配置说明（`:73-76`）。

其中最关键目标是把“类型可编译”作为强约束，减少运行时才暴露的 SDK 集成问题。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 关键流程

该 verifier 定义了 4 步执行流程（`plugins/agent-sdk-dev/agents/agent-sdk-verifier-ts.md:85-109`）：

1. 读取关键文件：`package.json`、`tsconfig.json`、主源码、环境文件与配置（`:87-93`）。
2. 对照官方文档：使用 WebFetch 获取 TS SDK 参考并做偏差分析（`:95-99`）。
3. 执行类型检查：运行 `npx tsc --noEmit` 并报告编译问题（`:101-104`）。
4. 分析 SDK 用法：校验调用参数、模式与官方示例一致性（`:106-109`）。

与 `/new-sdk-app` 的联动关系：上游命令在创建项目后也要求执行 `npx tsc --noEmit`，形成“创建阶段 + 验收阶段”双重类型门禁（`plugins/agent-sdk-dev/commands/new-sdk-app.md:117-123,164-166`）。

### 数据结构与协议

1. Agent 注册结构（YAML frontmatter）
- `name`、`description`、`model` 三字段承载注册语义。
- 证据：`plugins/agent-sdk-dev/agents/agent-sdk-verifier-ts.md:1-5`。

2. 输出协议（报告数据模型）
- 状态枚举：`PASS | PASS WITH WARNINGS | FAIL`。
- 固定段落：`Summary`、`Critical Issues`、`Warnings`、`Passed Checks`、`Recommendations`。
- 证据：`plugins/agent-sdk-dev/agents/agent-sdk-verifier-ts.md:111-143`。

3. 约束协议（范围控制）
- 通过 `What NOT to Focus On` 明确不评审一般风格问题，保证审查资源聚焦 SDK 合规。
- 证据：`plugins/agent-sdk-dev/agents/agent-sdk-verifier-ts.md:78-83`。

### 关键命令与检查动作

该文件内显式硬约束命令：
- `npx tsc --noEmit`（`plugins/agent-sdk-dev/agents/agent-sdk-verifier-ts.md:39,103`）

与上游命令流程配套的关键命令：
- `npm install @anthropic-ai/claude-agent-sdk@latest`
- `npm list @anthropic-ai/claude-agent-sdk`
- `npx tsc --noEmit`
- 证据：`plugins/agent-sdk-dev/commands/new-sdk-app.md:83,86,119`。

这使 TS verifier 在“策略检查 + 机器可执行检查”两方面都具备较强确定性。

## 关键代码路径与文件引用

核心被研究文件：
- `plugins/agent-sdk-dev/agents/agent-sdk-verifier-ts.md:1-145`

直接调用方：
- `plugins/agent-sdk-dev/commands/new-sdk-app.md:128-135`

能力说明与用户承诺：
- `plugins/agent-sdk-dev/README.md:83-116`
- `plugins/agent-sdk-dev/README.md:103-107,133-135`

插件注册与发现：
- `plugins/agent-sdk-dev/.claude-plugin/plugin.json:1-9`
- `.claude-plugin/marketplace.json:12-16`
- `plugins/README.md:15`

同目录对照文件：
- `plugins/agent-sdk-dev/agents/agent-sdk-verifier-py.md:1-140`

测试与脚本上下文：
- `plugins/agent-sdk-dev` 当前没有独立测试和脚本文件，主要以命令/agent 协议驱动执行（目录文件清单仅 5 个）。

## 依赖与外部交互

内部依赖：
- 依赖 `/new-sdk-app` 的语言判定和触发约定（`plugins/agent-sdk-dev/commands/new-sdk-app.md:31-33,132-133`）。
- 与 README 的对外流程承诺耦合（`plugins/agent-sdk-dev/README.md:11-23,83-116`）。

外部依赖：
- 文档站：`https://docs.claude.com/en/api/agent-sdk/typescript`（WebFetch）（`plugins/agent-sdk-dev/agents/agent-sdk-verifier-ts.md:97`）。
- Node/npm/TypeScript 工具链：用于安装、版本核验和类型编译。

与目标工程的文件系统交互（被审查对象）：
- `package.json`、`tsconfig.json`
- `index.ts`/`src/*`
- `.env.example`、`.gitignore`
- 证据：`plugins/agent-sdk-dev/agents/agent-sdk-verifier-ts.md:87-93`。

## 风险、边界与改进建议

风险：
1. 协议依赖执行者一致性：虽然有 `tsc` 硬检查，但其他项仍是提示词语义约束，存在口径波动。
2. 网络依赖风险：最佳实践比对依赖 WebFetch，受网络策略影响（`plugins/agent-sdk-dev/agents/agent-sdk-verifier-ts.md:95-99`）。
3. 模型固化风险：`model: sonnet` 固定值可能与未来组织模型策略不一致（`plugins/agent-sdk-dev/agents/agent-sdk-verifier-ts.md:4`）。
4. 构建脚本检查仍偏声明式：要求检查 scripts，但未定义最小脚本基线和失败级别（`plugins/agent-sdk-dev/agents/agent-sdk-verifier-ts.md:44-48`）。

边界：
- 该文件不负责项目脚手架生成；创建行为在 `commands/new-sdk-app.md`。
- 不审查一般 TypeScript 风格偏好，只聚焦 SDK 合规和可运行性。

改进建议：
1. 增补“硬失败条件”矩阵
- 明确哪些问题直接 `FAIL`（如 `tsc` 失败、SDK 未安装、密钥泄漏），减少主观判定。
2. 增加机器可读结果输出
- 在文本报告外增加 JSON 结构，便于 CI 或后续自动化收敛问题。
3. 定义最小脚本基线
- 明确 `package.json` 至少包含哪些脚本（例如 `start`, `typecheck`），并给出标准示例。
4. 模型参数外置
- 支持通过插件配置覆盖 `model` 字段，提升长期维护灵活性。
