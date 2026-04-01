# prompt.ts 研究文档

## 场景与职责

`prompt.ts` 负责生成 Bash 工具的**系统提示词（System Prompt）**。这些提示词指导 Claude 如何正确使用 Bash 工具，包括工具选择偏好、命令执行最佳实践、Git 操作指南、沙盒配置说明等。

**核心职责：**
1. 生成 Bash 工具的系统提示词
2. 提供工具使用偏好指导（优先使用专用工具而非 Bash）
3. 包含 Git 提交和 PR 创建详细指南
4. 集成沙盒配置信息
5. 支持不同用户类型（ant 用户 vs 外部用户）的差异化提示

## 功能点目的

### 1. 工具使用偏好指导

引导 Claude 优先使用专用工具而非 Bash 命令：
- **文件搜索**：使用 `GlobTool` 而非 `find`
- **内容搜索**：使用 `GrepTool` 而非 `grep`
- **读取文件**：使用 `FileReadTool` 而非 `cat/head/tail`
- **编辑文件**：使用 `FileEditTool` 而非 `sed/awk`
- **写入文件**：使用 `FileWriteTool` 而非 `echo >`

### 2. 命令执行指导

- 超时设置（默认和最大值）
- 后台任务使用指南
- 多命令并行 vs 串行执行
- 避免不必要的 `sleep` 命令

### 3. Git 操作指南

提供详细的 Git 工作流指导：
- 提交前检查（status, diff, log）
- 提交消息格式
- PR 创建流程
- 安全协议（不跳过 hooks，不强制推送等）

### 4. 沙盒配置集成

当沙盒启用时，向模型说明：
- 文件系统限制（允许/拒绝的路径）
- 网络限制（允许/拒绝的主机）
- 沙盒绕过条件

### 5. Undercover 模式

为 ant 内部用户提供特殊指令：
- 隐藏模型身份标识
- 剥离归因信息
- 内部技能使用指南

## 具体技术实现

### 核心函数

#### 1. getSimplePrompt（主入口）

```typescript
export function getSimplePrompt(): string
```

返回完整的 Bash 工具系统提示词，包含多个部分：
1. 工具描述和执行说明
2. 工具使用偏好
3. 详细指令（多命令、Git、sleep 等）
4. 沙盒配置（如启用）
5. Git 和 PR 指令

#### 2. getSimpleSandboxSection（沙盒提示词）

```typescript
function getSimpleSandboxSection(): string
```

生成沙盒配置说明：
- 文件系统读写限制
- 网络访问限制
- Unix 套接字配置
- 沙盒绕过指南（如果允许）

**关键实现细节**：
```typescript
// 将 UID 特定的临时目录路径替换为 $TMPDIR
// 避免跨用户全局提示缓存失效
const normalizeAllowOnly = (paths: string[]): string[] =>
  [...new Set(paths)].map(p => (p === claudeTempDir ? '$TMPDIR' : p))
```

#### 3. getCommitAndPRInstructions（Git 指南）

```typescript
function getCommitAndPRInstructions(): string
```

生成 Git 提交和 PR 创建指南，支持两种用户类型：

**Ant 用户**：
- 使用 `/commit` 和 `/commit-push-pr` 技能
- 简化的技能引用格式
- Undercover 指令（如启用）

**外部用户**：
- 完整的内联指令
- 详细的步骤说明
- 归因文本处理

#### 4. getBackgroundUsageNote（后台任务说明）

```typescript
function getBackgroundUsageNote(): string | null
```

返回后台任务使用说明，如果后台任务被禁用则返回 `null`。

### 提示词结构

```
getSimplePrompt()
├── 工具描述
│   └── "Executes a given bash command and returns its output."
├── 工具使用偏好
│   ├── 专用工具优先于 Bash
│   └── prependBullets(toolPreferenceItems)
├── 详细指令
│   ├── 目录/文件创建前检查
│   ├── 空格路径引用
│   ├── 超时设置
│   ├── 后台任务使用
│   ├── 多命令执行指南
│   ├── Git 命令指南
│   └── 避免 sleep 命令
├── 沙盒配置（条件）
│   └── getSimpleSandboxSection()
└── Git 和 PR 指令（条件）
    └── getCommitAndPRInstructions()
```

### 用户类型差异化

#### Ant 用户提示词特点

```typescript
if (process.env.USER_TYPE === 'ant') {
  // 1. Undercover 指令（如启用）
  const undercoverSection = isUndercover() ? getUndercoverInstructions() + '\n' : ''
  
  // 2. 技能引用（简化模式除外）
  const skillsSection = !isEnvTruthy(process.env.CLAUDE_CODE_SIMPLE)
    ? `For git commits and pull requests, use the \`/commit\` and \`/commit-push-pr\` skills...`
    : ''
  
  // 3. 简化的 Git 指南
  return `${undercoverSection}# Git operations\n\n${skillsSection}IMPORTANT: NEVER skip hooks...`
}
```

#### 外部用户提示词特点

```typescript
// 完整的内联 Git 指南
const { commit: commitAttribution, pr: prAttribution } = getAttributionTexts()

return `# Committing changes with git

Only create commits when requested by the user...
// 详细的 4 步流程说明
`
```

### 沙盒配置提示词

```typescript
function getSimpleSandboxSection(): string {
  if (!SandboxManager.isSandboxingEnabled()) {
    return ''
  }
  
  const fsReadConfig = SandboxManager.getFsReadConfig()
  const fsWriteConfig = SandboxManager.getFsWriteConfig()
  const networkRestrictionConfig = SandboxManager.getNetworkRestrictionConfig()
  
  // 去重配置路径（节省 token）
  const filesystemConfig = {
    read: { denyOnly: dedup(fsReadConfig.denyOnly), ... },
    write: { allowOnly: normalizeAllowOnly(fsWriteConfig.allowOnly), ... }
  }
  
  // 生成限制说明
  const restrictionsLines = []
  if (Object.keys(filesystemConfig).length > 0) {
    restrictionsLines.push(`Filesystem: ${jsonStringify(filesystemConfig)}`)
  }
  
  // 沙盒绕过指南（如果允许）
  const sandboxOverrideItems = allowUnsandboxedCommands
    ? ['You should always default to running commands within the sandbox...']
    : ['All commands MUST run in sandbox mode...']
}
```

## 关键代码路径与文件引用

### 调用关系

```
BashTool.tsx
  └── getSimplePrompt() [本文件导出]
        ├── getSimpleSandboxSection()
        │     ├── SandboxManager [src/utils/sandbox/sandbox-adapter.js]
        │     ├── getClaudeTempDir() [src/utils/permissions/filesystem.js]
        │     └── jsonStringify() [src/utils/slowOperations.js]
        └── getCommitAndPRInstructions()
              ├── getUndercoverInstructions() [src/utils/undercover.js]
              ├── shouldIncludeGitInstructions() [src/utils/gitSettings.js]
              ├── getAttributionTexts() [src/utils/attribution.js]
              └── TodoWriteTool, AGENT_TOOL_NAME 等工具引用
```

### 依赖文件

| 文件路径 | 依赖内容 | 用途 |
|---------|---------|------|
| `src/constants/prompts.js` | `prependBullets` | 提示词格式化 |
| `src/utils/attribution.js` | `getAttributionTexts` | 归因文本 |
| `src/utils/embeddedTools.js` | `hasEmbeddedSearchTools` | 嵌入式工具检测 |
| `src/utils/envUtils.js` | `isEnvTruthy` | 环境变量检查 |
| `src/utils/gitSettings.js` | `shouldIncludeGitInstructions` | Git 指令开关 |
| `src/utils/permissions/filesystem.js` | `getClaudeTempDir` | 临时目录路径 |
| `src/utils/sandbox/sandbox-adapter.js` | `SandboxManager` | 沙盒配置 |
| `src/utils/slowOperations.js` | `jsonStringify` | JSON 序列化 |
| `src/utils/timeouts.js` | `getDefaultBashTimeoutMs`, `getMaxBashTimeoutMs` | 超时配置 |
| `src/utils/undercover.js` | `getUndercoverInstructions`, `isUndercover` | Undercover 模式 |
| `src/tools/AgentTool/constants.js` | `AGENT_TOOL_NAME` | 工具名称 |
| `src/tools/FileEditTool/constants.js` | `FILE_EDIT_TOOL_NAME` | 工具名称 |
| `src/tools/FileReadTool/prompt.js` | `FILE_READ_TOOL_NAME` | 工具名称 |
| `src/tools/FileWriteTool/prompt.js` | `FILE_WRITE_TOOL_NAME` | 工具名称 |
| `src/tools/GlobTool/prompt.js` | `GLOB_TOOL_NAME` | 工具名称 |
| `src/tools/GrepTool/prompt.js` | `GREP_TOOL_NAME` | 工具名称 |
| `src/tools/TodoWriteTool/TodoWriteTool.js` | `TodoWriteTool` | 工具类 |
| `./toolName.js` | `BASH_TOOL_NAME` | 工具名称 |

### 被引用位置

| 文件路径 | 引用方式 | 用途 |
|---------|---------|------|
| `src/tools/BashTool/BashTool.tsx` | `import { getSimplePrompt, getDefaultTimeoutMs, getMaxTimeoutMs } from './prompt.js'` | 工具定义中的 prompt() 方法 |
| `src/utils/permissions/filesystem.ts` | 类似导入 | 文件系统权限提示词 |

## 依赖与外部交互

### 运行时依赖

```typescript
import { feature } from 'bun:bundle'
import { prependBullets } from '../../constants/prompts.js'
import { getAttributionTexts } from '../../utils/attribution.js'
import { hasEmbeddedSearchTools } from '../../utils/embeddedTools.js'
import { isEnvTruthy } from '../../utils/envUtils.js'
import { shouldIncludeGitInstructions } from '../../utils/gitSettings.js'
import { getClaudeTempDir } from '../../utils/permissions/filesystem.js'
import { SandboxManager } from '../../utils/sandbox/sandbox-adapter.js'
import { jsonStringify } from '../../utils/slowOperations.js'
import { getDefaultBashTimeoutMs, getMaxTimeoutMs } from '../../utils/timeouts.js'
import { getUndercoverInstructions, isUndercover } from '../../utils/undercover.js'
import { AGENT_TOOL_NAME } from '../AgentTool/constants.js'
import { FILE_EDIT_TOOL_NAME } from '../FileEditTool/constants.js'
import { FILE_READ_TOOL_NAME } from '../FileReadTool/prompt.js'
import { FILE_WRITE_TOOL_NAME } from '../FileWriteTool/prompt.js'
import { GLOB_TOOL_NAME } from '../GlobTool/prompt.js'
import { GREP_TOOL_NAME } from '../GrepTool/prompt.js'
import { TodoWriteTool } from '../TodoWriteTool/TodoWriteTool.js'
import { BASH_TOOL_NAME } from './toolName.js'
```

### 数据流

```
系统初始化
    ↓
BashTool.prompt() 调用 getSimplePrompt()
    ↓
收集各种配置信息
    ↓
┌──────────────┬──────────────┬──────────────┐
↓              ↓              ↓              ↓
工具偏好      沙盒配置       Git 指南       详细指令
    ↓              ↓              ↓              ↓
合并为完整提示词字符串
    ↓
返回给模型作为系统提示
```

## 风险、边界与改进建议

### 已知风险

1. **提示词注入风险**
   - 用户控制的配置（如沙盒路径）被包含在提示词中
   - 如果配置包含恶意内容，可能影响模型行为
   - **缓解**：配置值经过验证和清理

2. **提示词长度膨胀**
   - Git 指南非常详细（约 5KB）
   - 沙盒配置可能包含大量路径
   - **缓解**：使用 `dedup` 去重，使用 `$TMPDIR` 替换 UID 特定路径

3. **用户类型检测依赖环境变量**
   ```typescript
   process.env.USER_TYPE === 'ant'
   ```
   - 依赖运行时环境变量
   - 可能被错误设置

### 边界情况

| 场景 | 行为 |
|------|------|
| 沙盒禁用 | `getSimpleSandboxSection()` 返回空字符串 |
| Git 指令禁用 | `getCommitAndPRInstructions()` 可能只返回 undercover 部分 |
| 后台任务禁用 | `getBackgroundUsageNote()` 返回 null，相关提示被省略 |
| 简化模式 | Ant 用户的技能部分被省略 |
| 非 Undercover | Ant 用户的 undercover 部分被省略 |

### 改进建议

1. **提示词模块化**
   ```typescript
   // 建议：将提示词拆分为可配置模块
   interface PromptConfig {
     includeGitInstructions: boolean
     includeSandboxSection: boolean
     includeBackgroundTasks: boolean
     maxPromptLength: number
   }
   
   export function getSimplePrompt(config: PromptConfig): string
   ```

2. **动态提示词长度控制**
   ```typescript
   // 建议：根据上下文长度动态调整提示词
   function truncatePrompt(prompt: string, maxTokens: number): string {
     // 优先保留核心安全指令
     // 可省略详细示例和说明
   }
   ```

3. **提示词缓存优化**
   ```typescript
   // 建议：缓存提示词计算结果
   const promptCache = new Map<string, string>()
   
   export function getSimplePrompt(): string {
     const cacheKey = computeConfigHash()
     if (promptCache.has(cacheKey)) {
       return promptCache.get(cacheKey)!
     }
     // ... 生成提示词
     promptCache.set(cacheKey, prompt)
     return prompt
   }
   ```

4. **国际化支持**
   ```typescript
   // 建议：支持多语言提示词
   import { getLocale } from '../../utils/locale.js'
   
   const prompts = {
     en: { ... },
     zh: { ... },
     ja: { ... }
   }
   
   export function getSimplePrompt(): string {
     const locale = getLocale()
     return prompts[locale] || prompts.en
   }
   ```

5. **提示词 A/B 测试框架**
   ```typescript
   // 建议：支持提示词变体测试
   import { getFeatureValue } from '../../services/analytics/growthbook.js'
   
   export function getSimplePrompt(): string {
     const variant = getFeatureValue('bash_prompt_variant')
     switch (variant) {
       case 'concise': return getConcisePrompt()
       case 'detailed': return getDetailedPrompt()
       default: return getDefaultPrompt()
     }
   }
   ```

### 测试建议

```typescript
describe('prompt.ts', () => {
  it('includes tool preference items', () => {
    const prompt = getSimplePrompt()
    expect(prompt).toContain('File search: Use GlobTool')
    expect(prompt).toContain('Read files: Use FileReadTool')
  })
  
  it('includes timeout information', () => {
    const prompt = getSimplePrompt()
    expect(prompt).toContain(`${getDefaultTimeoutMs()}`)
    expect(prompt).toContain(`${getMaxTimeoutMs()}`)
  })
  
  it('includes sandbox section when enabled', () => {
    // 模拟沙盒启用
    jest.spyOn(SandboxManager, 'isSandboxingEnabled').mockReturnValue(true)
    const prompt = getSimplePrompt()
    expect(prompt).toContain('Command sandbox')
  })
  
  it('excludes sandbox section when disabled', () => {
    jest.spyOn(SandboxManager, 'isSandboxingEnabled').mockReturnValue(false)
    const prompt = getSimplePrompt()
    expect(prompt).not.toContain('Command sandbox')
  })
  
  it('normalizes TMPDIR in sandbox paths', () => {
    // 验证 UID 特定路径被替换
  })
})
```

### 架构评估

| 维度 | 评分 | 说明 |
|------|------|------|
| 可维护性 | ⭐⭐⭐ | 提示词硬编码，修改需发版 |
| 可配置性 | ⭐⭐⭐ | 依赖环境变量和全局配置 |
| 性能 | ⭐⭐⭐⭐ | 有去重优化，但无缓存 |
| 可读性 | ⭐⭐⭐⭐ | 结构清晰，但内容冗长 |
| 安全性 | ⭐⭐⭐⭐ | 用户输入经过处理 |

**总体评价**：该模块是系统提示词的核心组成部分，内容全面但较为冗长。建议引入提示词缓存和模块化配置，以提升性能和可维护性。
