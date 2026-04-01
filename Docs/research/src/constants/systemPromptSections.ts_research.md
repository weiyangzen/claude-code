# systemPromptSections.ts 深度研究文档

## 场景与职责

`src/constants/systemPromptSections.ts` 是 Claude Code CLI 的系统提示词片段（System Prompt Sections）管理模块，提供了一种模块化的方式来构建动态系统提示词。该模块解决了系统提示词中需要动态计算的部分（如当前工作目录、Git 状态等）与静态提示词之间的整合问题。

**主要使用场景：**
1. **动态系统提示词构建**：允许系统提示词的某些部分根据当前状态动态计算
2. **提示词缓存优化**：支持缓存不变的提示词片段，避免重复计算
3. **提示词生命周期管理**：在 `/clear` 和 `/compact` 命令后清理提示词状态

**设计目标：**
- 将系统提示词分解为可独立计算和缓存的片段
- 支持需要每轮重新计算的易变片段（volatile sections）
- 提供清晰的 API 来定义和解析提示词片段

## 功能点目的

### 1. 提示词片段定义

系统提示词被分解为多个 `SystemPromptSection`，每个片段包含：
- `name`: 片段标识名
- `compute`: 计算函数，返回片段内容（可为异步）
- `cacheBreak`: 是否打破缓存（每轮重新计算）

### 2. 缓存策略

**默认缓存策略**：
- 片段计算一次后缓存
- 缓存持续到 `/clear` 或 `/compact` 命令
- 适用于不常变化的内容（如项目结构、技能列表）

**易变片段策略**（`DANGEROUS_uncachedSystemPromptSection`）：
- 每轮对话重新计算
- 会打破提示词缓存（prompt cache）
- 需要显式说明打破缓存的原因

### 3. 生命周期管理

`clearSystemPromptSections()` 函数在以下场景被调用：
- 用户执行 `/clear` 命令
- 用户执行 `/compact` 命令
- 需要重置提示词状态的其他场景

同时会清理 Beta Header 的 latch 状态，确保新对话获得全新的功能评估。

## 具体技术实现

### 类型定义

```typescript
type ComputeFn = () => string | null | Promise<string | null>

type SystemPromptSection = {
  name: string
  compute: ComputeFn
  cacheBreak: boolean
}
```

### 核心函数

#### `systemPromptSection(name, compute)`

创建标准缓存型提示词片段：

```typescript
export function systemPromptSection(
  name: string,
  compute: ComputeFn,
): SystemPromptSection {
  return { name, compute, cacheBreak: false }
}
```

**使用示例**：
```typescript
const cwdSection = systemPromptSection('cwd', () => `Current directory: ${process.cwd()}`)
```

#### `DANGEROUS_uncachedSystemPromptSection(name, compute, _reason)`

创建易变提示词片段：

```typescript
export function DANGEROUS_uncachedSystemPromptSection(
  name: string,
  compute: ComputeFn,
  _reason: string,
): SystemPromptSection {
  return { name, compute, cacheBreak: true }
}
```

**命名意图**：
- `DANGEROUS_` 前缀警示开发者此操作会打破缓存
- `_reason` 参数强制要求文档化打破缓存的理由

#### `resolveSystemPromptSections(sections)`

解析所有提示词片段：

```typescript
export async function resolveSystemPromptSections(
  sections: SystemPromptSection[],
): Promise<(string | null)[]> {
  const cache = getSystemPromptSectionCache()

  return Promise.all(
    sections.map(async s => {
      if (!s.cacheBreak && cache.has(s.name)) {
        return cache.get(s.name) ?? null
      }
      const value = await s.compute()
      setSystemPromptSectionCacheEntry(s.name, value)
      return value
    }),
  )
}
```

**缓存逻辑**：
1. 检查片段是否标记为 `cacheBreak`
2. 检查缓存中是否已有该片段
3. 如未缓存或需要刷新，执行计算函数
4. 将结果存入缓存
5. 返回片段内容（可能为 null）

#### `clearSystemPromptSections()`

清理所有提示词片段状态：

```typescript
export function clearSystemPromptSections(): void {
  clearSystemPromptSectionState()
  clearBetaHeaderLatches()
}
```

### 缓存存储

缓存通过 `../bootstrap/state.js` 管理：

```typescript
import {
  clearBetaHeaderLatches,
  clearSystemPromptSectionState,
  getSystemPromptSectionCache,
  setSystemPromptSectionCacheEntry,
} from '../bootstrap/state.js'
```

缓存实现细节在 `bootstrap/state.ts` 中，使用模块级 Map 存储。

## 关键代码路径与文件引用

### 调用方分析

| 调用文件 | 调用内容 | 用途 |
|----------|----------|------|
| `src/services/compact/postCompactCleanup.ts` | `clearSystemPromptSections` | 压缩后清理提示词状态 |
| `src/tools/EnterWorktreeTool/EnterWorktreeTool.ts` | `clearSystemPromptSections` | 进入工作树时重置状态 |
| `src/tools/ExitWorktreeTool/ExitWorktreeTool.ts` | `clearSystemPromptSections` | 退出工作树时重置状态 |
| `src/utils/sessionRestore.ts` | `clearSystemPromptSections` | 会话恢复时清理状态 |

### 依赖模块

| 模块 | 用途 |
|------|------|
| `../bootstrap/state.js` | 缓存存储和状态管理 |

### 代码示例：在系统提示词构建中的使用

```typescript
// 假设的系统提示词构建代码
import { systemPromptSection, resolveSystemPromptSections } from './constants/systemPromptSections.js'

const sections = [
  systemPromptSection('skills', async () => {
    const skills = await loadSkills()
    return formatSkills(skills)
  }),
  systemPromptSection('cwd', () => `Current directory: ${process.cwd()}`),
  DANGEROUS_uncachedSystemPromptSection(
    'time',
    () => `Current time: ${new Date().toISOString()}`,
    'Time changes every turn and must be fresh'
  ),
]

const resolvedPrompts = await resolveSystemPromptSections(sections)
const fullSystemPrompt = resolvedPrompts.filter(Boolean).join('\n\n')
```

## 依赖与外部交互

### 外部依赖

1. **Bootstrap State 模块**
   - `getSystemPromptSectionCache()`: 获取片段缓存 Map
   - `setSystemPromptSectionCacheEntry()`: 设置缓存条目
   - `clearSystemPromptSectionState()`: 清理所有片段状态
   - `clearBetaHeaderLatches()`: 清理 Beta Header 状态

### 与 Beta Header 的关系

`clearSystemPromptSections()` 同时调用 `clearBetaHeaderLatches()`，这是因为：
- Beta Header（如 AFK 模式、快速模式、缓存编辑等）的评估状态存储在 latch 中
- 新对话需要重新评估这些功能开关
- 保持提示词片段状态与 Beta Header 状态的一致性

## 风险、边界与改进建议

### 潜在风险

1. **缓存一致性问题**
   - 如果 `cacheBreak` 标记使用不当，可能导致 stale 数据
   - 异步计算函数可能抛出异常，需要错误处理

2. **性能风险**
   - 过多的易变片段会导致每轮都重新计算大量内容
   - 计算函数如果耗时较长，会影响响应时间

3. **内存泄漏**
   - 缓存 Map 持续增长，如果片段名称动态生成可能导致内存泄漏
   - 当前实现依赖 `clearSystemPromptSections()` 显式清理

4. **并发安全**
   - `resolveSystemPromptSections` 使用 `Promise.all` 并行执行
   - 如果多个计算函数同时修改共享状态，可能导致竞态条件

### 边界情况

1. **null 返回值**：计算函数可以返回 null，会被过滤掉
2. **空片段名称**：未验证片段名称的唯一性
3. **异常处理**：计算函数抛出的异常会传播给调用方

### 改进建议

1. **添加错误处理**
   ```typescript
   export async function resolveSystemPromptSections(
     sections: SystemPromptSection[],
   ): Promise<(string | null)[]> {
     const cache = getSystemPromptSectionCache()

     return Promise.all(
       sections.map(async s => {
         try {
           if (!s.cacheBreak && cache.has(s.name)) {
             return cache.get(s.name) ?? null
           }
           const value = await s.compute()
           setSystemPromptSectionCacheEntry(s.name, value)
           return value
         } catch (error) {
           console.error(`Failed to compute system prompt section "${s.name}":`, error)
           return null
         }
       }),
     )
   }
   ```

2. **片段名称唯一性验证**
   ```typescript
   export function resolveSystemPromptSections(
     sections: SystemPromptSection[],
   ): Promise<(string | null)[]> {
     const names = sections.map(s => s.name)
     const duplicates = names.filter((name, i) => names.indexOf(name) !== i)
     if (duplicates.length > 0) {
       console.warn(`Duplicate system prompt section names: ${duplicates.join(', ')}`)
     }
     // ...
   }
   ```

3. **缓存大小限制**
   - 考虑为缓存添加大小限制或 LRU 策略
   - 防止长期运行会话的内存增长

4. **性能监控**
   - 添加计算函数执行时间的监控
   - 识别慢速片段进行优化

5. **类型安全增强**
   ```typescript
   // 为片段名称添加品牌类型，防止字符串混淆
   type SectionName = string & { __brand: 'SectionName' }
   ```

6. **文档和示例**
   - 添加更多使用示例
   - 说明何时应该使用易变片段
   - 提供缓存最佳实践指南
