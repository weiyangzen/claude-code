# MCP 名称规范化 (normalization.ts) 深度研究

## 1. 场景与职责

### 1.1 核心定位
`normalization.ts` 是 Claude Code MCP 系统的**名称规范化工具**，提供纯函数用于将 MCP 服务器名称转换为符合 API 要求的格式：
- **字符过滤**：移除不符合 `^[a-zA-Z0-9_-]{1,64}$` 模式的字符
- **Claude.ai 特殊处理**：对 Claude.ai 服务器名称进行额外规范化
- **无依赖设计**：零外部依赖，避免循环引用
- **API 兼容**：确保名称可用于 API 调用和工具名构建

### 1.2 业务场景
| 场景 | 说明 |
|------|------|
| 服务器注册 | 将用户输入的服务器名规范化为 API 安全格式 |
| 工具名构建 | 生成 `mcp__server__tool` 格式的规范工具名 |
| Claude.ai 连接器 | 处理 Claude.ai 服务器名称的特殊规范化 |
| 配置验证 | 确保配置中的服务器名符合要求 |

### 1.3 设计原则
- **零依赖**：纯工具函数，无外部导入
- **幂等性**：多次规范化结果一致
- **向后兼容**：Claude.ai 服务器特殊处理保持兼容性
- **简单可预测**：替换规则明确，易于理解

---

## 2. 功能点目的

### 2.1 API 兼容名称生成
**目的**：确保 MCP 服务器名称符合 API 要求的 `^[a-zA-Z0-9_-]{1,64}$` 模式。

**合法字符**：
- 字母：`a-z`, `A-Z`
- 数字：`0-9`
- 下划线：`_`
- 连字符：`-`

### 2.2 Claude.ai 服务器特殊处理
**目的**：处理 Claude.ai 连接器服务器名称的特殊格式。

**特殊规则**：
- 合并连续下划线（`___` → `_`）
- 去除首尾下划线
- 防止干扰 `__` 分隔符

### 2.3 工具名分隔符保护
**目的**：确保规范化后的名称不会干扰 `mcp__server__tool` 格式中的双下划线分隔符。

---

## 3. 具体技术实现

### 3.1 核心常量

```typescript
// Claude.ai server names are prefixed with this string
const CLAUDEAI_SERVER_PREFIX = 'claude.ai '
```

### 3.2 规范化函数

```typescript
/**
 * Normalize server names to be compatible with the API pattern ^[a-zA-Z0-9_-]{1,64}$
 * Replaces any invalid characters (including dots and spaces) with underscores.
 *
 * For claude.ai servers (names starting with "claude.ai "), also collapses
 * consecutive underscores and strips leading/trailing underscores to prevent
 * interference with the __ delimiter used in MCP tool names.
 */
export function normalizeNameForMCP(name: string): string {
  // Step 1: 替换所有非法字符为下划线
  let normalized = name.replace(/[^a-zA-Z0-9_-]/g, '_')
  
  // Step 2: Claude.ai 服务器特殊处理
  if (name.startsWith(CLAUDEAI_SERVER_PREFIX)) {
    normalized = normalized
      .replace(/_+/g, '_')     // 合并连续下划线
      .replace(/^_|_$/g, '')   // 去除首尾下划线
  }
  
  return normalized
}
```

### 3.3 实现细节分析

#### 3.3.1 正则表达式
```typescript
/[^a-zA-Z0-9_-]/g
```
- `[^...]`：否定字符类，匹配不在集合中的字符
- `a-zA-Z0-9_-`：允许的字符（字母、数字、下划线、连字符）
- `g`：全局标志，替换所有匹配

#### 3.3.2 Claude.ai 特殊处理
```typescript
if (name.startsWith(CLAUDEAI_SERVER_PREFIX)) {
  normalized = normalized
    .replace(/_+/g, '_')      // 一个或多个下划线替换为单个
    .replace(/^_|_$/g, '')    // 开头或结尾的下划线替换为空
}
```

**处理示例**：
```typescript
// 原始名称
'claude.ai My Server'

// Step 1: 替换非法字符
'claude_ai_My_Server'

// Step 2: 合并连续下划线
'claude_ai_My_Server'（无变化）

// Step 3: 去除首尾下划线
'claude_ai_My_Server'（无变化）
```

### 3.4 使用示例

#### 3.4.1 基本规范化
```typescript
normalizeNameForMCP('my-server')        // → 'my-server'（已是合法）
normalizeNameForMCP('my_server')        // → 'my_server'（已是合法）
normalizeNameForMCP('my.server')        // → 'my_server'
normalizeNameForMCP('my server')        // → 'my_server'
normalizeNameForMCP('my:server')        // → 'my_server'
normalizeNameForMCP('my/server')        // → 'my_server'
```

#### 3.4.2 Claude.ai 服务器
```typescript
// 原始名称包含前缀
normalizeNameForMCP('claude.ai Slack')
// Step 1: 'claude_ai_Slack'
// Step 2: 'claude_ai_Slack'（无连续下划线）
// Result: 'claude_ai_Slack'

normalizeNameForMCP('claude.ai My   Server')
// Step 1: 'claude_ai_My___Server'
// Step 2: 'claude_ai_My_Server'（合并连续下划线）
// Result: 'claude_ai_My_Server'

normalizeNameForMCP('claude.ai  Server ')
// Step 1: 'claude_ai__Server_'
// Step 2: 'claude_ai_Server'（合并 + 去首尾）
// Result: 'claude_ai_Server'
```

#### 3.4.3 边界情况
```typescript
normalizeNameForMCP('')                 // → ''（空字符串）
normalizeNameForMCP('a')                // → 'a'（单字符）
normalizeNameForMCP('___')              // → '___'（非 Claude.ai）
normalizeNameForMCP('claude.ai ___')    // → 'claude_ai'（Claude.ai 特殊处理）
normalizeNameForMCP('123-server_name')  // → '123-server_name'（数字开头合法）
```

---

## 4. 关键代码路径与文件引用

### 4.1 核心导出

| 导出 | 行号 | 用途 |
|------|------|------|
| `normalizeNameForMCP` | 17-22 | 名称规范化函数 |

### 4.2 依赖

| 依赖 | 说明 |
|------|------|
| 无 | 零外部依赖 |

### 4.3 调用方

| 文件 | 用途 |
|------|------|
| `mcpStringUtils.ts` | 构建 MCP 工具名前缀 |
| `utils.ts` | 过滤工具、命令、资源 |
| `config.ts` | 服务器名验证 |
| `../../utils/settings/permissionValidation.ts` | 权限规则验证 |
| `../../utils/claudeInChrome/common.ts` | 服务器名检查 |
| `../../utils/computerUse/common.ts` | 服务器名检查 |

---

## 5. 依赖与外部交互

### 5.1 零依赖设计

```typescript
/**
 * Pure utility functions for MCP name normalization.
 * This file has no dependencies to avoid circular imports.
 */
```

**设计原因**：
- 被多个基础模块依赖
- 避免循环引用风险
- 确保始终可用

### 5.2 API 模式要求

```
Pattern: ^[a-zA-Z0-9_-]{1,64}$

^          - 字符串开头
[a-zA-Z0-9_-] - 允许的字符类
{1,64}     - 长度限制 1-64
$          - 字符串结尾
```

**注意**：当前实现不检查长度限制，由调用方处理。

### 5.3 Claude.ai 前缀

```typescript
const CLAUDEAI_SERVER_PREFIX = 'claude.ai '
// 注意：末尾有空格
```

**典型 Claude.ai 服务器名**：
- `claude.ai Slack`
- `claude.ai GitHub`
- `claude.ai Linear`

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 功能边界
| 边界 | 说明 |
|------|------|
| 长度限制 | 不检查 64 字符限制 |
| 空字符串 | 返回空字符串，可能导致问题 |
| 全非法字符 | 可能返回全下划线字符串 |
| 大小写保留 | 保留原始大小写，可能导致不一致 |

#### 6.1.2 潜在冲突
```typescript
// 不同原始名称可能规范化后相同
normalizeNameForMCP('my.server')  // → 'my_server'
normalizeNameForMCP('my server')  // → 'my_server'
normalizeNameForMCP('my:server')  // → 'my_server'
```

### 6.2 潜在问题

#### 6.2.1 长度溢出
```typescript
// 64 字符限制未检查
const longName = 'a'.repeat(100)
normalizeNameForMCP(longName)  // → 100 个 'a'，超出限制
```

#### 6.2.2 空结果
```typescript
// 全非法字符且是 Claude.ai 服务器
normalizeNameForMCP('claude.ai ...')
// Step 1: 'claude_ai___'
// Step 2: 'claude_ai'（去除尾部下划线后）
// 结果有效，但可能非预期

// 极端情况
normalizeNameForMCP('claude.ai ... ')
// Step 1: 'claude_ai____'
// Step 2: 'claude_ai'（合并 + 去首尾）
```

### 6.3 改进建议

#### 6.3.1 功能增强
1. **长度验证**：
   ```typescript
   export function normalizeNameForMCP(name: string): string {
     let normalized = name.replace(/[^a-zA-Z0-9_-]/g, '_')
     if (name.startsWith(CLAUDEAI_SERVER_PREFIX)) {
       normalized = normalized.replace(/_+/g, '_').replace(/^_|_$/g, '')
     }
     // 截断到 64 字符
     return normalized.slice(0, 64)
   }
   ```

2. **空结果处理**：
   ```typescript
   export function normalizeNameForMCP(name: string): string | null {
     let normalized = name.replace(/[^a-zA-Z0-9_-]/g, '_')
     if (name.startsWith(CLAUDEAI_SERVER_PREFIX)) {
       normalized = normalized.replace(/_+/g, '_').replace(/^_|_$/g, '')
     }
     return normalized.length > 0 ? normalized : null
   }
   ```

3. **冲突检测**：
   ```typescript
   export function createNormalizationMap(names: string[]): Map<string, string[]> {
     const map = new Map<string, string[]>()
     for (const name of names) {
       const normalized = normalizeNameForMCP(name)
       const conflicts = map.get(normalized) || []
       conflicts.push(name)
       map.set(normalized, conflicts)
     }
     return map
   }
   ```

#### 6.3.2 性能优化
1. **正则预编译**：
   ```typescript
   const INVALID_CHAR_REGEX = /[^a-zA-Z0-9_-]/g
   const MULTI_UNDERSCORE_REGEX = /_+/g
   const LEADING_TRAILING_UNDERSCORE_REGEX = /^_|_$/g
   
   export function normalizeNameForMCP(name: string): string {
     let normalized = name.replace(INVALID_CHAR_REGEX, '_')
     if (name.startsWith(CLAUDEAI_SERVER_PREFIX)) {
       normalized = normalized
         .replace(MULTI_UNDERSCORE_REGEX, '_')
         .replace(LEADING_TRAILING_UNDERSCORE_REGEX, '')
     }
     return normalized
   }
   ```

2. **前缀缓存**：
   ```typescript
   const isClaudeAiCache = new Map<string, boolean>()
   
   export function normalizeNameForMCP(name: string): string {
     let isClaudeAi = isClaudeAiCache.get(name)
     if (isClaudeAi === undefined) {
       isClaudeAi = name.startsWith(CLAUDEAI_SERVER_PREFIX)
       isClaudeAiCache.set(name, isClaudeAi)
     }
     // ...
   }
   ```

#### 6.3.3 代码质量
1. **单元测试**：
   ```typescript
   describe('normalizeNameForMCP', () => {
     it('preserves valid characters', () => {
       expect(normalizeNameForMCP('abc-ABC_123')).toBe('abc-ABC_123')
     })
     
     it('replaces invalid characters', () => {
       expect(normalizeNameForMCP('my.server')).toBe('my_server')
       expect(normalizeNameForMCP('my server')).toBe('my_server')
     })
     
     it('handles Claude.ai servers specially', () => {
       expect(normalizeNameForMCP('claude.ai My   Server'))
         .toBe('claude_ai_My_Server')
     })
     
     it('handles empty string', () => {
       expect(normalizeNameForMCP('')).toBe('')
     })
   })
   ```

2. **文档完善**：
   ```typescript
   /**
    * Normalize server names to be compatible with the API pattern ^[a-zA-Z0-9_-]{1,64}$
    * 
    * @param name - The server name to normalize
    * @returns The normalized name (may be empty string)
    * 
    * @example
    * normalizeNameForMCP('my server') // 'my_server'
    * normalizeNameForMCP('claude.ai Slack') // 'claude_ai_Slack'
    */
   ```

### 6.4 测试建议

| 测试场景 | 优先级 |
|----------|--------|
| 合法字符保留 | 高 |
| 非法字符替换 | 高 |
| Claude.ai 特殊处理 | 高 |
| 空字符串 | 中 |
| 长字符串 | 中 |
| 全非法字符 | 中 |
| 性能测试（高频调用） | 低 |
