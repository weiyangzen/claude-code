# conversationRecovery.ts 深度研究文档

## 场景与职责

`conversationRecovery.ts` 是 Claude Code 的**会话恢复核心模块**，负责从各种来源加载、反序列化和准备历史会话数据，以便在 `--resume` 或 `--continue` 时恢复对话状态。它是连接持久化存储（磁盘上的 JSONL 文件）和运行时内存模型的关键桥梁。

### 核心职责

1. **会话加载**：从会话 ID、日志文件路径或直接提供 LogOption 加载历史对话
2. **消息反序列化**：将磁盘上的消息格式转换为运行时 Message 格式
3. **中断检测**：检测会话是否在中途被中断（如用户取消、进程崩溃）
4. **数据迁移**：处理旧版本附件类型的向后兼容
5. **技能状态恢复**：从消息中恢复被调用的技能状态
6. **跨项目恢复支持**：支持从不同项目目录恢复会话

### 使用场景

- **REPL 模式**：用户运行 `--resume <session-id>` 恢复之前的对话
- **Teleport 功能**：跨设备/跨目录迁移会话
- **SDK 模式**：程序化恢复会话状态
- **自动继续**：检测到中断后自动注入 "Continue from where you left off" 消息

---

## 功能点目的

### 1. 消息反序列化 (`deserializeMessages` / `deserializeMessagesWithInterruptDetection`)

**目的**：将持久化的消息列表转换为有效的运行时格式，同时清理无效数据。

**处理流程**：
1. **遗留附件迁移**：将 `new_file` → `file`，`new_directory` → `directory`
2. **权限模式验证**：过滤掉无效的 `permissionMode` 值
3. **工具使用过滤**：移除未解决的 tool_use 消息及其后续合成消息
4. **孤立思考消息过滤**：移除可能导致 API 错误的孤立思考消息
5. **空白消息过滤**：移除仅包含空白字符的助手消息
6. **中断检测**：判断会话是否在中途被中断
7. **合成消息注入**：在最后一个用户消息后添加合成助手标记

### 2. 中断检测 (`detectTurnInterruption`)

**目的**：确定会话是否在用户回合或助手回合中被中断，以便正确处理恢复。

**检测逻辑**：
- **无中断**：最后相关消息是助手消息（turn 已完成）
- **中断提示**：最后消息是普通用户文本（助手尚未开始响应）
- **中断回合**：最后消息是工具结果或附件（助手响应中途被打断）

**特殊处理**：
- Brief 模式下的 `SendUserMessage` 工具结果被视为正常终止
- 系统消息和进度消息在检测时被跳过
- API 错误消息被忽略，允许在重试耗尽后自动恢复

### 3. 技能状态恢复 (`restoreSkillStateFromMessages`)

**目的**：在会话恢复后重新填充 `STATE.invokedSkills`，确保技能在压缩后不会丢失。

**工作原理**：
- 扫描消息中的 `invoked_skills` 类型附件
- 调用 `addInvokedSkill` 将技能重新注册到全局状态
- 检测 `skill_listing` 附件以避免重复发送技能列表提醒

### 4. 跨目录会话加载 (`loadMessagesFromJsonlPath`)

**目的**：支持从任意路径的 JSONL 文件加载会话（用于 Teleport 和跨项目恢复）。

**实现细节**：
- 使用 `loadTranscriptFile` 加载并按 UUID 索引消息
- 识别叶子节点（没有其他消息指向的 UUID）
- 选择最新的非侧链叶子作为会话终点
- 重建完整的对话链

### 5. 统一会话加载入口 (`loadConversationForResume`)

**目的**：提供单一的会话加载接口，处理所有恢复场景。

**支持的源类型**：
- `undefined`：加载最近的会话（`--continue`）
- `string`：按会话 ID 加载
- `LogOption`：直接使用已加载的日志
- `sourceJsonlFile`：从指定 JSONL 路径加载

---

## 具体技术实现

### 关键数据类型

```typescript
// 反序列化结果
type DeserializeResult = {
  messages: Message[]
  turnInterruptionState: TurnInterruptionState
}

// 中断状态
 type TurnInterruptionState =
  | { kind: 'none' }                           // 无中断
  | { kind: 'interrupted_prompt'; message: NormalizedUserMessage }  // 用户提示中断
  | { kind: 'interrupted_turn' }               // 回合中断（内部使用）

// Teleport 远程响应
type TeleportRemoteResponse = {
  log: Message[]
  branch?: string
}
```

### 关键流程：反序列化

```
serializedMessages
    ↓
migrateLegacyAttachmentTypes()  // 向后兼容处理
    ↓
permissionMode 验证              // 过滤无效权限模式
    ↓
filterUnresolvedToolUses()      // 移除未匹配的工具使用
    ↓
filterOrphanedThinkingOnlyMessages()  // 移除孤立思考
    ↓
filterWhitespaceOnlyAssistantMessages()  // 移除空白消息
    ↓
detectTurnInterruption()        // 检测中断状态
    ↓
[如果是 interrupted_turn] → 注入合成继续消息
    ↓
[如果最后是用户消息] → 注入合成助手标记
    ↓
{ messages, turnInterruptionState }
```

### 关键流程：终端工具结果检测

```typescript
function isTerminalToolResult(result, messages, resultIdx): boolean {
  // 1. 提取 tool_use_id
  // 2. 向前查找对应的 tool_use
  // 3. 检查工具名是否为 BriefTool 或 SendUserFileTool
  // 4. 这些工具在 Brief 模式下是回合的合法终点
}
```

### 遗留附件迁移

| 旧类型 | 新类型 | 处理 |
|--------|--------|------|
| `new_file` | `file` | 添加 `displayPath` |
| `new_directory` | `directory` | 添加 `displayPath` |
| 无 `displayPath` | - | 从 `filename`/`path`/`skillDir` 回填 |

---

## 关键代码路径与文件引用

### 核心导出函数

| 函数 | 行号 | 用途 |
|------|------|------|
| `loadConversationForResume` | 456-597 | 主入口：加载会话用于恢复 |
| `deserializeMessages` | 154-156 | 简单反序列化（仅消息） |
| `deserializeMessagesWithInterruptDetection` | 164-252 | 完整反序列化（含中断检测） |
| `restoreSkillStateFromMessages` | 382-403 | 从消息恢复技能状态 |
| `loadMessagesFromJsonlPath` | 416-440 | 从 JSONL 路径加载 |

### 依赖文件

```
conversationRecovery.ts
├── 被调用方（上游）
│   ├── src/screens/ResumeConversation.tsx    # 恢复界面
│   ├── src/cli/print.ts                       # CLI 恢复
│   ├── src/utils/teleport.tsx                 # Teleport 功能
│   └── src/hooks/useTeleportResume.tsx        # Teleport Hook
├── 被依赖模块（下游）
│   ├── src/utils/sessionStorage.ts            # 会话存储操作
│   ├── src/utils/messages.ts                  # 消息工具函数
│   ├── src/utils/fileHistory.ts               # 文件历史恢复
│   ├── src/utils/plans.ts                     # 计划恢复
│   ├── src/utils/sessionStart.ts              # 会话启动钩子
│   └── src/bootstrap/state.ts                 # 全局状态
└── 类型定义
    ├── src/types/logs.ts                      # LogOption 等
    ├── src/types/message.ts                   # Message 类型
    └── src/types/permissions.ts               # PERMISSION_MODES
```

### 条件导入（特性标志）

```typescript
// 仅在 KAIROS 或 KAIROS_BRIEF 特性启用时加载
const BRIEF_TOOL_NAME = feature('KAIROS') || feature('KAIROS_BRIEF')
  ? require('../tools/BriefTool/prompt.js').BRIEF_TOOL_NAME
  : null
```

---

## 依赖与外部交互

### 运行时依赖

| 模块 | 用途 |
|------|------|
| `bun:bundle` | 特性标志检查 (`feature()`) |
| `crypto` | UUID 类型 |
| `path` | 路径处理 (`relative`) |

### 内部模块依赖

| 模块 | 导入内容 | 用途 |
|------|----------|------|
| `src/utils/cwd.js` | `getCwd` | 计算相对路径 |
| `src/bootstrap/state.js` | `addInvokedSkill` | 技能状态恢复 |
| `src/types/ids.js` | `asSessionId` | 会话 ID 类型转换 |
| `src/types/logs.js` | 多种日志类型 | 类型定义 |
| `src/types/message.js` | Message 类型 | 类型定义 |
| `src/types/permissions.js` | `PERMISSION_MODES` | 权限验证 |
| `src/utils/attachments.js` | `suppressNextSkillListing` | 技能列表控制 |
| `src/utils/fileHistory.js` | `copyFileHistoryForResume` | 文件历史恢复 |
| `src/utils/messages.js` | 多种消息工具 | 消息处理 |
| `src/utils/plans.js` | `copyPlanForResume` | 计划恢复 |
| `src/utils/sessionStart.js` | `processSessionStartHooks` | 启动钩子 |
| `src/utils/sessionStorage.js` | 多种存储工具 | 会话存储操作 |
| `src/utils/toolResultStorage.js` | `ContentReplacementRecord` | 类型定义 |

### 动态导入

```typescript
// UDS 客户端仅在 BG_SESSIONS 特性启用时加载
const { listAllLiveSessions } = await import('./udsClient.js')
```

---

## 风险、边界与改进建议

### 已知风险

1. **循环依赖风险**
   - `conversationRecovery.ts` → `messages.ts` → `debug.ts` → ...
   - 已通过条件导入和工具函数参数化缓解

2. **向后兼容复杂性**
   - 遗留附件类型迁移逻辑需要持续维护
   - 新附件类型添加时需要更新迁移逻辑

3. **中断检测误判**
   - Brief 模式下的工具结果检测依赖硬编码工具名
   - 如果工具名变更，可能导致误判

4. **性能问题**
   - `loadMessagesFromJsonlPath` 需要扫描整个 JSONL 文件
   - 大型会话文件可能导致加载延迟

### 边界情况

| 场景 | 处理 |
|------|------|
| 空消息列表 | 返回 `kind: 'none'` |
| 所有消息被过滤后为空 | 返回 `kind: 'none'` |
| 无效 cron 字符串 | 返回 `null` |
| 损坏的 JSONL | 依赖 `loadTranscriptFile` 的错误处理 |
| 并发恢复 | 依赖调用方的互斥控制 |

### 改进建议

1. **迁移逻辑优化**
   - 考虑使用版本号标记会话格式，而非特征检测
   - 将迁移逻辑提取到独立的 `migrations.ts` 模块

2. **性能优化**
   - 为 `loadMessagesFromJsonlPath` 添加分页或流式加载支持
   - 缓存已解析的会话链，避免重复解析

3. **测试覆盖**
   - 添加更多遗留格式消息的单元测试
   - 测试各种中断场景的恢复行为

4. **类型安全**
   - 减少 `as SerializedMessage` 类型断言的使用
   - 使用更严格的类型守卫替代运行时类型检查

5. **文档完善**
   - 添加更多关于中断检测算法的注释
   - 记录遗留附件类型的历史背景

### 相关 Issue/PR 参考

- `#20467`: Brief 模式下的工具结果处理
- `#14373`, `#23537`: 进度消息导致的链分叉问题
- `CC-34`: `switchSession` 的原子性保证
