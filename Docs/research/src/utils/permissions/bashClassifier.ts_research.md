# bashClassifier.ts 深度研究

## 场景与职责

`bashClassifier.ts` 是 Claude Code 权限系统的**存根模块 (Stub Module)**，用于外部构建。该文件实现了 Bash 命令分类器接口，但所有功能都被禁用，返回默认值。这是因为在非 Ant 构建中，分类器权限功能是禁用的（ANT-ONLY 功能）。

### 核心职责
1. **接口兼容**: 提供与 Ant 构建中完整分类器相同的接口
2. **功能禁用**: 所有分类功能返回"禁用"状态
3. **Tree-shaking 友好**: 代码结构支持构建时的死代码消除

### 设计背景
Bash 分类器是自动模式的一部分，用于语义匹配 Bash 命令与用户定义的描述规则。由于该功能依赖 Ant 内部基础设施（如特定的 AI 模型、提示词等），在外部构建中不可用。

---

## 功能点目的

### 1. 提示前缀常量 (`PROMPT_PREFIX`)
```typescript
export const PROMPT_PREFIX = 'prompt:'
```
用于标识提示类型的规则内容。

### 2. 分类器结果类型 (`ClassifierResult`)
```typescript
export type ClassifierResult = {
  matches: boolean           // 是否匹配
  matchedDescription?: string // 匹配的描述
  confidence: 'high' | 'medium' | 'low'  // 置信度
  reason: string             // 原因说明
}
```

### 3. 分类器行为类型 (`ClassifierBehavior`)
```typescript
export type ClassifierBehavior = 'deny' | 'ask' | 'allow'
```
定义分类器可以建议的三种行为。

### 4. 提示描述提取 (`extractPromptDescription`)
```typescript
export function extractPromptDescription(_ruleContent: string | undefined): string | null {
  return null  // 始终返回 null（禁用）
}
```

### 5. 提示规则内容创建 (`createPromptRuleContent`)
```typescript
export function createPromptRuleContent(description: string): string {
  return `${PROMPT_PREFIX} ${description.trim()}`
}
```
创建格式化的提示规则内容。

### 6. 功能启用检查 (`isClassifierPermissionsEnabled`)
```typescript
export function isClassifierPermissionsEnabled(): boolean {
  return false  // 始终返回 false
}
```

### 7. Bash 提示描述获取
三个函数始终返回空数组：
- `getBashPromptDenyDescriptions(context)`
- `getBashPromptAskDescriptions(context)`
- `getBashPromptAllowDescriptions(context)`

### 8. Bash 命令分类 (`classifyBashCommand`)
```typescript
export async function classifyBashCommand(
  _command: string,
  _cwd: string,
  _descriptions: string[],
  _behavior: ClassifierBehavior,
  _signal: AbortSignal,
  _isNonInteractiveSession: boolean,
): Promise<ClassifierResult> {
  return {
    matches: false,
    confidence: 'high',
    reason: 'This feature is disabled',
  }
}
```

### 9. 通用描述生成 (`generateGenericDescription`)
```typescript
export async function generateGenericDescription(
  _command: string,
  specificDescription: string | undefined,
  _signal: AbortSignal,
): Promise<string | null> {
  return specificDescription || null
}
```

---

## 具体技术实现

### 存根模式实现

所有函数都遵循存根模式：
1. **输入参数**: 使用下划线前缀表示未使用（`_paramName`）
2. **返回值**: 返回安全默认值
3. **异步函数**: 返回立即解析的 Promise

```typescript
// 同步函数 - 返回静态值
export function isClassifierPermissionsEnabled(): boolean {
  return false
}

// 异步函数 - 返回立即解析的 Promise
export async function classifyBashCommand(...): Promise<ClassifierResult> {
  return {
    matches: false,
    confidence: 'high',
    reason: 'This feature is disabled',
  }
}
```

### 类型导出

即使功能被禁用，类型定义仍然完整导出：
```typescript
export type ClassifierResult = { /* ... */ }
export type ClassifierBehavior = 'deny' | 'ask' | 'allow'
```

这确保：
1. 类型检查通过
2. 接口一致性
3. 未来启用功能时代码兼容

---

## 关键代码路径与文件引用

### 调用方

| 调用方 | 路径 | 场景 |
|--------|------|------|
| 权限检查逻辑 | 多个位置 | 检查分类器是否启用 |
| Bash 工具 | `src/tools/BashTool/` | Bash 命令分类（如启用） |

### 与 Ant 构建的关系

在 Ant 内部构建中，此文件被完整实现版本替换，提供：
- 实际的 AI 分类器调用
- 提示词模板
- 结果缓存
- 遥测集成

---

## 依赖与外部交互

### 模块依赖

```
bashClassifier.ts
    ← 无依赖（纯存根模块）
```

### 外部使用模式

```typescript
import { isClassifierPermissionsEnabled, classifyBashCommand } from './bashClassifier.js'

// 检查功能是否启用
if (isClassifierPermissionsEnabled()) {
  // 在 Ant 构建中，这会进入实际分类逻辑
  const result = await classifyBashCommand(command, cwd, descriptions, behavior, signal, false)
  // ...
} else {
  // 在外部构建中，跳过分类
  console.log('Bash classifier is disabled')
}
```

---

## 风险、边界与改进建议

### 当前风险

1. **功能差异**:
   - Ant 构建和外部构建在此功能上有显著差异
   - 可能导致用户在不同构建间体验不一致

2. **错误消息**:
   - `classifyBashCommand` 返回 `"This feature is disabled"`
   - 这可能暴露内部实现细节

3. **类型与实际行为不匹配**:
   - 类型系统显示功能可用，但运行时始终禁用
   - 可能导致开发者困惑

### 边界情况

1. **空描述处理**:
   ```typescript
   createPromptRuleContent('') // 返回 "prompt:"
   ```

2. **信号处理**:
   ```typescript
   // AbortSignal 被忽略
   const controller = new AbortController()
   controller.abort()
   await classifyBashCommand('ls', '/', [], 'allow', controller.signal, false)
   // 仍然返回结果，不会抛出 AbortError
   ```

### 改进建议

1. **功能标志集成**:
   ```typescript
   import { feature } from 'bun:bundle'
   
   export function isClassifierPermissionsEnabled(): boolean {
     return feature('BASH_CLASSIFIER')
   }
   ```

2. **更清晰的错误消息**:
   ```typescript
   export async function classifyBashCommand(...): Promise<ClassifierResult> {
     return {
       matches: false,
       confidence: 'high',
       reason: 'Bash classifier requires Anthropic internal infrastructure',
     }
   }
   ```

3. **编译时排除**:
   ```typescript
   // 使用条件编译完全排除此模块
   if (process.env.USER_TYPE !== 'ant') {
     // 外部构建：导出空实现
     module.exports = {
       isClassifierPermissionsEnabled: () => false,
       classifyBashCommand: async () => ({ matches: false, confidence: 'high', reason: 'Disabled' }),
       // ...
     }
   }
   ```

4. **文档化**:
   ```typescript
   /**
    * @file Bash command classifier stub for external builds.
    * 
    * This module provides a stub implementation of the Bash classifier
    * for external builds. The full implementation is only available
    * in Anthropic internal builds.
    * 
    * All functions return safe default values:
    * - isClassifierPermissionsEnabled() → false
    * - classifyBashCommand() → { matches: false, reason: 'Disabled' }
    * - All getters → []
    */
   ```

5. **类型区分**:
   ```typescript
   // 考虑使用不同的类型来表示禁用状态
   export type ClassifierResult =
     | { type: 'match'; matches: true; matchedDescription: string; confidence: 'high' | 'medium' | 'low'; reason: string }
     | { type: 'noMatch'; matches: false; confidence: 'high'; reason: string }
     | { type: 'disabled'; matches: false; confidence: 'high'; reason: 'Feature disabled' }
   ```

### 测试建议

1. **存根行为测试**:
   ```typescript
   describe('bashClassifier stub', () => {
     it('always reports disabled', () => {
       expect(isClassifierPermissionsEnabled()).toBe(false)
     })
     
     it('returns disabled result', async () => {
       const result = await classifyBashCommand('ls', '/', [], 'allow', new AbortController().signal, false)
       expect(result.matches).toBe(false)
       expect(result.reason).toContain('disabled')
     })
   })
   ```

2. **集成测试**:
   - 验证调用方正确处理 `isClassifierPermissionsEnabled() === false`
   - 验证禁用状态下的降级行为
