# argumentSubstitution.ts 研究文档

## 场景与职责

`argumentSubstitution.ts` 提供技能（Skill）和命令提示词中的参数占位符替换功能。支持多种参数引用语法，使用 shell-quote 库进行安全的参数解析。

## 功能点目的

### 1. 参数解析 (`parseArguments`)
- 使用 shell-quote 解析参数字符串
- 支持引号字符串（单引号、双引号）
- 保留变量语法（`$VAR` 不展开）
- 失败时回退到简单空格分割

### 2. 参数名解析 (`parseArgumentNames`)
- 从 frontmatter 解析参数名列表
- 支持空格分隔的字符串或字符串数组
- 过滤无效名称（空字符串、纯数字）

### 3. 渐进式参数提示 (`generateProgressiveArgumentHint`)
- 显示剩余未填充的参数
- 格式：`[arg2] [arg3]`

### 4. 参数替换 (`substituteArguments`)
- 支持 `$ARGUMENTS` - 完整参数字符串
- 支持 `$ARGUMENTS[n]` - 索引参数
- 支持 `$n` - 简写索引（`$0`, `$1`）
- 支持 `$name` - 命名参数
- 无占位符时可选择追加参数

## 具体技术实现

### 关键数据结构

```typescript
// 参数解析结果
export type ShellParseResult =
  | { success: true; tokens: ParseEntry[] }
  | { success: false; error: string }
```

### 参数解析流程

```typescript
export function parseArguments(args: string): string[] {
  if (!args || !args.trim()) {
    return []
  }

  // 使用 shell-quote 解析，保留 $VAR 语法
  const result = tryParseShellCommand(args, key => `$${key}`)
  if (!result.success) {
    // 回退到简单空格分割
    return args.split(/\s+/).filter(Boolean)
  }

  // 只保留字符串类型的 token
  return result.tokens.filter(
    (token): token is string => typeof token === 'string',
  )
}
```

### 参数名验证

```typescript
const isValidName = (name: string): boolean =>
  typeof name === 'string' && name.trim() !== '' && !/^\d+$/.test(name)

export function parseArgumentNames(
  argumentNames: string | string[] | undefined,
): string[] {
  if (!argumentNames) return []

  if (Array.isArray(argumentNames)) {
    return argumentNames.filter(isValidName)
  }
  if (typeof argumentNames === 'string') {
    return argumentNames.split(/\s+/).filter(isValidName)
  }
  return []
}
```

### 替换逻辑

```typescript
export function substituteArguments(
  content: string,
  args: string | undefined,
  appendIfNoPlaceholder = true,
  argumentNames: string[] = [],
): string {
  if (args === undefined || args === null) {
    return content
  }

  const parsedArgs = parseArguments(args)
  const originalContent = content

  // 1. 替换命名参数（如 $foo）
  for (let i = 0; i < argumentNames.length; i++) {
    const name = argumentNames[i]
    if (!name) continue
    // 匹配 $name 但不匹配 $name[...] 或 $nameXxx
    content = content.replace(
      new RegExp(`\\$${name}(?![\\[\\w])`, 'g'),
      parsedArgs[i] ?? '',
    )
  }

  // 2. 替换索引参数（$ARGUMENTS[0], $ARGUMENTS[1]）
  content = content.replace(/\$ARGUMENTS\[(\d+)\]/g, (_, indexStr: string) => {
    const index = parseInt(indexStr, 10)
    return parsedArgs[index] ?? ''
  })

  // 3. 替换简写索引（$0, $1）
  content = content.replace(/\$(\d+)(?!\w)/g, (_, indexStr: string) => {
    const index = parseInt(indexStr, 10)
    return parsedArgs[index] ?? ''
  })

  // 4. 替换完整参数（$ARGUMENTS）
  content = content.replaceAll('$ARGUMENTS', args)

  // 5. 无占位符时追加
  if (content === originalContent && appendIfNoPlaceholder && args) {
    content = content + `\n\nARGUMENTS: ${args}`
  }

  return content
}
```

## 关键代码路径与文件引用

### 本文件导出
- `parseArguments(args: string): string[]` - 解析参数
- `parseArgumentNames(argumentNames): string[]` - 解析参数名
- `generateProgressiveArgumentHint(argNames, typedArgs): string | undefined` - 生成提示
- `substituteArguments(content, args, appendIfNoPlaceholder?, argumentNames?): string` - 替换参数

### 依赖文件
| 文件 | 用途 |
|------|------|
| `src/utils/bash/shellQuote.ts` | shell-quote 包装器（`tryParseShellCommand`） |

### 调用方
- `src/skills/loadSkillsDir.ts` - 技能加载
- `src/hooks/useTypeahead.tsx` - 类型提示
- `src/utils/plugins/loadPluginCommands.ts` - 插件命令加载
- `src/utils/hooks/hookHelpers.ts` - Hook 辅助函数

## 依赖与外部交互

### 外部 npm 依赖
- `shell-quote` - 通过 `src/utils/bash/shellQuote.ts` 间接使用

### 参数语法支持
| 语法 | 示例 | 说明 |
|------|------|------|
| `$ARGUMENTS` | `$ARGUMENTS` | 完整参数字符串 |
| `$ARGUMENTS[n]` | `$ARGUMENTS[0]` | 第 n 个参数 |
| `$n` | `$0`, `$1` | 简写索引 |
| `$name` | `$foo` | 命名参数 |

## 风险、边界与改进建议

### 已知限制
1. **正则性能**：命名参数替换使用动态正则，参数多时可能影响性能
2. **转义处理**：不支持转义的 `$`（如 `\$foo`）
3. **递归替换**：不支持嵌套参数引用

### 边界条件
1. **空参数**：空字符串参数与无参数区分
2. **索引越界**：返回空字符串而非错误
3. **命名冲突**：数字名称与索引语法冲突（已过滤）

### 安全风险
1. **注入风险**：参数直接替换到内容中，需确保调用方已验证
2. **正则 DoS**：恶意构造的参数名可能导致 ReDoS

### 改进建议
1. **转义支持**：添加 `\$` 转义机制
2. **默认值**：支持 `${name:-default}` 语法
3. **类型检查**：为命名参数添加类型验证
4. **性能优化**：缓存编译后的正则表达式

### 测试建议
1. 测试各种引号组合的参数解析
2. 测试边界索引（负数、超大数）
3. 测试特殊字符在参数中的处理
4. 测试命名参数与索引参数的优先级
