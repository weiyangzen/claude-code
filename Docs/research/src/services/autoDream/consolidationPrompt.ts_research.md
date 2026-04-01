# consolidationPrompt.ts 深度研究文档

## 1. 场景与职责

### 1.1 模块定位
`consolidationPrompt.ts` 是 auto-dream 模块的**提示词构建器**，负责生成发送给 forked agent 的 "Dream" 提示词，指导其执行记忆整合任务。

### 1.2 业务场景
- **记忆整合指导**：告诉 AI 如何回顾、整理和归档历史会话信息
- **文件系统导航**：指导 AI 在记忆目录和会话记录中高效查找信息
- **结构化输出**：确保整合后的记忆符合预期的格式和组织结构

### 1.3 核心职责
1. **提示词模板管理**：定义四阶段记忆整合流程（Orient/Gather/Consolidate/Prune）
2. **路径信息注入**：动态插入记忆目录和会话目录路径
3. **工具约束说明**：告知 AI 可用的工具和限制条件

---

## 2. 功能点目的

### 2.1 四阶段整合流程

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Orient    │ → │   Gather    │ → │ Consolidate │ → │    Prune    │
│  (定向)      │    │  (收集)      │    │  (整合)      │    │  (修剪)      │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
```

| 阶段 | 目的 | 关键动作 |
|------|------|----------|
| **Orient** | 了解当前记忆结构 | `ls` 记忆目录，阅读 `MEMORY.md` |
| **Gather** | 识别新信息 | 查看日志、对比现有记忆、搜索会话 |
| **Consolidate** | 写入/更新记忆 | 合并新信号到主题文件，删除矛盾信息 |
| **Prune** | 维护索引质量 | 保持 `MEMORY.md` 在 200 行/25KB 以内 |

### 2.2 与 KAIROS 模式的关系
- **独立设计**：本模块从 `dream.ts` 提取，确保 auto-dream 不依赖 KAIROS 功能标志
- **共存策略**：KAIROS 模式使用磁盘技能（disk-skill）的 `/dream`，auto-dream 使用 forked agent

### 2.3 提示词设计原则
1. **具体指令**：明确的步骤和工具使用指导
2. **约束清晰**：文件大小限制、索引格式要求
3. **效率优先**：避免全量读取大文件，推荐使用 `grep` 精准搜索

---

## 3. 具体技术实现

### 3.1 代码实现
```typescript
import {
  DIR_EXISTS_GUIDANCE,
  ENTRYPOINT_NAME,
  MAX_ENTRYPOINT_LINES,
} from '../../memdir/memdir.js'

export function buildConsolidationPrompt(
  memoryRoot: string,
  transcriptDir: string,
  extra: string,
): string {
  return `# Dream: Memory Consolidation

You are performing a dream — a reflective pass over your memory files...

Memory directory: \`${memoryRoot}\`
${DIR_EXISTS_GUIDANCE}

Session transcripts: \`${transcriptDir}\`...

## Phase 1 — Orient
...

## Phase 2 — Gather recent signal
...

## Phase 3 — Consolidate
...

## Phase 4 — Prune and index
...

Return a brief summary...${extra ? `\n\n## Additional context\n\n${extra}` : ''}`
}
```

### 3.2 提示词结构详解

#### 3.2.1 头部信息
```markdown
# Dream: Memory Consolidation

You are performing a dream — a reflective pass over your memory files. 
Synthesize what you've learned recently into durable, well-organized memories 
so that future sessions can orient quickly.

Memory directory: `/home/user/.claude/projects/repo/memory/`
> If the directory doesn't exist yet, create it.

Session transcripts: `/home/user/.claude/projects/repo/sessions/` 
(large JSONL files — grep narrowly, don't read whole files)
```

**关键信息**：
- **任务定义**：明确这是"梦境"（Dream）任务
- **目标说明**：将近期学习整合为持久化记忆
- **路径注入**：动态插入 `memoryRoot` 和 `transcriptDir`
- **性能提示**：提醒会话文件很大，使用 `grep` 而非全量读取

#### 3.2.2 Phase 1 — Orient（定向阶段）
```markdown
## Phase 1 — Orient

- `ls` the memory directory to see what already exists
- Read `MEMORY.md` to understand the current index
- Skim existing topic files so you improve them rather than creating duplicates
- If `logs/` or `sessions/` subdirectories exist (assistant-mode layout), 
  review recent entries there
```

**目的**：让 AI 了解当前记忆结构，避免重复创建或破坏现有组织。

#### 3.2.3 Phase 2 — Gather（收集阶段）
```markdown
## Phase 2 — Gather recent signal

Look for new information worth persisting. Sources in rough priority order:

1. **Daily logs** (`logs/YYYY/MM/YYYY-MM-DD.md`) if present — append-only stream
2. **Existing memories that drifted** — facts contradicting current codebase
3. **Transcript search** — grep JSONL for narrow terms:
   `grep -rn "<narrow term>" /path/to/transcripts/ --include="*.jsonl" | tail -50`

Don't exhaustively read transcripts. Look only for things you already suspect matter.
```

**优先级设计**：
1. **日志优先**：KAIROS 模式的每日日志是最高效的信息源
2. **差异检测**：识别与当前代码库矛盾的旧记忆
3. **精准搜索**：基于已有线索搜索会话记录，避免全量扫描

#### 3.2.4 Phase 3 — Consolidate（整合阶段）
```markdown
## Phase 3 — Consolidate

For each thing worth remembering, write or update a memory file...

Focus on:
- Merging new signal into existing topic files rather than creating near-duplicates
- Converting relative dates ("yesterday", "last week") to absolute dates
- Deleting contradicted facts — if today's investigation disproves an old memory, fix it
```

**关键约束**：
- **合并而非创建**：优先更新现有主题文件
- **绝对时间**：将相对时间转换为绝对日期，确保长期可读
- **矛盾处理**：主动删除已被证伪的旧记忆

#### 3.2.5 Phase 4 — Prune and Index（修剪和索引阶段）
```markdown
## Phase 4 — Prune and index

Update `MEMORY.md` so it stays under 200 lines AND under ~25KB. 
It's an **index**, not a dump — each entry should be one line under ~150 characters: 
`- [Title](file.md) — one-line hook`. Never write memory content directly into it.

- Remove pointers to stale/wrong/superseded memories
- Demote verbose entries: if over ~200 chars, shorten and move detail
- Add pointers to newly important memories
- Resolve contradictions — if two files disagree, fix the wrong one
```

**索引维护规则**：
- **大小限制**：200 行 或 25KB（软限制）
- **格式规范**：单行条目，包含标题、链接和简介
- **内容分离**：索引只存指针，详细内容放在主题文件

### 3.3 额外上下文注入
```typescript
const extra = `

**Tool constraints for this run:** Bash is restricted to read-only commands 
(\`ls\`, \`find\`, \`grep\`, \`cat\`, \`stat\`, \`wc\`, \`head\`, \`tail\`, and similar). 
Anything that writes, redirects to a file, or modifies state will be denied.

Sessions since last consolidation (${sessionIds.length}):
${sessionIds.map(id => `- ${id}`).join('\n')}`
```

**动态注入内容**：
- **工具约束**：明确告知只读 Bash 限制
- **会话列表**：列出本次需要回顾的会话 ID，帮助 AI 了解工作范围

---

## 4. 关键代码路径与文件引用

### 4.1 调用方
| 文件 | 调用位置 | 用途 |
|------|----------|------|
| `autoDream.ts:222` | `buildConsolidationPrompt()` | 构建 forked agent 提示词 |

### 4.2 调用链
```
buildConsolidationPrompt (consolidationPrompt.ts:10)
  ├── ENTRYPOINT_NAME → memdir.ts:34 (值为 'MEMORY.md')
  ├── DIR_EXISTS_GUIDANCE → memdir.ts (目录不存在时的指导文本)
  └── MAX_ENTRYPOINT_LINES → memdir.ts:35 (值为 200)
```

### 4.3 常量定义来源
| 常量 | 来源文件 | 值 | 说明 |
|------|----------|-----|------|
| `ENTRYPOINT_NAME` | `memdir.ts:34` | `'MEMORY.md'` | 记忆索引文件名 |
| `MAX_ENTRYPOINT_LINES` | `memdir.ts:35` | `200` | 索引文件行数限制 |
| `MAX_ENTRYPOINT_BYTES` | `memdir.ts:38` | `25000` | 索引文件字节限制 |
| `DIR_EXISTS_GUIDANCE` | `memdir.ts` | 动态文本 | 目录不存在指导 |

---

## 5. 依赖与外部交互

### 5.1 导入依赖
```typescript
import {
  DIR_EXISTS_GUIDANCE,
  ENTRYPOINT_NAME,
  MAX_ENTRYPOINT_LINES,
} from '../../memdir/memdir.js'
```

### 5.2 依赖说明
| 依赖 | 用途 |
|------|------|
| `ENTRYPOINT_NAME` | 在提示词中引用 MEMORY.md |
| `MAX_ENTRYPOINT_LINES` | 告知 AI 索引文件的行数限制 |
| `DIR_EXISTS_GUIDANCE` | 指导 AI 在目录不存在时如何创建 |

### 5.3 与 memdir 模块的关系
```
memdir.ts (记忆目录核心)
  ├── ENTRYPOINT_NAME = 'MEMORY.md'
  ├── MAX_ENTRYPOINT_LINES = 200
  ├── MAX_ENTRYPOINT_BYTES = 25000
  ├── DIR_EXISTS_GUIDANCE
  └── truncateEntrypointContent()  // 索引截断逻辑

consolidationPrompt.ts
  └── 使用上述常量构建提示词
```

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 提示词版本漂移
- **风险**：`memdir.ts` 中的常量变更后，提示词可能未同步更新
- **示例**：`MAX_ENTRYPOINT_LINES` 从 200 改为 300，但提示词仍说 200
- **缓解**：使用常量引用而非硬编码，确保单一数据源

#### 6.1.2 路径注入安全风险
- **风险**：`memoryRoot` 或 `transcriptDir` 包含特殊字符可能导致提示词注入
- **评估**：路径来自受控的系统函数，风险较低
- **建议**：对注入路径进行转义验证

#### 6.1.3 提示词过长
- **风险**：`extra` 中的会话列表可能很长，导致提示词超出模型上下文
- **当前**：提示词本身约 2KB，会话列表每个 ID 约 40 字节
- **计算**：即使 100 个会话，额外内容约 4KB，仍在安全范围

### 6.2 边界条件

| 场景 | 行为 |
|------|------|
| `extra` 为空字符串 | 不渲染额外上下文部分 |
| `sessionIds` 为空数组 | 显示 "Sessions since last consolidation (0):" 空列表 |
| 记忆目录不存在 | 提示词包含 `DIR_EXISTS_GUIDANCE` 指导创建 |
| 会话目录权限不足 | AI 执行 `ls`/`grep` 时会收到权限错误，需自行处理 |

### 6.3 改进建议

#### 6.3.1 国际化支持
```typescript
// 建议：支持多语言提示词
import { getLocale } from '../../utils/locale.js'

const PROMPTS = {
  en: { /* 英文提示词 */ },
  zh: { /* 中文提示词 */ },
  // ...
}

export function buildConsolidationPrompt(
  memoryRoot: string,
  transcriptDir: string,
  extra: string,
  locale = 'en'
): string {
  const t = PROMPTS[locale] || PROMPTS.en
  return t.template.replace('{memoryRoot}', memoryRoot)...
}
```

#### 6.3.2 会话列表截断
```typescript
// 建议：限制会话列表长度，避免极端情况
const MAX_SESSIONS_IN_PROMPT = 50
const displayedSessions = sessionIds.slice(0, MAX_SESSIONS_IN_PROMPT)
const omittedCount = sessionIds.length - MAX_SESSIONS_IN_PROMPT

const extra = `
Sessions since last consolidation (${sessionIds.length}):
${displayedSessions.map(id => `- ${id}`).join('\n')}
${omittedCount > 0 ? `... and ${omittedCount} more sessions` : ''}
`
```

#### 6.3.3 提示词版本追踪
```typescript
// 建议：在提示词中添加版本标识，便于调试
const PROMPT_VERSION = '2024-01-15-v1'

export function buildConsolidationPrompt(...): string {
  return `# Dream: Memory Consolidation (v${PROMPT_VERSION})
...
`
}
```

#### 6.3.4 动态阶段调整
```typescript
// 建议：根据会话数量动态调整提示词重点
interface PromptOptions {
  sessionCount: number
  hasDailyLogs: boolean
  memoryFileCount: number
}

export function buildConsolidationPrompt(
  memoryRoot: string,
  transcriptDir: string,
  extra: string,
  options: PromptOptions
): string {
  let phases = [/* 默认四阶段 */]
  
  if (options.sessionCount === 0) {
    // 跳过 Gather 阶段，专注维护现有记忆
    phases = phases.filter(p => p.name !== 'Gather')
  }
  
  if (options.memoryFileCount > 100) {
    // 增加归档指导
    phases.push(ARCHIVE_PHASE)
  }
  
  return buildPromptFromPhases(phases)
}
```

#### 6.3.5 A/B 测试支持
```typescript
// 建议：支持提示词变体测试
import { getFeatureValue_CACHED_MAY_BE_STALE } from '../analytics/growthbook.js'

const PROMPT_VARIANTS = {
  control: buildControlPrompt,
  detailed: buildDetailedPrompt,
  concise: buildConcisePrompt,
}

export function buildConsolidationPrompt(...): string {
  const variant = getFeatureValue_CACHED_MAY_BE_STALE('tengu_dream_prompt_variant', 'control')
  const builder = PROMPT_VARIANTS[variant] || PROMPT_VARIANTS.control
  return builder(memoryRoot, transcriptDir, extra)
}
```

### 6.4 测试建议
建议增加以下测试：
1. 提示词包含所有必需的占位符替换
2. `extra` 为空时不包含额外上下文标记
3. 特殊字符在路径中被正确转义
4. 提示词长度在合理范围内（< 8KB）
5. 提示词结构符合预期（包含所有四个阶段）
