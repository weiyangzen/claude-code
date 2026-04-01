# MCP 环境变量展开 (envExpansion.ts) 深度研究

## 1. 场景与职责

### 1.1 核心定位
`envExpansion.ts` 是 Claude Code MCP 系统的**环境变量处理工具**，提供轻量级的环境变量展开功能：
- **变量展开**：支持 `${VAR}` 语法展开为实际环境变量值
- **默认值支持**：支持 `${VAR:-default}` 语法提供默认值
- **缺失报告**：收集未定义变量列表用于错误报告
- **配置集成**：被 `config.ts` 用于 MCP 服务器配置的环境变量处理

### 1.2 业务场景
| 场景 | 示例 |
|------|------|
| API 密钥注入 | `"command": "my-server --key ${API_KEY}"` |
| 路径配置 | `"args": ["${HOME}/config.json"]` |
| 默认值回退 | `"url": "${MCP_URL:-https://default.example.com}"` |
| 环境特定配置 | `"env": { "STAGE": "${STAGE:-dev}" }` |

### 1.3 设计原则
- **单一职责**：纯工具函数，无业务逻辑
- **轻量依赖**：零外部依赖，避免循环引用
- **Shell 兼容**：语法兼容 POSIX shell 的 `${VAR:-default}`
- **错误透明**：保留未展开变量，由调用方决定如何处理

---

## 2. 功能点目的

### 2.1 环境变量展开
**目的**：允许在 MCP 配置中引用环境变量，实现配置与敏感信息的分离。

**支持语法**：
- `${VAR}`：展开为环境变量值
- `${VAR:-default}`：变量未定义时使用默认值

### 2.2 缺失变量检测
**目的**：帮助用户识别配置中引用但未定义的环境变量。

**实现**：返回 `missingVars` 数组，包含所有未找到的环境变量名。

### 2.3 配置安全
**目的**：避免在配置文件中硬编码敏感信息。

**安全收益**：
- API 密钥、密码等敏感信息存储在环境变量
- 配置文件可以安全地提交到版本控制
- 不同环境使用不同环境变量值

---

## 3. 具体技术实现

### 3.1 核心函数

```typescript
/**
 * Expand environment variables in a string value
 * Handles ${VAR} and ${VAR:-default} syntax
 * @returns Object with expanded string and list of missing variables
 */
export function expandEnvVarsInString(value: string): {
  expanded: string
  missingVars: string[]
} {
  const missingVars: string[] = []

  const expanded = value.replace(/\$\{([^}]+)\}/g, (match, varContent) => {
    // Split on :- to support default values (limit to 2 parts to preserve :- in defaults)
    const [varName, defaultValue] = varContent.split(':-', 2)
    const envValue = process.env[varName]

    if (envValue !== undefined) {
      return envValue
    }
    if (defaultValue !== undefined) {
      return defaultValue
    }

    // Track missing variable for error reporting
    missingVars.push(varName)
    // Return original if not found (allows debugging but will be reported as error)
    return match
  })

  return {
    expanded,
    missingVars,
  }
}
```

### 3.2 实现细节分析

#### 3.2.1 正则表达式
```typescript
/\$\{([^}]+)\}/g
```
- `\$\{`：匹配 `${` 字面量
- `([^}]+)`：捕获组，匹配一个或多个非 `}` 字符
- `\}`：匹配 `}` 字面量
- `g`：全局匹配，处理字符串中所有变量引用

#### 3.2.2 默认值解析
```typescript
const [varName, defaultValue] = varContent.split(':-', 2)
```
- 使用 `split(':-', 2)` 限制分割为 2 部分
- 保留默认值中可能包含的 `:` 字符
- 例如：`${VAR:-http://example.com}` → `['VAR', 'http://example.com']`

#### 3.2.3 缺失变量处理
```typescript
if (envValue !== undefined) {
  return envValue
}
if (defaultValue !== undefined) {
  return defaultValue
}
missingVars.push(varName)
return match  // 保留原始匹配文本
```
- 优先使用环境变量值
- 其次使用默认值
- 最后保留原始文本并记录缺失

### 3.3 使用示例

#### 3.3.1 基本展开
```typescript
// 环境变量：API_KEY=secret123
expandEnvVarsInString('Server --key ${API_KEY}')
// 返回：{ expanded: 'Server --key secret123', missingVars: [] }
```

#### 3.3.2 默认值
```typescript
// 环境变量：未设置 STAGE
expandEnvVarsInString('https://${STAGE:-dev}.example.com')
// 返回：{ expanded: 'https://dev.example.com', missingVars: [] }
```

#### 3.3.3 缺失变量
```typescript
// 环境变量：未设置 MISSING_VAR
expandEnvVarsInString('Value is ${MISSING_VAR}')
// 返回：{ expanded: 'Value is ${MISSING_VAR}', missingVars: ['MISSING_VAR'] }
```

#### 3.3.4 复杂默认值
```typescript
// 环境变量：未设置 URL
expandEnvVarsInString('${URL:-https://api.example.com:8080/v1}')
// 返回：{ expanded: 'https://api.example.com:8080/v1', missingVars: [] }
```

### 3.4 调用方集成

在 `config.ts` 中的使用：

```typescript
function expandEnvVars(config: McpServerConfig): {
  expanded: McpServerConfig
  missingVars: string[]
} {
  const missingVars: string[] = []

  function expandString(str: string): string {
    const { expanded, missingVars: vars } = expandEnvVarsInString(str)
    missingVars.push(...vars)
    return expanded
  }

  switch (config.type) {
    case 'stdio':
      return {
        expanded: {
          ...config,
          command: expandString(config.command),
          args: config.args.map(expandString),
          env: config.env ? mapValues(config.env, expandString) : undefined,
        },
        missingVars: [...new Set(missingVars)]
      }
    case 'sse':
    case 'http':
    case 'ws':
      return {
        expanded: {
          ...config,
          url: expandString(config.url),
          headers: config.headers ? mapValues(config.headers, expandString) : undefined,
        },
        missingVars: [...new Set(missingVars)]
      }
    // ... 其他类型原样返回
  }
}
```

---

## 4. 关键代码路径与文件引用

### 4.1 核心导出

| 导出 | 行号 | 用途 |
|------|------|------|
| `expandEnvVarsInString` | 10-38 | 环境变量展开函数 |

### 4.2 依赖

| 依赖 | 说明 |
|------|------|
| `process.env` | Node.js 全局环境变量对象 |
| `String.prototype.replace` | 字符串替换 |
| `String.prototype.split` | 字符串分割 |

### 4.3 调用方

| 文件 | 用途 |
|------|------|
| `config.ts` | MCP 服务器配置的环境变量展开 |

---

## 5. 依赖与外部交互

### 5.1 零外部依赖设计

```typescript
/**
 * Shared utilities for expanding environment variables in MCP server configurations
 */
```

该文件 intentionally 不导入任何外部模块：
- 避免循环依赖风险
- 保持轻量，可被任何模块安全导入
- 纯函数，易于测试

### 5.2 Node.js 环境依赖

```typescript
// 依赖 Node.js 全局对象
process.env  // 环境变量存储
```

### 5.3 调用链

```
config.ts:parseMcpConfig()
├── expandEnvVars(config)  // 遍历配置对象
│   └── expandEnvVarsInString(str)  // 展开单个字符串
│       └── process.env[varName]    // 读取环境变量
└── 收集 missingVars 生成警告
```

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 安全风险
| 风险 | 描述 | 缓解措施 |
|------|------|----------|
| 敏感信息泄露 | 展开后的值可能出现在日志中 | 调用方负责敏感信息脱敏 |
| 命令注入 | `${VAR}` 展开到命令参数中 | 配置验证时检查危险字符 |

#### 6.1.2 功能边界
| 边界 | 说明 |
|------|------|
| 仅支持 `${}` 语法 | 不支持 `$VAR` 简写形式 |
| 仅支持 `:-` 默认值 | 不支持 `:=`、`:+`、`:?` 等其他 shell 修饰符 |
| 不支持嵌套变量 | 不支持 `${VAR_${SUFFIX}}` |
| 不支持变量替换 | 不支持 `${VAR/pattern/replacement}` |

### 6.2 潜在问题

#### 6.2.1 默认值中的 `}`
```typescript
// 问题：默认值中包含 } 会导致解析错误
expandEnvVarsInString('${VAR:-a}b}')  // 无法正确解析

// 当前行为：正则匹配到第一个 }，结果为 'VAR:-a'
```

#### 6.2.2 空字符串处理
```typescript
// 环境变量：EMPTY=""
expandEnvVarsInString('${EMPTY}')
// 返回：{ expanded: '', missingVars: [] }
// 这是正确的行为：空字符串是有效的值
```

#### 6.2.3 性能考虑
```typescript
// 问题：大量变量引用时的性能
// 每个变量触发一次正则替换
// 对于 N 个变量，时间复杂度 O(N * L)，L 为字符串长度

// 当前文件大小：38 行，非性能关键路径
```

### 6.3 改进建议

#### 6.3.1 功能扩展
1. **支持 `$VAR` 简写**：
   ```typescript
   // 添加对 $VAR 的支持
   value.replace(/\$\{([^}]+)\}|\$([A-Za-z_][A-Za-z0-9_]*)/g, ...)
   ```

2. **支持更多默认值语法**：
   ```typescript
   // ${VAR:=default} - 设置默认值
   // ${VAR:+replacement} - 变量存在时替换
   // ${VAR:?error} - 缺失时报错
   ```

3. **支持嵌套展开**：
   ```typescript
   // 变量值中可能包含其他变量引用
   // API_KEY=${PREFIX}_key
   // 需要递归展开
   ```

#### 6.3.2 安全增强
1. **敏感变量标记**：
   ```typescript
   const SENSITIVE_PATTERNS = [/key/i, /secret/i, /password/i, /token/i]
   function isSensitiveVar(varName: string): boolean {
     return SENSITIVE_PATTERNS.some(p => p.test(varName))
   }
   ```

2. **展开深度限制**：
   ```typescript
   function expandEnvVarsInString(value: string, depth = 0): ... {
     if (depth > 10) throw new Error('Expansion depth exceeded')
     // ...
   }
   ```

#### 6.3.3 可观测性
1. **调试模式**：
   ```typescript
   if (process.env.CLAUDE_CODE_DEBUG_ENV_EXPANSION) {
     console.log(`[env] Expanding: ${value} -> ${expanded}`)
   }
   ```

#### 6.3.4 代码质量
1. **单元测试**：当前文件无测试，建议添加：
   ```typescript
   describe('expandEnvVarsInString', () => {
     it('expands simple variable', () => { ... })
     it('uses default value', () => { ... })
     it('reports missing variables', () => { ... })
     it('handles empty string value', () => { ... })
   })
   ```

2. **类型安全**：
   ```typescript
   // 使用 branded type 区分展开前后的字符串
type RawConfigString = string & { __brand: 'RawConfigString' }
type ExpandedConfigString = string & { __brand: 'ExpandedConfigString' }
   ```

### 6.4 测试建议

| 测试场景 | 优先级 |
|----------|--------|
| 基本变量展开 | 高 |
| 默认值语法 | 高 |
| 缺失变量检测 | 高 |
| 特殊字符处理 | 中 |
| 空字符串值 | 中 |
| 性能测试（大字符串） | 低 |
