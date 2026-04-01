# plans.ts 研究文档

## 场景与职责

本模块提供计划文件（Plan）的完整生命周期管理功能。核心职责包括：

1. **计划标识管理**：生成和管理会话的计划 slug（单词短语标识符）
2. **计划目录管理**：确定计划文件存储目录
3. **计划文件读写**：读取和写入计划文件内容
4. **会话恢复支持**：从日志恢复计划文件（用于远程会话）
5. **文件快照**：在远程会话中持久化计划文件到转录

该模块是 Plan Mode 功能的核心基础设施，支持主会话和子代理的计划文件管理。

## 功能点目的

### 1. `getPlanSlug()` / `setPlanSlug()` / `clearPlanSlug()` - Slug 管理
- **目的**：生成和管理会话的计划文件标识
- **Slug 格式**：`adjective-verb-noun`（如 `gleaming-brewing-phoenix`）
- **唯一性保证**：生成后检查文件是否存在，最多重试 10 次
- **缓存**：使用 `getPlanSlugCache()` 按 sessionId 缓存

### 2. `getPlansDirectory()` - 计划目录获取
- **目的**：确定计划文件的存储目录
- **优先级**：
  1. `settings.json` 中的 `plansDirectory`（相对于项目根目录）
  2. 默认：`~/.claude/plans/`
- **安全检查**：验证路径在项目根目录内，防止路径遍历
- **优化**：使用 `lodash/memoize` 缓存，避免重复 mkdir

### 3. `getPlanFilePath()` - 计划文件路径
- **目的**：生成计划文件的完整路径
- **命名规则**：
  - 主会话：`{slug}.md`
  - 子代理：`{slug}-agent-{agentId}.md`

### 4. `getPlan()` - 计划内容读取
- **目的**：读取计划文件内容
- **容错**：文件不存在时返回 `null`，其他错误记录日志

### 5. `copyPlanForResume()` - 恢复时计划复制
- **目的**：会话恢复时恢复计划文件
- **流程**：
  1. 从日志消息中提取 slug
  2. 尝试直接读取计划文件
  3. 文件不存在且为远程会话时尝试恢复：
     - 从文件快照恢复
     - 从消息历史恢复（ExitPlanMode 工具输入、planContent 字段、plan_file_reference 附件）
  4. 写入恢复的计划文件

### 6. `copyPlanForFork()` - Fork 时计划复制
- **目的**：Fork 会话时复制计划文件
- **特点**：生成新的 slug，避免原会话和 Fork 会话互相覆盖

### 7. `persistFileSnapshotIfRemote()` - 远程会话文件快照
- **目的**：在远程会话（CCR）中将计划文件增量保存到转录
- **触发**：计划文件变更时调用
- **用途**：远程会话中本地文件不持久化，通过转录保留计划内容

## 具体技术实现

### 关键流程

#### Slug 生成流程

```
getPlanSlug(sessionId?)
    ↓
从缓存获取
    ↓
存在? → 返回
    ↓
不存在 → generateWordSlug()（形容词-动词-名词）
    ↓
检查文件是否存在（最多 10 次重试）
    ↓
存入缓存 → 返回
```

#### 恢复流程

```
copyPlanForResume(log, targetSessionId?)
    ↓
从日志消息提取 slug
    ↓
设置 slug 到目标会话
    ↓
尝试读取计划文件
    ↓
成功? → 返回 true
    ↓
失败 ENOENT 且为远程会话?
    否 → 返回 false
    是 → 尝试恢复
        ↓
        从文件快照查找
        找到? → 使用该内容
        否 → 从消息历史恢复
            ↓
            扫描消息（倒序）
            - ExitPlanMode 工具输入中的 plan
            - UserMessage 的 planContent 字段
            - AttachmentMessage 的 plan_file_reference
        ↓
        找到内容? → 写入文件 → 返回 true
        否 → 返回 false
```

### 数据结构

```typescript
// 计划文件快照条目
{
  key: 'plan'
  path: string
  content: string
}

// 内容替换记录（用于恢复）
type ContentReplacementRecord = {
  kind: 'tool-result'
  toolUseId: string
  replacement: string
}
```

### 文件路径结构

```
~/.claude/plans/
├── {slug}.md                    # 主会话计划
├── {slug}-agent-{agentId}.md    # 子代理计划
└── ...
```

### 恢复来源优先级

1. **文件快照**（`findFileSnapshotEntry`）
   - 在远程会话中增量写入
   - 最可靠的恢复来源

2. **ExitPlanMode 工具输入**
   - `normalizeToolInput` 注入的计划内容
   - 存在于转录中

3. **UserMessage.planContent**
   - "clear context and implement" 流程设置

4. **AttachmentMessage.plan_file_reference**
   - auto-compact 创建的附件

## 依赖与外部交互

### 直接依赖

| 模块 | 用途 |
|------|------|
| `crypto` | `randomUUID()` 用于快照消息 |
| `fs/promises` | 文件操作 |
| `lodash-es/memoize.js` | 计划目录缓存 |
| `path` | 路径操作 |
| `../bootstrap/state.js` | `getPlanSlugCache()`, `getSessionId()` |
| `../tools/ExitPlanModeTool/constants.js` | `EXIT_PLAN_MODE_V2_TOOL_NAME` |
| `./cwd.js` | `getCwd()` |
| `./debug.js` | 调试日志 |
| `./envUtils.js` | `getClaudeConfigHomeDir()` |
| `./errors.js` | `isENOENT()` |
| `./filePersistence/outputsScanner.js` | `getEnvironmentKind()` |
| `./fsOperations.js` | 文件系统操作 |
| `./log.js` | 错误日志 |
| `./settings/settings.js` | `getInitialSettings()` |
| `./words.js` | `generateWordSlug()` |

### 调用方

| 调用方 | 用途 |
|--------|------|
| `src/services/compact/compact.ts` | 压缩时处理计划文件 |
| `src/setup.ts` | 初始化时恢复计划 |
| `src/commands/clear/conversation.ts` | 清除命令处理计划 |
| `src/screens/REPL.tsx` | REPL 界面计划操作 |
| `src/commands/plan/plan.tsx` | 计划命令 |
| `src/tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts` | 退出计划模式 |
| `src/tools/FileWriteTool/UI.tsx` | 文件写入 UI |
| `src/tools/FileReadTool/UI.tsx` | 文件读取 UI |
| `src/tools/FileEditTool/UI.tsx` | 文件编辑 UI |
| 以及其他 UI 组件 | 计划文件显示 |

## 风险、边界与改进建议

### 已知风险

1. **Slug 冲突**
   - 风险：`generateWordSlug()` 可能生成已存在的 slug
   - 缓解：最多 10 次重试
   - 潜在问题：极端情况下仍可能冲突

2. **恢复失败**
   - 风险：远程会话中计划文件可能无法恢复
   - 缓解：多来源恢复（快照、消息历史）
   - 潜在问题：所有来源都失败时计划丢失

3. **并发修改**
   - 风险：主会话和子代理同时修改计划
   - 现状：各自独立的文件（`-agent-{id}.md`）
   - 潜在问题：合并逻辑复杂

4. **缓存一致性**
   - 风险：`getPlansDirectory()` 的 memoize 缓存
   - 潜在问题：设置变更后未刷新缓存

### 边界情况

| 场景 | 行为 |
|------|------|
| Slug 生成 10 次都冲突 | 使用第 10 个 slug（可能覆盖） |
| 计划目录在项目外 | 回退到 `~/.claude/plans/`，记录错误 |
| 计划文件读取失败 | 返回 `null`，记录错误 |
| 恢复时无日志 slug | 返回 `false` |
| 非远程会话恢复 | 不尝试恢复，直接返回文件是否存在 |
| 快照无 plan 条目 | 尝试消息历史恢复 |
| 消息历史无计划 | 恢复失败，记录日志 |

### 改进建议

1. **Slug 生成优化**
   - 当前：随机单词组合
   - 建议：
     - 添加时间戳成分降低冲突
     - 使用 UUID 作为后备
     - 冲突时自动递增后缀

2. **原子写入**
   - 当前：直接写入目标文件
   - 建议：
     - 写入临时文件后重命名
     - 防止写入中断导致文件损坏

3. **版本控制**
   - 建议：
     - 计划文件版本历史
     - 支持回滚到之前版本
     - 显示变更差异

4. **同步机制**
   - 建议：
     - 主会话和子代理计划同步
     - 冲突检测和解决策略

5. **恢复增强**
   - 建议：
     - 恢复时验证内容完整性
     - 提供恢复报告（成功/失败来源）
     - 支持手动选择恢复来源

6. **缓存管理**
   - 建议：
     - 监听设置变更自动刷新缓存
     - 提供手动刷新接口

7. **遥测集成**
   - 建议：
     - 记录恢复成功率
     - 记录恢复来源分布
     - 记录 slug 冲突频率

8. **测试覆盖**
   - 建议：
     - 恢复流程单元测试
     - 并发场景测试
     - 边界条件测试
