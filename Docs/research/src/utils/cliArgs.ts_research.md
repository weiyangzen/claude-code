# src/utils/cliArgs.ts 深度研究文档

## 场景与职责

`cliArgs.ts` 提供 CLI 参数解析的辅助函数，用于在 Commander.js 正式处理参数之前，提前解析某些关键标志。这主要用于需要在应用初始化阶段就获取的参数（如 `--settings`）。

设计背景：
- Commander.js 在 `init()` 之后处理参数
- 某些参数（如 `--settings`）影响配置加载，需要在 `init()` 之前解析
- 需要处理 Commander.js 的 `--` 分隔符传递行为

## 功能点目的

### 1. 提前参数解析 (`eagerParseCliFlag`)
- 在 Commander.js 处理之前解析特定标志
- 支持 `--flag=value` 和 `--flag value` 两种格式
- 用于 `--settings` 等影响初始化的参数

### 2. `--` 分隔符处理 (`extractArgsAfterDoubleDash`)
- 处理 Unix 标准的 `--` 参数分隔符
- Commander.js 的 `.passThroughOptions()` 会将 `--` 作为位置参数传递
- 此函数纠正解析，正确提取 `--` 后的命令和参数

## 具体技术实现

### 提前参数解析

```typescript
export function eagerParseCliFlag(
  flagName: string,
  argv: string[] = process.argv,
): string | undefined {
  for (let i = 0; i < argv.length; i++) {
    const arg = argv[i]
    // 处理 --flag=value 语法
    if (arg?.startsWith(`${flagName}=`)) {
      return arg.slice(flagName.length + 1)
    }
    // 处理 --flag value 语法
    if (arg === flagName && i + 1 < argv.length) {
      return argv[i + 1]
    }
  }
  return undefined
}
```

### `--` 分隔符处理

```typescript
export function extractArgsAfterDoubleDash(
  commandOrValue: string,
  args: string[] = [],
): { command: string; args: string[] } {
  if (commandOrValue === '--' && args.length > 0) {
    return {
      command: args[0]!,
      args: args.slice(1),
    }
  }
  return { command: commandOrValue, args }
}
```

### 使用示例

```typescript
// 提前解析 --settings
const settingsPath = eagerParseCliFlag('--settings')
if (settingsPath) {
  // 在 init() 之前加载自定义设置
  await loadCustomSettings(settingsPath)
}

// 处理 -- 分隔符
// 用户输入: claude --print -- some-script.js arg1 arg2
// Commander 解析: positional1 = "--", rest = ["some-script.js", "arg1", "arg2"]
const { command, args } = extractArgsAfterDoubleDash(positional1, rest)
// 结果: command = "some-script.js", args = ["arg1", "arg2"]
```

## 依赖与外部交互

### 无外部依赖

此模块是纯工具函数，不依赖任何外部模块。

### 被依赖方

| 模块 | 用途 |
|------|------|
| `src/main.tsx` | 提前解析 `--settings` 参数 |

## 风险、边界与改进建议

### 已知风险

1. **参数解析不一致**
   - `eagerParseCliFlag` 是简化实现，可能与其他参数解析器行为不一致
   - 例如：不处理引号、不处理转义、不支持数组值

2. **索引越界**
   - 已检查 `i + 1 < argv.length`，但边界情况仍需注意

### 边界情况

1. **重复标志**
   - 如果同一标志出现多次，返回第一次出现的值
   - 这与大多数参数解析器的行为一致

2. **空值**
   - `--flag `（空格后无值）不会返回空字符串，而是返回 undefined
   - 因为 `i + 1 < argv.length` 检查会失败

3. **`--` 后无参数**
   - `extractArgsAfterDoubleDash('--', [])` 返回 `{ command: '--', args: [] }`
   - 保持原值不变

4. **嵌套 `--`**
   - 不特殊处理，第一个 `--` 后的所有内容都作为参数

### 改进建议

1. **功能增强**
   - 支持布尔标志（存在性检测）
   - 支持数组值（`--flag val1 --flag val2`）
   - 支持负数索引（从末尾解析）

2. **健壮性**
   - 添加参数验证（如标志名格式检查）
   - 处理引号包裹的值
   - 添加更详细的错误信息

3. **测试覆盖**
   - 添加单元测试覆盖各种边界情况
   - 测试与 Commander.js 的行为一致性

4. **代码示例**

```typescript
// 改进版本（支持布尔标志和数组）
export function eagerParseCliFlag(
  flagName: string,
  argv: string[] = process.argv,
  options: { multiple?: boolean; boolean?: boolean } = {},
): string | string[] | boolean | undefined {
  const results: string[] = []
  
  for (let i = 0; i < argv.length; i++) {
    const arg = argv[i]
    
    // 处理 --flag=value
    if (arg?.startsWith(`${flagName}=`)) {
      const value = arg.slice(flagName.length + 1)
      if (options.boolean) return true
      if (options.multiple) results.push(value)
      else return value
    }
    
    // 处理 --flag value
    if (arg === flagName) {
      if (options.boolean) return true
      if (i + 1 < argv.length && !argv[i + 1]!.startsWith('-')) {
        if (options.multiple) results.push(argv[i + 1]!)
        else return argv[i + 1]
      }
    }
  }
  
  return options.multiple ? results : undefined
}
```
