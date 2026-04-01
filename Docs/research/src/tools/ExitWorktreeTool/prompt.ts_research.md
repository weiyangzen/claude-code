# prompt.ts 研究文档

## 场景与职责

`prompt.ts` 为 `ExitWorktreeTool` 提供面向大语言模型的系统提示词（system prompt）。该提示词在工具被注册到模型上下文时注入，用于指导模型：
- 该工具的适用范围（仅当前会话中由 `EnterWorktree` 创建的 worktree）
- 何时应该调用该工具（仅在用户明确要求时）
- 各参数的含义与约束
- 工具执行后的副作用（恢复 CWD、清理缓存、tmux 行为等）

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| Scope 说明 | 明确限制工具只能操作当前会话的 `EnterWorktree` worktree，防止模型误操作手动创建的 worktree 或历史 worktree |
| When to Use | 防止模型 proactively 调用该工具，仅允许在用户明确请求时触发 |
| Parameters 说明 | 详细解释 `action`（keep/remove）和 `discard_changes` 的语义与交互逻辑 |
| Behavior 说明 | 告知模型退出后会恢复 CWD、清理缓存、处理 tmux 会话，并可再次调用 `EnterWorktree` |

## 具体技术实现

### 代码内容

```ts
export function getExitWorktreeToolPrompt(): string {
  return `Exit a worktree session created by EnterWorktree and return the session to the original working directory.

## Scope

This tool ONLY operates on worktrees created by EnterWorktree in this session. It will NOT touch:
- Worktrees you created manually with \`git worktree add\`
- Worktrees from a previous session (even if created by EnterWorktree then)
- The directory you're in if EnterWorktree was never called

If called outside an EnterWorktree session, the tool is a **no-op**: it reports that no worktree session is active and takes no action. Filesystem state is unchanged.

## When to Use

- The user explicitly asks to "exit the worktree", "leave the worktree", "go back", or otherwise end the worktree session
- Do NOT call this proactively — only when the user asks

## Parameters

- \`action\` (required): \`"keep"\` or \`"remove"\`
  - \`"keep"\` — leave the worktree directory and branch intact on disk. Use this if the user wants to come back to the work later, or if there are changes to preserve.
  - \`"remove"\` — delete the worktree directory and its branch. Use this for a clean exit when the work is done or abandoned.
- \`discard_changes\` (optional, default false): only meaningful with \`action: "remove"\`. If the worktree has uncommitted files or commits not on the original branch, the tool will REFUSE to remove it unless this is set to \`true\`. If the tool returns an error listing changes, confirm with the user before re-invoking with \`discard_changes: true\`.

## Behavior

- Restores the session's working directory to where it was before EnterWorktree
- Clears CWD-dependent caches (system prompt sections, memory files, plans directory) so the session state reflects the original directory
- If a tmux session was attached to the worktree: killed on \`remove\`, left running on \`keep\` (its name is returned so the user can reattach)
- Once exited, EnterWorktree can be called again to create a fresh worktree
`
}
```

### 结构分析

| 章节 | 内容要点 |
|------|----------|
| 标题句 | 一句话概括工具核心目的 |
| `## Scope` | 三段否定式列举 + no-op 说明，建立安全边界 |
| `## When to Use` | 正向触发词示例 + 显式禁止 proactive 调用 |
| `## Parameters` | `action` 的二元语义解释；`discard_changes` 的默认值、适用条件、错误重试流程 |
| `## Behavior` | 副作用清单，帮助模型理解调用后的系统状态变化 |

### 关键措辞设计

- **"ONLY operates on worktrees created by EnterWorktree in this session"**：大写 ONLY 强调排他性
- **"no-op"**：明确告知模型，即使误调用也不会产生破坏性副作用，降低模型的调用焦虑
- **"Do NOT call this proactively"**：直接指令，防止模型在任务完成时自动清理 worktree
- **"REFUSE to remove" / "confirm with the user"**：将权限确认责任明确交给模型，引导其在收到 error 后向用户确认

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/tools/ExitWorktreeTool/prompt.ts` | 本文件，导出 `getExitWorktreeToolPrompt` |
| `src/tools/ExitWorktreeTool/ExitWorktreeTool.ts` | 在 `buildTool` 的 `prompt()` 方法中调用 `getExitWorktreeToolPrompt()` |
| `src/Tool.ts` | 定义 `Tool.prompt` 接口，要求返回 `Promise<string>` |
| `src/tools/EnterWorktreeTool/prompt.ts` | 对称文件，提供进入 worktree 的模型提示词 |

## 依赖与外部交互

### 与系统提示组装流程的关系

在 `ExitWorktreeTool.ts` 中：

```ts
async prompt() {
  return getExitWorktreeToolPrompt()
}
```

- `buildTool` 将 `prompt` 方法注册到工具对象
- 在每次向模型发送请求前，`systemPrompt` 组装逻辑会遍历可用工具，调用各自的 `prompt()` 获取描述文本
- 该文本与其他工具提示词、系统指令、CLAUDE.md 内容拼接后送入模型

### 与模型行为的耦合

提示词的质量直接影响模型调用该工具的准确性：
- 如果 Scope 描述不够清晰，模型可能在用户未请求时主动调用 `ExitWorktree`
- 如果 `discard_changes` 解释不够详细，模型可能在首次收到拒绝错误后，未经用户确认就直接重试 `discard_changes: true`
- 当前文案通过"confirm with the user"明确将确认步骤纳入模型的推理链

## 风险、边界与改进建议

### 风险与边界

1. **提示词与实现逻辑的同步风险**
   - `prompt.ts` 中的描述是模型可见的"契约"，而 `ExitWorktreeTool.ts` 中的 `validateInput` 和 `call` 是实际执行逻辑。如果两者出现分歧（例如提示词说会清理某缓存而代码未清理），模型可能基于错误假设做出决策。
   - 当前两者基本一致，但未来若增加新的副作用（如清理 LSP 缓存），需要同步更新提示词。

2. **"no-op" 表述的精确性**
   - 提示词声称"If called outside an EnterWorktree session, the tool is a no-op"。严格来说，这取决于 `validateInput` 的实现：它会返回一个 `result: false` 的验证错误，而不是静默成功。对模型而言这确实是"无文件系统副作用"，但技术上并非完全无操作（仍有验证逻辑执行）。这种表述在 UX 层面是合理的，但在技术文档中需要区分。

3. **缺少参数示例（JSON 样例）**
   - 提示词中没有给出具体的工具调用 JSON 示例。对于复杂参数交互（如 `discard_changes` 与 `action` 的联动），一个简短示例可以降低模型出错率。

4. **国际化与可维护性**
   - 提示词为硬编码英文。虽然模型对英文提示词理解最好，但如果未来需要支持多语言 CLI 界面，当前的纯字符串模板难以扩展。

### 改进建议

1. **增加参数调用示例**
   在 `## Parameters` 或新增 `## Example` 章节中加入 JSON 示例：
   ```markdown
   ## Example
   To safely remove a clean worktree:
   ```json
   { "action": "remove" }
   ```
   To remove a worktree with uncommitted changes after user confirmation:
   ```json
   { "action": "remove", "discard_changes": true }
   ```
   ```
   这可以帮助模型更准确地构造参数，尤其是处理可选布尔字段时。

2. **提取可复用的行为描述**
   - `restoreSessionToOriginalCwd` 中清理的缓存列表（system prompt sections、memory files、plans directory）与提示词中的描述完全一致。可以考虑在代码中定义一个常量数组，并在提示词中通过模板字符串动态引用，确保未来增减缓存时不会遗漏文档更新。

3. **考虑增加错误码说明**
   - `validateInput` 实际返回了 `errorCode: 1/2/3`，但提示词中仅模糊提到"returns an error listing changes"。若补充：
     - `errorCode 1` = 无 active session
     - `errorCode 2` = 存在未保存变更，需确认
     - `errorCode 3` = 无法验证状态，需显式确认
     模型在遇到不同错误时可以采取更精准的重试策略。

4. **版本化提示词（长期）**
   - 如果 worktree 功能持续迭代，可以考虑为提示词增加版本标识（如 `v2`），便于 A/B 测试不同提示词对模型调用准确率的影响。
