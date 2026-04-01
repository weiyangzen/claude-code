# MagicDocs Service 研究文档

## 场景与职责

MagicDocs 是 Claude Code 中的一项自动化文档维护服务。它通过检测特殊的文档标记（`# MAGIC DOC: [title]`），在后台自动更新 Markdown 文档，将从对话中获得的新知识、洞察和信息整合到这些文档中。

**核心职责：**
1. 自动检测带有 Magic Doc 标记的 Markdown 文件
2. 在 REPL 空闲时（无工具调用时）触发文档更新
3. 使用 forked subagent 在后台执行文档更新
4. 维护一个追踪列表，记录所有需要自动更新的 Magic Doc 文件

**使用场景：**
- 开发者希望在项目中维护自动更新的架构文档
- 需要记录代码库的关键设计决策和模式
- 希望文档能随着对话深入自动累积知识

---

## 功能点目的

### 1. Magic Doc 头部检测

**功能：** 识别文件是否为 Magic Doc

**检测规则：**
- 头部格式：`# MAGIC DOC: [title]`（不区分大小写）
- 可选的自定义指令：标题后的斜体行（`*instructions*` 或 `_instructions_`）

**代码位置：** `detectMagicDocHeader()` 函数（line 52-81）

```typescript
const MAGIC_DOC_HEADER_PATTERN = /^#\s*MAGIC\s+DOC:\s*(.+)$/im
const ITALICS_PATTERN = /^[_*](.+?)[_*]\s*$/m
```

### 2. Magic Doc 注册与追踪

**功能：** 维护一个全局的 Magic Doc 追踪列表

**数据结构：**
```typescript
type MagicDocInfo = {
  path: string
}

const trackedMagicDocs = new Map<string, MagicDocInfo>()
```

**关键函数：**
- `registerMagicDoc(filePath)` - 注册新的 Magic Doc（line 87-94）
- `clearTrackedMagicDocs()` - 清空追踪列表（line 44-46）

### 3. 文档自动更新

**功能：** 在 REPL 空闲时自动更新所有追踪的 Magic Doc

**触发条件：**
1. `querySource === 'repl_main_thread'` - 仅在主线程
2. `!hasToolCallsInLastAssistantTurn(messages)` - 最后一轮无工具调用
3. `trackedMagicDocs.size > 0` - 有待更新的文档

**更新流程：**
1. 克隆 FileStateCache 以隔离 Magic Docs 操作
2. 重新读取文档内容（绕过缓存）
3. 重新检测标题和指令
4. 构建更新 prompt
5. 使用 `runAgent()` 启动 magic-docs agent 执行更新

### 4. 权限控制

**功能：** 限制 Magic Docs agent 只能编辑目标文档

**实现：** 自定义 `canUseTool` 回调（line 172-192）
- 仅允许 `Edit` 工具
- 仅允许编辑当前正在更新的 Magic Doc 文件

---

## 具体技术实现

### 关键流程

#### 初始化流程

```typescript
export async function initMagicDocs(): Promise<void> {
  if (process.env.USER_TYPE === 'ant') {
    // 1. 注册文件读取监听器
    registerFileReadListener((filePath: string, content: string) => {
      const result = detectMagicDocHeader(content)
      if (result) {
        registerMagicDoc(filePath)
      }
    })
    
    // 2. 注册 post-sampling hook
    registerPostSamplingHook(updateMagicDocs)
  }
}
```

**调用位置：** `src/utils/backgroundHousekeeping.ts:32`

#### 更新流程

```typescript
const updateMagicDocs = sequential(async function (context: REPLHookContext): Promise<void> {
  // 1. 检查触发条件
  if (querySource !== 'repl_main_thread') return
  if (hasToolCallsInLastAssistantTurn(messages)) return
  if (trackedMagicDocs.size === 0) return
  
  // 2. 顺序更新每个文档
  for (const docInfo of Array.from(trackedMagicDocs.values())) {
    await updateMagicDoc(docInfo, context)
  }
})
```

**注意：** 使用 `sequential` 包装器确保并发调用按顺序执行，避免竞态条件。

#### 单文档更新流程

```typescript
async function updateMagicDoc(docInfo: MagicDocInfo, context: REPLHookContext): Promise<void> {
  // 1. 克隆缓存并删除当前文档条目
  const clonedReadFileState = cloneFileStateCache(toolUseContext.readFileState)
  clonedReadFileState.delete(docInfo.path)
  
  // 2. 读取当前文档内容
  const result = await FileReadTool.call({ file_path: docInfo.path }, clonedToolUseContext)
  
  // 3. 重新检测头部
  const detected = detectMagicDocHeader(currentDoc)
  if (!detected) {
    trackedMagicDocs.delete(docInfo.path)  // 不再有效，移除追踪
    return
  }
  
  // 4. 构建更新 prompt
  const userPrompt = await buildMagicDocsUpdatePrompt(currentDoc, docInfo.path, detected.title, detected.instructions)
  
  // 5. 运行 agent 执行更新
  for await (const _message of runAgent({
    agentDefinition: getMagicDocsAgent(),
    promptMessages: [createUserMessage({ content: userPrompt })],
    toolUseContext: clonedToolUseContext,
    canUseTool,  // 自定义权限控制
    isAsync: true,
    forkContextMessages: messages,  // 共享对话上下文
    querySource: 'magic_docs',
    override: { systemPrompt, userContext, systemContext },
    availableTools: clonedToolUseContext.options.tools,
  })) {
    // 消费消息直到完成
  }
}
```

### 数据结构

#### MagicDocInfo
```typescript
type MagicDocInfo = {
  path: string  // 文档的绝对路径
}
```

#### 内置 Agent 定义
```typescript
function getMagicDocsAgent(): BuiltInAgentDefinition {
  return {
    agentType: 'magic-docs',
    whenToUse: 'Update Magic Docs',
    tools: [FILE_EDIT_TOOL_NAME],  // 仅允许 Edit 工具
    model: 'sonnet',
    source: 'built-in',
    baseDir: 'built-in',
    getSystemPrompt: () => '',  // 使用 override systemPrompt
  }
}
```

### 协议与约定

1. **头部格式约定：**
   - 必须位于文件第一行
   - 格式：`# MAGIC DOC: Title`
   - 可选指令行：标题后的斜体行

2. **更新时机约定：**
   - 仅在主线程 REPL 空闲时触发
   - 使用 `sequential` 确保顺序执行

3. **权限控制约定：**
   - 仅允许使用 Edit 工具
   - 仅允许编辑当前目标文档

---

## 关键代码路径与文件引用

### 核心文件

| 文件路径 | 职责 |
|---------|------|
| `src/services/MagicDocs/magicDocs.ts` | MagicDocs 核心实现 |
| `src/services/MagicDocs/prompts.ts` | 更新 prompt 模板构建 |

### 依赖文件

| 文件路径 | 用途 |
|---------|------|
| `src/tools/FileReadTool/FileReadTool.ts` | 读取文档内容，`registerFileReadListener` |
| `src/tools/FileEditTool/constants.ts` | `FILE_EDIT_TOOL_NAME` 常量 |
| `src/tools/AgentTool/runAgent.ts` | `runAgent` 函数，执行子 agent |
| `src/tools/AgentTool/loadAgentsDir.ts` | `BuiltInAgentDefinition` 类型 |
| `src/utils/hooks/postSamplingHooks.ts` | `registerPostSamplingHook`, `REPLHookContext` |
| `src/utils/sequential.ts` | `sequential` 包装器 |
| `src/utils/fileStateCache.ts` | `cloneFileStateCache` |
| `src/utils/messages.ts` | `createUserMessage`, `hasToolCallsInLastAssistantTurn` |

### 调用方文件

| 文件路径 | 调用方式 |
|---------|---------|
| `src/utils/backgroundHousekeeping.ts:32` | `void initMagicDocs()` - 启动时初始化 |
| `src/commands/clear/caches.ts:125` | `clearTrackedMagicDocs()` - 清除缓存时调用 |

### 关键代码行号

```
magicDocs.ts
├── 31-35:    正则表达式定义
├── 38-42:    MagicDocInfo 类型定义
├── 44-46:    clearTrackedMagicDocs 函数
├── 52-81:    detectMagicDocHeader 函数
├── 87-94:    registerMagicDoc 函数
├── 99-109:   getMagicDocsAgent 函数
├── 114-212:  updateMagicDoc 函数
├── 217-240:  updateMagicDocs hook 函数
└── 242-254:  initMagicDocs 函数
```

---

## 依赖与外部交互

### 外部依赖

1. **FileReadTool**
   - `registerFileReadListener` - 监听文件读取事件
   - `FileReadTool.call()` - 读取文档内容
   - 输出类型：`FileReadToolOutput`

2. **Agent 系统**
   - `runAgent()` - 执行 Magic Docs 更新 agent
   - `BuiltInAgentDefinition` - Agent 定义类型

3. **Hook 系统**
   - `registerPostSamplingHook` - 注册 post-sampling hook
   - `REPLHookContext` - Hook 上下文

4. **工具系统**
   - `ToolUseContext` - 工具使用上下文
   - `Tool` - 工具类型

5. **缓存系统**
   - `cloneFileStateCache` - 克隆文件状态缓存
   - `FileStateCache` - 文件状态缓存类型

6. **消息系统**
   - `createUserMessage` - 创建用户消息
   - `hasToolCallsInLastAssistantTurn` - 检查工具调用

### 环境变量

- `USER_TYPE === 'ant'` - 仅在内部构建中启用 Magic Docs

### 常量

- `FILE_EDIT_TOOL_NAME = 'Edit'` - 允许使用的工具名

---

## 风险、边界与改进建议

### 潜在风险

1. **竞态条件风险**
   - 使用 `sequential` 包装器缓解，但如果多个文档同时更新仍可能产生冲突
   - 建议：考虑添加文档级别的锁机制

2. **无限循环风险**
   - 如果 Magic Doc 更新触发新的文件读取，可能导致循环
   - 缓解：使用克隆的 `readFileState` 并删除当前文档条目

3. **权限绕过风险**
   - `canUseTool` 回调仅检查 `file_path` 字段
   - 如果 agent 使用其他方式指定路径可能绕过限制

4. **内容丢失风险**
   - 文档更新是"当前状态"导向的，会删除过时信息
   - 如果更新逻辑有误，可能丢失重要历史信息

### 边界情况

1. **文件删除/不可访问**
   - 处理：`isFsInaccessible()` 检查 + 错误消息匹配
   - 行为：从追踪列表中删除

2. **Magic Doc 头部被移除**
   - 检测：`detectMagicDocHeader()` 返回 null
   - 行为：从追踪列表中删除

3. **空追踪列表**
   - 检查：`docCount === 0`
   - 行为：直接返回，不执行任何操作

4. **非主线程调用**
   - 检查：`querySource !== 'repl_main_thread'`
   - 行为：直接返回，不更新文档

5. **存在待处理工具调用**
   - 检查：`hasToolCallsInLastAssistantTurn(messages)`
   - 行为：延迟更新，等待对话空闲

### 改进建议

1. **可观测性增强**
   - 添加 analytics 事件追踪文档更新频率、成功率
   - 添加日志记录更新前后的内容变化统计

2. **错误恢复机制**
   - 当前错误仅通过 `logError` 记录
   - 建议：添加重试机制和失败通知

3. **批量更新优化**
   - 当前是顺序更新，如果文档较多可能耗时较长
   - 建议：考虑并行更新无依赖关系的文档

4. **自定义更新频率**
   - 当前每次空闲都尝试更新
   - 建议：支持配置最小更新间隔或仅在特定条件下触发

5. **更新预览功能**
   - 当前直接应用更新
   - 建议：支持生成 diff 预览，让用户确认后再应用

6. **文档健康检查**
   - 定期验证 Magic Doc 的完整性
   - 检测头部格式是否仍然有效

7. **用户控制增强**
   - 添加 `/magic-docs` 命令手动控制
   - 支持暂停/恢复特定文档的自动更新
