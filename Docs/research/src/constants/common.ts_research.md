# common.ts 深度研究文档

## 场景与职责

`common.ts` 是 Claude Code CLI 中提供通用日期和时间工具函数的核心模块。它专注于解决跨时区日期处理的复杂性，特别是确保系统提示词（system prompt）中的日期信息稳定且可缓存。

### 核心使用场景
1. **系统提示词日期**：为 AI 模型提供当前日期上下文，支持时间敏感的任务
2. **Prompt 缓存稳定性**：通过 memoization 确保日期在会话期间保持一致，避免午夜时分缓存失效
3. **多格式日期输出**：支持 ISO 格式（用于系统提示词）和本地化月份年份格式（用于工具提示词）
4. **测试覆盖**：通过环境变量支持日期覆盖，便于测试特定日期场景

---

## 功能点目的

### 1. 本地 ISO 日期获取 (`getLocalISODate`)

**功能**：获取用户本地时区的当前日期，格式为 `YYYY-MM-DD`。

**特殊支持**：
- 环境变量 `CLAUDE_CODE_OVERRIDE_DATE` 可覆盖日期（仅内部 ant 用户使用）
- 使用本地时间而非 UTC，确保与用户感知一致

**使用场景**：
- 系统提示词中的日期信息
- 需要准确日期的任务规划

### 2. 会话起始日期 (`getSessionStartDate`)

**功能**：返回会话开始时的日期，使用 lodash 的 `memoize` 缓存。

**设计原理**：
```
问题：如果每次请求都调用 getLocalISODate()，午夜时分会发生什么？
- 请求1 (23:59): 日期 = 2026-04-01
- 请求2 (00:01): 日期 = 2026-04-02
- 系统提示词变化 → Prompt 缓存失效 → 性能下降 + 成本增加

解决方案：memoize 缓存日期，整个会话使用固定日期
```

**两种使用模式**：
1. **交互模式**：通过 `memoize(getUserContext)` 在 `context.ts` 中隐式获得
2. **简单模式** (`--bare`)：显式调用 `getSessionStartDate()` 避免缓存失效

**边界处理**：
- 午夜后日期变化通过 `getDateChangeAttachments` 在对话尾部追加新日期
- 简单模式禁用附件，选择接受"午夜后的旧日期"而非"缓存失效"

### 3. 本地月份年份 (`getLocalMonthYear`)

**功能**：返回用户本地时区的 "Month YYYY" 格式（如 "February 2026"）。

**设计原理**：
- 按月变化而非按日变化
- 用于工具提示词，最小化缓存破坏

---

## 具体技术实现

### 数据结构

```typescript
// 函数类型
export function getLocalISODate(): string
export const getSessionStartDate: () => string  // memoized
export function getLocalMonthYear(): string
```

### 实现细节

#### `getLocalISODate`
```typescript
export function getLocalISODate(): string {
  // 1. 检查 ant-only 日期覆盖
  if (process.env.CLAUDE_CODE_OVERRIDE_DATE) {
    return process.env.CLAUDE_CODE_OVERRIDE_DATE
  }

  // 2. 获取本地时间组件
  const now = new Date()
  const year = now.getFullYear()
  const month = String(now.getMonth() + 1).padStart(2, '0')
  const day = String(now.getDate()).padStart(2, '0')
  
  // 3. 返回 ISO 格式
  return `${year}-${month}-${day}`
}
```

#### `getSessionStartDate`
```typescript
// 使用 lodash memoize，首次调用后返回缓存值
export const getSessionStartDate = memoize(getLocalISODate)

// 清除缓存（如需）
getSessionStartDate.cache?.clear?.()
```

#### `getLocalMonthYear`
```typescript
export function getLocalMonthYear(): string {
  // 支持日期覆盖
  const date = process.env.CLAUDE_CODE_OVERRIDE_DATE
    ? new Date(process.env.CLAUDE_CODE_OVERRIDE_DATE)
    : new Date()
  
  // 使用美式英语格式
  return date.toLocaleString('en-US', { month: 'long', year: 'numeric' })
}
```

### 关键代码路径

#### 1. 系统提示词构造路径

```
构造系统提示词
    ↓
src/constants/prompts.ts 的 getSystemPrompt()
    ↓
调用 getSessionStartDate() 获取缓存日期
    ↓
注入到环境信息中: "Date: 2026-04-01"
    ↓
作为静态内容（cacheScope: 'global'）缓存
```

**关键文件引用**：
- `src/constants/prompts.ts`: 系统提示词生成，使用 `getSessionStartDate`
- `src/context.ts`: 用户上下文，使用 `getSessionStartDate`

#### 2. 附件日期变更路径

```
午夜后新请求
    ↓
src/utils/attachments.ts
    ↓
检测到日期变化
    ↓
添加日期变更附件到对话尾部（而非修改系统提示词）
    ↓
保持系统提示词缓存不变
```

**关键文件引用**：
- `src/utils/attachments.ts`: 附件管理，使用 `getLocalISODate`

#### 3. 缓存清除路径

```
用户执行 /clear 命令
    ↓
src/commands/clear/caches.ts
    ↓
清除各种缓存，包括日期缓存
    ↓
下次请求获取新日期
```

**关键文件引用**：
- `src/commands/clear/caches.ts`: 缓存清除命令

#### 4. Web 搜索工具路径

```
使用 Web 搜索工具
    ↓
src/tools/WebSearchTool/prompt.ts
    ↓
使用 getLocalMonthYear() 获取 "February 2026"
    ↓
注入到工具提示词中
```

**关键文件引用**：
- `src/tools/WebSearchTool/prompt.ts`: Web 搜索提示词

---

## 依赖与外部交互

### 内部依赖

| 导入 | 用途 |
|------|------|
| `lodash-es/memoize.js` | 日期缓存实现 |

### 被依赖方

| 文件 | 使用的函数 | 用途 |
|------|-----------|------|
| `src/constants/prompts.ts` | `getSessionStartDate` | 系统提示词日期 |
| `src/context.ts` | `getSessionStartDate` | 用户上下文 |
| `src/utils/attachments.ts` | `getLocalISODate` | 日期变更附件 |
| `src/commands/clear/caches.ts` | `getSessionStartDate` | 缓存清除 |
| `src/tools/WebSearchTool/prompt.ts` | `getLocalMonthYear` | 工具提示词 |

### 环境变量交互

| 变量 | 用途 | 适用用户 |
|------|------|---------|
| `CLAUDE_CODE_OVERRIDE_DATE` | 覆盖当前日期 | 仅 ant 用户 |

---

## 风险、边界与改进建议

### 当前风险

1. **时区歧义**
   - `getLocalISODate` 使用系统本地时区
   - 远程会话或容器环境中可能与用户实际时区不一致

2. **日期覆盖安全风险**
   - `CLAUDE_CODE_OVERRIDE_DATE` 没有验证格式
   - 无效日期可能导致下游解析错误

3. **Memoize 缓存泄漏**
   - `getSessionStartDate` 使用全局 memoize 缓存
   - 长时间运行的进程可能累积缓存（虽然日期函数缓存很小）

4. **午夜边界用户体验**
   - 简单模式 (`--bare`) 选择接受"旧日期"而非缓存失效
   - 跨午夜的长任务可能报告错误的日期

### 边界情况

| 场景 | 行为 |
|------|------|
| 会话跨越午夜 | 系统提示词保持旧日期，附件报告新日期 |
| 时区变更 | 会话期间不响应系统时区变化 |
| 无效覆盖日期 | 原样返回，下游组件可能解析失败 |
| 缓存清除后 | 重新获取当前日期，可能与会话开始日期不同 |
| 夏令时切换 | 依赖系统时间处理，可能有 1 小时偏差 |

### 改进建议

1. **日期格式验证**
   ```typescript
   export function getLocalISODate(): string {
     if (process.env.CLAUDE_CODE_OVERRIDE_DATE) {
       const date = process.env.CLAUDE_CODE_OVERRIDE_DATE
       if (!/^\d{4}-\d{2}-\d{2}$/.test(date)) {
         console.warn(`Invalid CLAUDE_CODE_OVERRIDE_DATE: ${date}`)
         // 回退到实际日期
       } else {
         return date
       }
     }
     // ...
   }
   ```

2. **时区感知增强**
   ```typescript
   // 建议添加时区信息
   export function getLocalISODateWithTz(): { date: string; timezone: string } {
     return {
       date: getLocalISODate(),
       timezone: Intl.DateTimeFormat().resolvedOptions().timeZone
     }
   }
   ```

3. **会话日期持久化**
   ```typescript
   // 建议将会话起始日期持久化到会话存储
   // 避免缓存清除后日期"跳跃"
   export function getPersistentSessionDate(): string {
     // 优先从会话存储读取，不存在时计算并存储
   }
   ```

4. **自动化测试覆盖**
   - 模拟不同时区的测试用例
   - 模拟午夜边界条件
   - 验证缓存行为

### 与相关模块的关系

```
common.ts (日期工具)
    ↓ 被使用
constants/prompts.ts (系统提示词)
    ↓ 调用
utils/api.ts (API 请求)
    ↓ 影响
services/api/claude.ts (Claude API)
```

日期稳定性是整个 prompt 缓存策略的关键环节，修改需谨慎评估对缓存命中率的影响。
