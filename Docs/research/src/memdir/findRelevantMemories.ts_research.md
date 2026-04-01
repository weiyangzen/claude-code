# findRelevantMemories.ts 研究文档

## 场景与职责

`findRelevantMemories.ts` 实现了一个**智能记忆召回系统**，用于根据用户查询从记忆目录中筛选最相关的记忆文件。这是记忆系统的核心组件之一，在查询时动态选择应该注入到对话上下文中的记忆。

### 核心职责
1. **记忆相关性筛选**：通过扫描记忆文件头部信息，使用 Sonnet 模型智能选择最相关的记忆（最多5个）
2. **避免重复**：支持 `alreadySurfaced` 参数过滤已展示过的记忆
3. **新鲜度感知**：返回记忆的修改时间（mtime），供调用方展示记忆新鲜度
4. **工具使用感知**：排除最近使用工具的参考文档（避免噪音），但保留警告/已知问题类记忆

### 使用场景
- 主对话循环中根据用户查询动态召回相关记忆
- 与 `extractMemories.ts` 配合，后者使用 `scanMemoryFiles` 扫描但不进行相关性筛选

---

## 功能点目的

### 1. `findRelevantMemories()` - 主入口函数
**目的**：根据查询召回相关记忆文件路径

**关键参数**：
- `query`: 用户查询文本
- `memoryDir`: 记忆目录路径
- `signal`: 中止信号
- `recentTools`: 最近使用的工具列表（避免重复推荐）
- `alreadySurfaced`: 已展示过的记忆路径集合

**返回值**：`RelevantMemory[]` - 包含路径和修改时间的记忆列表

### 2. `selectRelevantMemories()` - 内部选择函数
**目的**：调用 Sonnet 模型进行智能选择

**实现机制**：
- 使用 `sideQuery` 进行轻量级 API 调用
- 构造包含查询、可用记忆清单、最近使用工具的提示
- 要求模型返回 JSON 格式的 `selected_memories` 数组
- 过滤无效文件名，确保返回结果安全

### 3. 遥测支持
**目的**：在 `MEMORY_SHAPE_TELEMETRY` feature flag 开启时，记录记忆召回的形状数据

---

## 具体技术实现

### 关键流程

```
findRelevantMemories(query, memoryDir, signal, recentTools, alreadySurfaced)
  ↓
scanMemoryFiles(memoryDir, signal)  // 扫描所有记忆文件头部
  ↓
过滤已展示的记忆 (alreadySurfaced)
  ↓
selectRelevantMemories(query, memories, signal, recentTools)
  ↓
构造提示词 → sideQuery(Sonnet模型) → 解析JSON响应
  ↓
验证并返回 RelevantMemory[]
```

### 提示词工程 (`SELECT_MEMORIES_SYSTEM_PROMPT`)

```typescript
const SELECT_MEMORIES_SYSTEM_PROMPT = `You are selecting memories that will be useful to Claude Code as it processes a user's query...

Return a list of filenames for the memories that will clearly be useful to Claude Code as it processes the user's query (up to 5).
```

**关键约束**：
- 最多选择5个记忆
- 不确定的记忆不要包含（保守策略）
- 排除最近使用工具的 API 文档（`recentTools` 参数）
- 保留工具的警告/已知问题类记忆

### JSON Schema 输出格式

```typescript
{
  type: 'object',
  properties: {
    selected_memories: { type: 'array', items: { type: 'string' } }
  },
  required: ['selected_memories'],
  additionalProperties: false
}
```

### 数据结构

```typescript
export type RelevantMemory = {
  path: string    // 绝对文件路径
  mtimeMs: number // 修改时间戳（毫秒）
}
```

---

## 关键代码路径与文件引用

### 内部依赖
| 文件 | 用途 |
|------|------|
| `memoryScan.ts` | `scanMemoryFiles()` - 扫描记忆目录，`formatMemoryManifest()` - 格式化记忆清单 |
| `memoryTypes.ts` | `MemoryHeader` 类型定义 |

### 外部依赖
| 文件 | 用途 |
|------|------|
| `../utils/sideQuery.ts` | 轻量级 API 查询封装 |
| `../utils/model/model.ts` | `getDefaultSonnetModel()` - 获取默认 Sonnet 模型 |
| `../utils/slowOperations.ts` | `jsonParse()` - 安全 JSON 解析 |
| `../utils/debug.ts` | `logForDebugging()` - 调试日志 |
| `../utils/errors.ts` | `errorMessage()` - 错误信息提取 |
| `bun:bundle` | `feature()` - feature flag 检查 |

### 调用方
| 文件 | 用途 |
|------|------|
| `src/utils/messages.ts` | `wrapMessagesInSystemReminder()` - 在系统提醒中注入相关记忆 |
| `src/query/stopHooks.ts` | 查询停止时处理记忆相关逻辑 |

---

## 依赖与外部交互

### API 调用
- **sideQuery**: 使用 Sonnet 模型进行相关性判断
  - Model: `getDefaultSonnetModel()`
  - Max tokens: 256
  - Output format: JSON Schema
  - Query source: `'memdir_relevance'`

### Feature Flags
- `MEMORY_SHAPE_TELEMETRY`: 启用记忆召回形状遥测

### 错误处理策略
1. **中止信号**：如果 `signal.aborted`，立即返回空数组
2. **API 失败**：记录调试日志，返回空数组（优雅降级）
3. **JSON 解析失败**：通过 `jsonParse` 安全处理

---

## 风险、边界与改进建议

### 已知风险

1. **API 依赖风险**
   - 相关性判断完全依赖外部 API 调用
   - API 失败时返回空数组（保守策略）
   - 网络延迟可能影响响应时间

2. **选择数量限制**
   - 硬编码最多5个记忆，可能遗漏重要上下文
   - 保守的选择策略可能导致召回不足

3. **提示词缓存问题**
   - 每次查询都需要构造新的提示词
   - 无法利用提示词缓存优化

### 边界情况

1. **空记忆目录**：`scanMemoryFiles` 返回空数组，直接返回空结果
2. **全部记忆已展示**：`alreadySurfaced` 过滤后为空，返回空结果
3. **模型返回无效文件名**：通过 `validFilenames` 集合过滤
4. **recentTools 为空**：跳过工具排除逻辑

### 改进建议

1. **缓存优化**
   - 考虑对相似查询的记忆选择结果进行缓存
   - 使用查询指纹（fingerprint）作为缓存键

2. **自适应选择数量**
   - 根据查询复杂度动态调整选择数量
   - 基于记忆重要性评分而非固定数量

3. **批量处理优化**
   - 当前每次查询都重新扫描目录
   - 考虑使用文件系统监视器（watcher）增量更新

4. **更好的错误恢复**
   - 当前 API 失败直接返回空数组
   - 可考虑降级到基于关键词的本地匹配

5. **可观测性增强**
   - 添加更多指标：选择率、平均选择数量、API 延迟
   - 记录被过滤的记忆原因（用于调试）
