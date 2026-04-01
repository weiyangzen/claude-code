# commentLabel.ts 研究文档

## 场景与职责

`commentLabel.ts` 是 BashTool 的**命令标签提取模块**，专门用于从 bash 命令中提取首行注释作为显示标签。在全屏模式下，这个标签既是非详细工具使用标签，也是折叠组的提示文本，用于向用户展示 Claude 编写的命令的人类可读描述。

### 核心职责
1. **注释提取**：从命令字符串中提取第一行的 `#` 注释内容
2. **Shebang 排除**：区分 shebang (`#!`) 和普通注释 (`#`)
3. **标签清理**：去除注释标记和前后空白，生成干净的标签文本

## 功能点目的

### 1. 注释标签提取
函数 `extractBashCommentLabel` 实现了从 bash 命令中提取人类可读标签的功能：

```typescript
export function extractBashCommentLabel(command: string): string | undefined
```

**设计目的**：
- 允许 Claude 在命令前添加描述性注释，帮助用户理解命令意图
- 在全屏模式下提供更友好的命令显示（而非原始命令字符串）
- 支持折叠组的提示文本显示

### 2. Shebang 排除逻辑
```typescript
if (!firstLine.startsWith('#') || firstLine.startsWith('#!')) return undefined
```

**为什么排除 shebang**：
- Shebang (`#!/bin/bash`) 是解释器指令，不是人类可读描述
- 普通注释 (`# 这是描述`) 才是为阅读者准备的

## 具体技术实现

### 关键流程

#### 1. 首行提取
```typescript
const nl = command.indexOf('\n')
const firstLine = (nl === -1 ? command : command.slice(0, nl)).trim()
```

- 查找第一个换行符
- 如果没有换行符，整个命令视为首行
- 使用 `trim()` 去除前后空白

#### 2. 注释类型判断
```typescript
if (!firstLine.startsWith('#') || firstLine.startsWith('#!')) return undefined
```

- 必须以 `#` 开头
- 不能以 `#!` 开头（排除 shebang）

#### 3. 标签清理
```typescript
return firstLine.replace(/^#+\s*/, '') || undefined
```

- 使用正则表达式 `^#+\s*` 去除开头的多个 `#` 和后续空白
- 如果结果为空字符串，返回 `undefined`

### 代码示例

```typescript
// 示例 1: 普通注释
extractBashCommentLabel("# 列出当前目录\nls -la")
// 返回: "列出当前目录"

// 示例 2: Shebang
extractBashCommentLabel("#!/bin/bash\nls -la")
// 返回: undefined

// 示例 3: 多行注释
extractBashCommentLabel("## 配置检查\ncat config.ini")
// 返回: "配置检查"

// 示例 4: 无注释
extractBashCommentLabel("ls -la")
// 返回: undefined

// 示例 5: 空注释
extractBashCommentLabel("#\nls -la")
// 返回: undefined
```

## 关键代码路径与文件引用

### 导出函数
- `extractBashCommentLabel`: 主入口函数

### 调用方
- `src/tools/BashTool/UI.tsx`: 在 `renderToolUseMessage` 中使用
  ```typescript
  if (isFullscreenEnvEnabled()) {
    const label = extractBashCommentLabel(command)
    if (label) {
      return label.length > MAX_COMMAND_DISPLAY_CHARS ? label.slice(0, MAX_COMMAND_DISPLAY_CHARS) + '…' : label
    }
  }
  ```

- `src/utils/collapseReadSearch.ts`: 用于折叠组的提示文本

### 相关文件
- `src/tools/BashTool/UI.tsx`: 命令渲染 UI 组件
- `src/utils/collapseReadSearch.ts`: 折叠搜索工具结果

## 依赖与外部交互

### 运行时依赖
该模块**无外部依赖**，是纯字符串处理函数。

### 函数签名
```typescript
export function extractBashCommentLabel(command: string): string | undefined
```

| 参数 | 类型 | 描述 |
|------|------|------|
| command | string | 完整的 bash 命令字符串 |

| 返回值 | 类型 | 描述 |
|--------|------|------|
| - | string | 提取的注释标签（已清理） |
| - | undefined | 无有效注释或只有 shebang |

## 风险、边界与改进建议

### 已知风险

1. **首行限制**
   - 仅处理第一行注释，多行描述注释只有第一行有效
   - 命令前有空行时可能误判

2. **注释位置敏感**
   - 注释必须在命令的最开始
   - `echo foo # 描述` 这种行尾注释不会被提取

3. **无验证机制**
   - 不验证注释内容的质量或适当性
   - 恶意用户可能通过注释进行社会工程学攻击

### 边界情况

| 输入 | 输出 | 说明 |
|------|------|------|
| `""` | `undefined` | 空字符串 |
| `"#"` | `undefined` | 只有 `#` 无内容 |
| `"#!"` | `undefined` | 不完整 shebang |
| `"#\ncommand"` | `undefined` | `#` 后无内容 |
| `"### 标题"` | `"标题"` | 多个 `#` |
| `"#  多空格  "` | `"多空格"` | 内部和尾部空格保留 |

### 改进建议

1. **支持行尾注释**
   ```typescript
   // 可选：支持提取行尾注释作为备选
   function extractInlineComment(command: string): string | undefined
   ```

2. **多行注释支持**
   ```typescript
   // 可选：支持合并连续的多行注释
   function extractMultiLineComment(command: string): string | undefined
   ```

3. **注释长度限制**
   ```typescript
   // 当前调用方处理截断，可考虑在提取时限制
   const MAX_LABEL_LENGTH = 200
   ```

4. **HTML/特殊字符转义**
   - 确保提取的标签在 UI 中安全显示
   - 当前调用方在 React 中使用，自动转义

5. **性能优化**
   - 当前实现已足够高效（O(n) 扫描）
   - 对于极长命令，可考虑使用 `split('\n', 1)` 优化
   ```typescript
   const firstLine = command.split('\n', 1)[0]?.trim() ?? ''
   ```

### 测试建议

建议添加以下测试用例：
```typescript
describe('extractBashCommentLabel', () => {
  it('提取普通注释', () => {
    expect(extractBashCommentLabel('# 描述\nls')).toBe('描述')
  })
  it('排除 shebang', () => {
    expect(extractBashCommentLabel('#!/bin/bash\nls')).toBeUndefined()
  })
  it('处理多 #', () => {
    expect(extractBashCommentLabel('### 标题\nls')).toBe('标题')
  })
  it('空输入', () => {
    expect(extractBashCommentLabel('')).toBeUndefined()
  })
  it('无换行符', () => {
    expect(extractBashCommentLabel('# 描述')).toBe('描述')
  })
})
```
