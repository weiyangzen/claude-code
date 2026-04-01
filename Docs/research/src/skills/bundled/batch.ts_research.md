# batch.ts 研究文档

## 场景与职责

`batch.ts` 实现了 `/batch` 这一内置技能（bundled skill），用于将大规模、可并行化的代码库变更分解为 5–30 个独立的并行工作单元，每个单元在隔离的 git worktree 中由后台 agent 执行，并最终各自提交 PR。该技能面向需要跨大量文件进行迁移、重构、批量重命名等"横扫式"机械变更的场景。

## 功能点目的

1. **计划模式引导**：强制模型先进入 Plan Mode（`EnterPlanMode`），通过子 agent 调研范围、分解任务、确定 e2e 测试方案，再退出计划模式等待用户批准。
2. **并行工作编排**：计划获批后，一次性批量启动多个后台 agent（`Agent` 工具），每个 agent 使用 `isolation: "worktree"` 与 `run_in_background: true`。
3. **进度跟踪**：渲染状态表格，解析每个 agent 返回结果中的 `PR: <url>` 行，动态更新状态与 PR 链接。
4. **前置校验**：非 git 仓库拒绝执行；空指令给出用法示例。

## 具体技术实现

### 关键流程

- `registerBatchSkill()` → `registerBundledSkill({ name: 'batch', ... })`
- `getPromptForCommand(args)` 异步入口：
  1. `args.trim()` 判空 → 返回 `MISSING_INSTRUCTION_MESSAGE`
  2. `getIsGit()` 判 git 仓库 → 非 git 返回 `NOT_A_GIT_REPO_MESSAGE`
  3. 否则返回 `buildPrompt(instruction)` 生成的大型系统提示文本

### 数据结构

```ts
const MIN_AGENTS = 5
const MAX_AGENTS = 30
```

- `WORKER_INSTRUCTIONS`：固定模板字符串，包含简化（`simplify` 技能）、运行单元测试、e2e 测试、提交并推送 PR、报告 `PR: <url>` 的规范。
- `buildPrompt(instruction: string): string`：返回完整的 Markdown 格式系统提示，分三阶段（Research & Plan / Spawn Workers / Track Progress）。

### 协议/命令

- 依赖工具常量：`AGENT_TOOL_NAME`、`ASK_USER_QUESTION_TOOL_NAME`、`ENTER_PLAN_MODE_TOOL_NAME`、`EXIT_PLAN_MODE_TOOL_NAME`、`SKILL_TOOL_NAME`
- 依赖 git 检测：`getIsGit()`（来自 `src/utils/git.js`，基于 `findGitRoot(getCwd())`）

## 关键代码路径与文件引用

- 源文件：`src/skills/bundled/batch.ts`
- 注册入口：`src/skills/bundled/index.ts` 中 `registerBatchSkill()` 被调用
- 核心注册器：`src/skills/bundledSkills.ts` 的 `registerBundledSkill()`
- 依赖的常量文件：
  - `src/tools/AgentTool/constants.js`
  - `src/tools/AskUserQuestionTool/prompt.js`
  - `src/tools/EnterPlanModeTool/constants.js`
  - `src/tools/ExitPlanModeTool/constants.js`
  - `src/tools/SkillTool/constants.js`
- 依赖的 git 工具：`src/utils/git.ts`（`getIsGit`）

## 依赖与外部交互

| 依赖 | 作用 |
|------|------|
| `registerBundledSkill` | 将 `/batch` 注册为可识别的命令 |
| `getIsGit` | 校验当前目录是否为 git 仓库 |
| `AGENT_TOOL_NAME` 等 | 在 prompt 中引用正确的工具名称，避免硬编码 |

- **无外部网络调用**：纯 prompt 生成器，所有执行逻辑由模型通过工具调用完成。
- **用户可显式调用**：`userInvocable: true`，`disableModelInvocation: true`（防止模型自动调用，必须由用户显式发起）。

## 风险、边界与改进建议

1. **边界：agent 数量范围**：`MIN_AGENTS=5`、`MAX_AGENTS=30`，但 prompt 仅给出文字约束，实际执行时由模型自行决定拆分数量，存在过度拆分或不足拆分的风险。
2. **边界：git worktree 资源消耗**：大量并发 worktree 可能显著占用磁盘与内存，尤其在单机上同时运行 30 个 agent 时。
3. **风险：PR 链接解析脆弱性**：要求 agent 最终消息必须包含 `PR: <url>` 单行，格式偏差会导致进度表格无法正确提取 PR 链接。
4. **改进建议**：
   - 增加对当前磁盘剩余空间的预检查，避免 worktree  explosion。
   - 考虑在 `buildPrompt` 中注入当前仓库的默认分支名、测试命令等上下文，减少每个子 agent 的重复探测。
   - 对 `PR: <url>` 的解析增加更宽松的正则匹配（如支持 `PR: none — reason` 的自动分类）。
