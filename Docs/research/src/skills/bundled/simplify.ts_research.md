# simplify.ts 研究文档

## 场景与职责

`simplify.ts` 实现了 `/simplify` 内置技能，用于在代码修改后启动一轮系统化的代码审查与清理。它指导模型并行启动三个专门的审查 agent（代码复用、代码质量、效率），聚合它们的发现后直接修复问题。该技能常被 `/batch` 的 worker instructions 引用作为每个工作单元完成后的收尾步骤。

## 功能点目的

1. **变更范围识别**：通过 `git diff`（或 `git diff HEAD`）获取当前修改，若无 git 变更则回退到最近修改的文件。
2. **三维度并行审查**：
   - **Code Reuse**：检查新代码是否重复了现有工具函数或模式。
   - **Code Quality**：检查冗余状态、参数膨胀、复制粘贴、泄露抽象、stringly-typed 代码、不必要的 JSX 嵌套、冗余注释。
   - **Efficiency**：检查不必要的工作、错失的并发、热路径膨胀、无条件的循环更新、不必要的存在性检查、内存泄漏、过度宽泛的操作。
3. **自动修复**：等待所有审查 agent 返回后，聚合发现并直接修改代码。

## 具体技术实现

### 关键流程

- `registerSimplifySkill()` → `registerBundledSkill({ name: 'simplify', ... })`
- `getPromptForCommand(args)` 入口：
  1. 返回固定的 `SIMPLIFY_PROMPT`
  2. 若用户提供了额外参数，追加 `## Additional Focus`

### Prompt 结构（SIMPLIFY_PROMPT）

1. **Phase 1: Identify Changes**
   - 运行 `git diff` 或 `git diff HEAD`
   - 无变更时审查最近修改的文件
2. **Phase 2: Launch Three Review Agents in Parallel**
   - 使用 `Agent` 工具一次性并发启动三个子 agent
   - 每个 agent 传递完整 diff 作为上下文
3. **Phase 3: Fix Issues**
   - 聚合发现，跳过 false positive
   - 简要总结修复内容

### 注册参数

| 字段 | 值 |
|------|-----|
| `name` | `'simplify'` |
| `userInvocable` | `true` |

## 关键代码路径与文件引用

- 源文件：`src/skills/bundled/simplify.ts`
- 注册入口：`src/skills/bundled/index.ts`
- 核心注册器：`src/skills/bundledSkills.ts`
- Agent 工具常量：`src/tools/AgentTool/constants.ts`（`AGENT_TOOL_NAME`）

## 依赖与外部交互

| 依赖 | 作用 |
|------|------|
| `AGENT_TOOL_NAME` | 在 prompt 中引用 Agent 工具以启动并行审查 |
| `registerBundledSkill` | 注册技能 |

- **无直接外部调用**：所有执行逻辑通过模型读取 prompt 后自行调用工具完成。
- **与 `/batch` 的集成**：`batch.ts` 的 `WORKER_INSTRUCTIONS` 将 `/simplify` 列为 worker 完成修改后的标准步骤之一。

## 风险、边界与改进建议

1. **边界：完全依赖模型自律**：`/simplify` 没有任何强制性的代码分析工具或 lint 集成，所有审查维度都是 prompt 中的文字描述，模型可能遗漏或执行不一致。
2. **边界：子 agent 成本不可控**：三个并行 agent 的调用会消耗额外的 API token，对于大型 diff 可能导致显著的成本上升。
3. **风险：修复可能引入新问题**：Phase 3 要求模型在聚合发现后直接修复，但没有要求再次运行测试或重新 diff 验证，存在"修坏"的风险。
4. **改进建议**：
   - 引入实际的静态分析工具（如 TypeScript compiler API、ESLint、Biome）作为审查 agent 的输入，减少纯 prompt 审查的主观性。
   - 在 Phase 3 之后增加一个强制性的 "验证步骤"：要求模型运行测试套件并再次执行 `git diff`，确认修复没有引入回归。
   - 为三个审查维度分别创建独立的内置 agent 定义（类似 `exploreAgent`/`planAgent`），使系统提示更精炼、行为更可预测。
