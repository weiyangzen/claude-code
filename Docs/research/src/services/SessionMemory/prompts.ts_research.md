# SessionMemory/prompts.ts 研究文档

## 场景与职责

`prompts.ts` 是 Session Memory 系统的提示词管理模块，负责：
1. **维护默认模板**：定义会话记忆文件的标准 Markdown 结构（9个章节）
2. **加载自定义配置**：支持用户通过 `~/.claude/session-memory/config/template.md` 和 `prompt.md` 自定义模板和提示词
3. **构建更新提示**：生成用于驱动 forked agent 更新会话记忆文件的完整提示词
4. **内容截断处理**：在 compaction 场景下对超长的 session memory 内容进行智能截断

该模块是 Session Memory 系统的"内容生成器"，与 `sessionMemory.ts`（协调器）和 `sessionMemoryUtils.ts`（状态管理）形成完整闭环。

## 功能点目的

### 1. 默认模板结构 (`DEFAULT_SESSION_MEMORY_TEMPLATE`)
定义9个标准章节：
- **Session Title**: 会话标题（5-10字）
- **Current State**: 当前正在进行的工作
- **Task specification**: 用户原始需求与设计决策
- **Files and Functions**: 关键文件及其内容概述
- **Workflow**: 常用 bash 命令及执行顺序
- **Errors & Corrections**: 遇到的错误及修复方法
- **Codebase and System Documentation**: 系统组件及其协作方式
- **Learnings**: 有效/无效的方法总结
- **Key results**: 用户请求的具体输出结果
- **Worklog**: 逐步工作记录

### 2. 默认更新提示 (`getDefaultUpdatePrompt`)
生成指导 forked agent 如何更新记忆文件的指令，核心约束：
- 严格保持模板结构（章节标题和斜体描述行不可修改）
- 仅更新实际内容区域
- 禁止提及"note-taking"或"session notes extraction"等元指令
- 每个章节内容限制在 ~2000 tokens
- 总内容限制在 12000 tokens

### 3. 自定义配置加载
支持两个自定义文件：
- `~/.claude/session-memory/config/template.md`: 自定义模板结构
- `~/.claude/session-memory/config/prompt.md`: 自定义更新提示词（支持 `{{variableName}}` 变量替换）

### 4. 章节大小分析与提醒
- `analyzeSectionSizes`: 解析记忆文件，计算各章节的 token 估算值
- `generateSectionReminders`: 生成章节大小超限提醒，驱动 AI 进行内容压缩

### 5. 内容截断 (`truncateSessionMemoryForCompact`)
在 compaction 场景下，防止 session memory 占用过多 token：
- 按章节逐个检查大小
- 超过 `MAX_SECTION_LENGTH * 4` 字符的章节在行边界处截断
- 添加 `[... section truncated for length ...]` 标记

## 具体技术实现

### 关键常量
```typescript
const MAX_SECTION_LENGTH = 2000        // 单章节 token 上限
const MAX_TOTAL_SESSION_MEMORY_TOKENS = 12000  // 总 token 上限
```

### 变量替换机制
```typescript
function substituteVariables(
  template: string,
  variables: Record<string, string>,
): string {
  // 单次替换避免两个问题：
  // 1. $ 后引用损坏（replacer 函数将 $ 视为字面量）
  // 2. 用户内容恰好包含 {{varName}} 时的双重替换
  return template.replace(/\{\{(\w+)\}\}/g, (match, key: string) =>
    Object.prototype.hasOwnProperty.call(variables, key)
      ? variables[key]!
      : match,
  )
}
```

### 章节解析算法
```typescript
function analyzeSectionSizes(content: string): Record<string, number> {
  // 按行解析，识别 "# " 开头的章节标题
  // 累计每个标题下的内容行
  // 使用 roughTokenCountEstimation(content.length / 4) 估算 token
}
```

### 截断算法
```typescript
function flushSessionSection(
  sectionHeader: string,
  sectionLines: string[],
  maxCharsPerSection: number,
): { lines: string[]; wasTruncated: boolean } {
  // 在行边界处截断（避免截断单词）
  // 保留章节标题，截断内容，添加截断标记
}
```

## 关键代码路径与文件引用

### 导出函数
| 函数名 | 用途 | 调用方 |
|--------|------|--------|
| `loadSessionMemoryTemplate()` | 加载自定义模板 | `sessionMemory.ts:setupSessionMemoryFile`, `isSessionMemoryEmpty` |
| `loadSessionMemoryPrompt()` | 加载自定义提示词 | `buildSessionMemoryUpdatePrompt` |
| `buildSessionMemoryUpdatePrompt()` | 构建完整更新提示 | `sessionMemory.ts:extractSessionMemory`, `manuallyExtractSessionMemory` |
| `isSessionMemoryEmpty()` | 检查记忆文件是否为空（仅模板） | `sessionMemoryCompact.ts:trySessionMemoryCompaction` |
| `truncateSessionMemoryForCompact()` | 截断过长内容 | `sessionMemoryCompact.ts:createCompactionResultFromSessionMemory` |

### 依赖模块
```typescript
// Token 估算
import { roughTokenCountEstimation } from '../../services/tokenEstimation.js'

// 配置目录路径
import { getClaudeConfigHomeDir } from '../../utils/envUtils.js'

// 错误处理
import { getErrnoCode, toError } from '../../utils/errors.js'
import { logError } from '../../utils/log.js'
```

## 依赖与外部交互

### 被调用方
1. **`sessionMemory.ts`**
   - `setupSessionMemoryFile`: 初始化记忆文件时使用默认模板
   - `extractSessionMemory`: 构建更新提示词触发 forked agent
   - `manuallyExtractSessionMemory`: /summary 命令手动触发

2. **`sessionMemoryCompact.ts`**
   - `isSessionMemoryEmpty`: 判断是否需要回退到传统 compaction
   - `truncateSessionMemoryForCompact`: 防止 session memory 占用过多 token

### 依赖的底层服务
1. **`tokenEstimation.ts`**
   - `roughTokenCountEstimation`: 基于字符长度的简单 token 估算（length / 4）

2. **`envUtils.ts`**
   - `getClaudeConfigHomeDir`: 获取 `~/.claude` 配置目录路径

3. **Node.js fs/promises**
   - `readFile`: 读取自定义模板和提示词文件

## 风险、边界与改进建议

### 风险点

1. **模板结构严格性**
   - 风险：AI 可能误修改章节标题或斜体描述行，破坏模板结构
   - 缓解：提示词中反复强调结构保护规则，但无法完全杜绝

2. **Token 估算精度**
   - 风险：`roughTokenCountEstimation` 使用简单的 length/4 估算，与实际 token 数可能有偏差
   - 影响：可能导致章节超限或过早截断

3. **自定义配置错误处理**
   - 行为：读取自定义配置失败时静默回退到默认值
   - 风险：用户可能意识不到自定义配置未生效

4. **变量替换安全风险**
   - 缓解：使用 `Object.prototype.hasOwnProperty.call` 防止原型链污染
   - 但：用户自定义提示词中的变量仍可能被意外替换

### 边界情况

1. **空内容处理**
   - `isSessionMemoryEmpty` 比较的是 trim 后的完整内容，区分大小写

2. **章节截断边界**
   - 在行边界处截断，避免截断单词
   - 但：可能导致某些行（如代码块）被截断

3. **超长章节处理**
   - 超过 2000 tokens 的章节会收到压缩提醒
   - 但：AI 可能无法有效压缩，导致反复超限

### 改进建议

1. **模板结构验证**
   - 添加运行时模板结构验证，检测并修复被破坏的模板

2. **更精确的 Token 估算**
   - 考虑使用更精确的估算方法（如按空格分词）

3. **自定义配置热重载**
   - 支持配置变更时自动重新加载

4. **章节压缩策略优化**
   - 为不同章节类型定义不同的压缩策略（如 Worklog 可更激进地压缩）

5. **添加元数据追踪**
   - 在记忆文件中添加版本信息，支持未来格式迁移
