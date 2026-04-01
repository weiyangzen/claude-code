# skillify.ts 研究文档

## 场景与职责

`skillify.ts` 实现了 `/skillify` 内置技能，用于将当前会话中完成的可重复工作流程捕获为一个可复用的 SKILL.md 文件。该技能仅对 Anthropic 内部员工（`USER_TYPE === 'ant'`）可用，通过分析会话记忆摘要和用户消息，引导用户完成多轮确认 interview，最终生成并保存技能文档。

## 功能点目的

1. **会话分析**：读取当前会话的 session memory 摘要和用户消息历史，识别可重复流程、输入参数、步骤、成功标准、用户纠正点。
2. **结构化 Interview**：通过 `AskUserQuestion` 工具分轮次与用户确认：
   - Round 1：技能名称、目标、成功标准
   - Round 2：步骤、参数、执行方式（inline vs forked）、保存位置（repo vs personal）
   - Round 3：每个步骤的产出、成功标准、人工检查点、并行性
   - Round 4：触发时机、触发短语
3. **生成 SKILL.md**：按照标准 frontmatter + Markdown 格式创建技能文件。
4. **确认后保存**：先向用户展示完整的 SKILL.md 内容（YAML 代码块高亮），经用户确认后再写入文件。

## 具体技术实现

### 关键流程

- `registerSkillifySkill()`：
  1. 若 `process.env.USER_TYPE !== 'ant'` 直接返回
  2. 否则 `registerBundledSkill({ name: 'skillify', ... })`
- `getPromptForCommand(args, context)` 入口：
  1. `getSessionMemoryContent()` 异步读取 session memory
  2. `getMessagesAfterCompactBoundary(context.messages)` 获取压缩边界后的消息
  3. `extractUserMessages(messages)` 过滤并拼接所有用户文本消息
  4. 用上述内容替换 `SKILLIFY_PROMPT` 中的 `{{sessionMemory}}`、`{{userMessages}}`、`{{userDescriptionBlock}}` 占位符
  5. 返回组装后的 prompt

### 数据结构

```ts
function extractUserMessages(messages: Message[]): string[] {
  return messages
    .filter((m): m is Extract<typeof m, { type: 'user' }> => m.type === 'user')
    .map(m => {
      const content = m.message.content
      if (typeof content === 'string') return content
      return content
        .filter((b): b is Extract<typeof b, { type: 'text' }> => b.type === 'text')
        .map(b => b.text)
        .join('\n')
    })
    .filter(text => text.trim().length > 0)
}
```

### 注册参数

| 字段 | 值 |
|------|-----|
| `name` | `'skillify'` |
| `allowedTools` | `['Read','Write','Edit','Glob','Grep','AskUserQuestion','Bash(mkdir:*)']` |
| `userInvocable` | `true` |
| `disableModelInvocation` | `true`（必须由用户显式调用） |
| `argumentHint` | `'[description of the process you want to capture]'` |

## 关键代码路径与文件引用

- 源文件：`src/skills/bundled/skillify.ts`
- 注册入口：`src/skills/bundled/index.ts`
- 核心注册器：`src/skills/bundledSkills.ts`
- Session Memory 工具：`src/services/SessionMemory/sessionMemoryUtils.ts`（`getSessionMemoryContent`）
- 消息工具：`src/utils/messages.ts`（`getMessagesAfterCompactBoundary`）
- 消息类型：`src/types/message.ts`（`Message` 类型）

## 依赖与外部交互

| 依赖 | 作用 |
|------|------|
| `getSessionMemoryContent` | 读取当前会话的 memory 摘要文件 |
| `getMessagesAfterCompactBoundary` | 获取 compact 后保留的消息列表 |
| `registerBundledSkill` | 注册技能 |

- **无网络调用**：纯本地 prompt 组装与文件操作。
- **Session Memory 依赖**：若 session memory 功能被禁用或文件不存在，`getSessionMemoryContent` 返回 `null`，prompt 中会显示 "No session memory available."。

## 风险、边界与改进建议

1. **边界：Ant-only 限制**：与 `lorem-ipsum`、`remember`、`stuck` 一样，`skillify` 仅限内部员工使用，外部用户无法将自己常用的工作流程保存为技能。
2. **边界：消息提取仅包含文本块**：`extractUserMessages` 只提取 `type === 'text'` 的内容块，若用户发送了图片、工具结果等作为上下文，这些内容不会被纳入 skillify 分析。
3. **风险：Interview 轮次可能过度冗长**：Prompt 要求分 4 轮以上使用 `AskUserQuestion`，对于简单流程（如 2 步操作）可能过度询问，导致用户体验下降。
4. **风险：保存位置的安全性**：模型被引导保存到 `.claude/skills/<name>/SKILL.md` 或 `~/.claude/skills/<name>/SKILL.md`，但 prompt 中未对 `<name>` 做路径遍历校验，理论上可能生成包含 `..` 的路径（虽然 `Write`/`Edit` 工具本身有路径校验）。
5. **改进建议**：
   - 放宽 `USER_TYPE !== 'ant'` 限制，让普通用户也能使用 skillify（这是产品层面的决策）。
   - 在 `extractUserMessages` 中增加对 tool_result 和 image 块的引用摘要，帮助模型理解用户如何通过工具交互塑造流程。
   - 增加一个 "简单模式" 路径：若模型判断流程非常简单，可跳过部分 interview 轮次直接生成 SKILL.md。
   - 在生成保存路径时，显式对技能名进行 sanitize（如替换非法文件系统字符），避免路径问题。
