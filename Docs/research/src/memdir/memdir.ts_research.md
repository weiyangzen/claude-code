# memdir.ts 研究文档

## 场景与职责

`memdir.ts` 是记忆系统的**核心提示词构建模块**，负责生成指导 Claude 如何使用记忆系统的系统提示词。它是连接记忆存储机制与模型行为的桥梁。

### 核心职责
1. **记忆提示词构建**：生成完整的记忆系统使用指南（包含类型说明、保存方法、访问时机等）
2. **入口点内容截断**：处理 `MEMORY.md` 文件的大小和行数限制
3. **目录存在性保证**：确保记忆目录存在，避免模型浪费时间检查
4. **多模式支持**：支持普通模式、KAIROS 助手模式、团队记忆模式

### 使用场景
- 系统提示词初始化时加载记忆指南
- Agent 记忆系统构建提示词
- 每日日志模式（KAIROS）的提示词生成

---

## 功能点目的

### 1. `truncateEntrypointContent()` - 入口点内容截断
**目的**：确保 `MEMORY.md` 内容不超过限制（200行 / 25KB）

**截断策略**：
- 先按行截断（自然边界）
- 再按字节截断（在最后一个换行符处截断，避免截断 mid-line）
- 追加警告信息说明截断原因

**限制常量**：
```typescript
export const MAX_ENTRYPOINT_LINES = 200
export const MAX_ENTRYPOINT_BYTES = 25_000
```

### 2. `ensureMemoryDirExists()` - 目录存在性保证
**目的**：确保记忆目录存在，避免模型执行 `ls`/`mkdir -p`

**实现细节**：
- 使用 `fs.mkdir`（递归模式）
- 内部已处理 `EEXIST`，无需 try/catch
- 仅记录真正的错误（`EACCES`/`EPERM`/`EROFS`）

### 3. `buildMemoryLines()` - 记忆指南构建（核心）
**目的**：构建类型化的记忆行为指导文本

**包含章节**：
- 记忆系统介绍
- 四种记忆类型说明（user/feedback/project/reference）
- 不应保存的内容指南
- 保存方法（单步或两步流程）
- 访问时机
- 信任召回内容的指导
- 与其他持久化机制的对比（Plan/Task）

### 4. `buildMemoryPrompt()` - 完整提示词构建
**目的**：构建包含 `MEMORY.md` 内容的完整提示词

**流程**：
1. 调用 `buildMemoryLines()` 获取基础指南
2. 读取 `MEMORY.md` 文件内容
3. 应用截断逻辑
4. 记录遥测数据
5. 组合成完整提示词

### 5. `buildAssistantDailyLogPrompt()` - KAIROS 每日日志模式
**目的**：为长期运行的助手会话构建追加式日志提示词

**特点**：
- 记忆写入日期命名的日志文件（`logs/YYYY/MM/YYYY-MM-DD.md`）
- 追加而非重写
- 夜间 `/dream` 技能负责蒸馏成主题文件

### 6. `loadMemoryPrompt()` - 统一加载入口
**目的**：根据启用的记忆系统类型分发到对应的提示词构建器

**分发逻辑**：
```
KAIROS + autoEnabled → buildAssistantDailyLogPrompt()
TEAMMEM + teamEnabled → buildCombinedMemoryPrompt()
autoEnabled → buildMemoryLines()
其他 → null（记忆禁用）
```

---

## 具体技术实现

### 关键流程

#### 标准记忆提示词构建流程
```
loadMemoryPrompt()
  ↓
检查 feature flags 和启用状态
  ↓
ensureMemoryDirExists()  // 确保目录存在
  ↓
buildMemoryLines() / buildCombinedMemoryPrompt() / buildAssistantDailyLogPrompt()
  ↓
返回提示词字符串
```

#### MEMORY.md 加载流程（buildMemoryPrompt）
```
buildMemoryPrompt({ displayName, memoryDir, extraGuidelines })
  ↓
fs.readFileSync(entrypoint)  // 同步读取
  ↓
truncateEntrypointContent(raw)  // 截断处理
  ↓
logMemoryDirCounts()  // 异步记录文件统计
  ↓
组合 lines + entrypoint 内容
```

### 数据结构

#### EntrypointTruncation
```typescript
export type EntrypointTruncation = {
  content: string           // 截断后的内容
  lineCount: number         // 原始行数
  byteCount: number         // 原始字节数
  wasLineTruncated: boolean // 是否因行数被截断
  wasByteTruncated: boolean // 是否因字节数被截断
}
```

### 提示词章节组织

```typescript
// 基础结构
[
  `# ${displayName}`,
  `You have a persistent, file-based memory system at \`${memoryDir}\`. ${DIR_EXISTS_GUIDANCE}`,
  ...TYPES_SECTION_INDIVIDUAL,    // 四种类型说明
  ...WHAT_NOT_TO_SAVE_SECTION,    // 不应保存的内容
  ...howToSave,                   // 保存方法
  ...WHEN_TO_ACCESS_SECTION,      // 访问时机
  ...TRUSTING_RECALL_SECTION,     // 信任召回内容
  '## Memory and other forms of persistence',
  ...buildSearchingPastContextSection(autoDir),  // 搜索历史上下文
]
```

---

## 关键代码路径与文件引用

### 内部依赖
| 文件 | 用途 |
|------|------|
| `memoryTypes.ts` | `TYPES_SECTION_*`, `WHAT_NOT_TO_SAVE_SECTION`, `WHEN_TO_ACCESS_SECTION`, `TRUSTING_RECALL_SECTION`, `MEMORY_FRONTMATTER_EXAMPLE` |
| `paths.ts` | `getAutoMemPath()`, `isAutoMemoryEnabled()` |
| `teamMemPaths.ts` | `isTeamMemoryEnabled()`, `getTeamMemPath()`（条件加载） |
| `teamMemPrompts.ts` | `buildCombinedMemoryPrompt()`（条件加载） |

### 外部依赖
| 文件 | 用途 |
|------|------|
| `../bootstrap/state.ts` | `getKairosActive()`, `getOriginalCwd()` |
| `../services/analytics/growthbook.ts` | `getFeatureValue_CACHED_MAY_BE_STALE()` |
| `../services/analytics/index.ts` | `logEvent()` |
| `../utils/fsOperations.ts` | `getFsImplementation()` |
| `../utils/sessionStorage.ts` | `getProjectDir()` |
| `../utils/settings/settings.ts` | `getInitialSettings()` |
| `../utils/debug.ts` | `logForDebugging()` |
| `../utils/format.ts` | `formatFileSize()` |
| `../utils/envUtils.ts` | `isEnvTruthy()` |
| `../utils/embeddedTools.ts` | `hasEmbeddedSearchTools()` |
| `../tools/GrepTool/prompt.ts` | `GREP_TOOL_NAME` |
| `../tools/REPLTool/constants.ts` | `isReplModeEnabled()` |

### 调用方
| 文件 | 用途 |
|------|------|
| `src/constants/prompts.ts` | 系统提示词构建 |
| `src/utils/claudemd.ts` | `truncateEntrypointContent()` 复用 |
| `src/tools/AgentTool/agentMemory.ts` | Agent 记忆提示词 |
| `src/components/agents/generateAgent.ts` | Agent 生成 |
| `src/components/agents/new-agent-creation/` | Agent 创建向导 |

---

## 依赖与外部交互

### Feature Flags
| Flag | 用途 |
|------|------|
| `TEAMMEM` | 启用团队记忆支持 |
| `KAIROS` | 启用助手每日日志模式 |
| `tengu_coral_fern` | 启用"搜索历史上下文"章节 |
| `tengu_moth_copse` | 跳过索引步骤（skipIndex） |

### 环境变量
| 变量 | 用途 |
|------|------|
| `CLAUDE_COWORK_MEMORY_EXTRA_GUIDELINES` | 注入额外的记忆策略指南 |
| `CLAUDE_CODE_DISABLE_AUTO_MEMORY` | 禁用自动记忆 |

### 遥测事件
| 事件 | 说明 |
|------|------|
| `tengu_memdir_loaded` | 记忆目录加载（包含文件/子目录计数） |
| `tengu_memdir_disabled` | 记忆被禁用（记录禁用原因） |
| `tengu_team_memdir_disabled` | 团队记忆被禁用 |

---

## 风险、边界与改进建议

### 已知风险

1. **同步文件读取**
   - `buildMemoryPrompt()` 使用 `readFileSync`
   - 在提示词构建的关键路径上，可能阻塞事件循环
   - 文件较大时可能影响启动性能

2. **提示词缓存失效**
   - `MEMORY.md` 内容变化会导致提示词缓存失效
   - 频繁编辑 MEMORY.md 可能影响性能

3. **硬编码限制**
   - 200行/25KB 限制是经验值，可能不适合所有场景
   - 长行索引条目可能意外被截断

### 边界情况

1. **MEMORY.md 不存在**：显示空状态提示
2. **目录不可读**：`logMemoryDirCounts` 失败静默处理
3. **Cowork 覆盖路径**：支持 `CLAUDE_COWORK_MEMORY_PATH_OVERRIDE`
4. **团队记忆目录不存在**：递归创建时同时创建父目录

### 改进建议

1. **异步化改造**
   - 将 `buildMemoryPrompt` 改为异步
   - 使用 `readFile` 替代 `readFileSync`

2. **智能截断**
   - 基于语义而非纯机械截断
   - 保留最重要的索引条目

3. **增量更新**
   - 支持 MEMORY.md 的增量更新检测
   - 仅在有变化时重建提示词

4. **可配置限制**
   - 将行数/字节限制改为可配置
   - 基于模型上下文窗口动态调整

5. **更好的错误处理**
   - 区分临时错误（权限）和永久错误（不存在）
   - 提供用户友好的错误提示
