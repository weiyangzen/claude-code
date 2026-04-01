# src/utils/keyboardShortcuts.ts 研究文档

## 场景与职责

`keyboardShortcuts.ts` 解决 macOS 终端中一个特定的键盘输入问题：当终端未启用 "Option as Meta" 设置时，用户按下 `Option+T`、`Option+P`、`Option+O` 等组合键，终端不会发送标准的 `alt+t`、`alt+p`、`alt+o` 转义序列，而是发送一个特殊字符（如 `†`、`π`、`ø`）。

该模块维护了一个特殊字符到键绑定等价形式的映射表，使 Claude Code 的输入处理系统能够正确识别这些 macOS Option 键快捷操作。

调用方：
- `src/components/PromptInput/PromptInput.tsx`

## 功能点目的

### `MACOS_OPTION_SPECIAL_CHARS`
映射表定义：

```typescript
export const MACOS_OPTION_SPECIAL_CHARS = {
  '†': 'alt+t', // Option+T -> thinking toggle
  'π': 'alt+p', // Option+P -> model picker
  'ø': 'alt+o', // Option+O -> fast mode
} as const satisfies Record<string, string>
```

这三个快捷键分别对应：
- `alt+t`：切换 thinking 模式
- `alt+p`：打开模型选择器
- `alt+o`：切换 fast 模式

### `isMacosOptionChar`
类型守卫函数，检查给定字符是否在映射表中：

```typescript
export function isMacosOptionChar(
  char: string,
): char is keyof typeof MACOS_OPTION_SPECIAL_CHARS {
  return char in MACOS_OPTION_SPECIAL_CHARS
}
```

## 具体技术实现

### 映射表的类型安全
使用 `as const satisfies Record<string, string>` 确保：
1. 映射值都是字符串。
2. `typeof MACOS_OPTION_SPECIAL_CHARS` 可以推导出具体的字面量联合类型（`'†' | 'π' | 'ø'`）。
3. `keyof typeof MACOS_OPTION_SPECIAL_CHARS` 因此也是精确的字面量联合类型。

### 调用方使用方式
在 `PromptInput.tsx` 中，输入处理逻辑可能在 `onInput` 或 keypress 事件处理中检查：

```typescript
if (isMacosOptionChar(inputChar)) {
  const shortcut = MACOS_OPTION_SPECIAL_CHARS[inputChar]
  // 将 shortcut 传递给键绑定处理系统
}
```

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/utils/keyboardShortcuts.ts:4-8` | `MACOS_OPTION_SPECIAL_CHARS` 映射表 |
| `src/utils/keyboardShortcuts.ts:10-14` | `isMacosOptionChar` 类型守卫 |
| `src/components/PromptInput/PromptInput.tsx` | 调用方，处理终端输入 |

## 依赖与外部交互

### 依赖
- 无任何内部或外部依赖。

### 调用方
- `src/components/PromptInput/PromptInput.tsx`

## 风险、边界与改进建议

### 风险与边界
1. **映射不完整**：目前只覆盖了 3 个 Option+字母 组合（T/P/O）。macOS 的 Option 键还会产生大量其他特殊字符（如 `å` = Option+A、`ß` = Option+S 等），如果未来为这些操作添加键绑定，需要持续维护此映射表。
2. **国际化键盘布局差异**：不同国家/地区的 Mac 键盘布局下，Option+字母 产生的字符可能不同。例如，某些欧洲键盘布局中 Option+T 可能不产生 `†`。这意味着该修复对非美式键盘布局的 macOS 用户可能不完全有效。
3. **与 "Option as Meta" 的冲突**：如果用户启用了 "Option as Meta"，终端会正确发送 `alt+t` 等序列，不会发送特殊字符。此时该模块不会被触发，这是预期行为。
4. **无法覆盖其他修饰键组合**：该模块只处理 Option 键产生的特殊字符，不处理 Control、Command 或其他组合键的等效形式。

### 改进建议
1. **扩展映射表**：收集更多常用的 alt 快捷键对应的 macOS Option 特殊字符，建立一个更完整的映射。可以参考 iTerm2 或 Terminal.app 的键绑定文档。
2. **键盘布局感知**：考虑通过环境变量或终端能力检测来判断用户是否使用美式键盘布局，对于非美式布局提供降级提示（如"你的键盘布局可能不支持 Option 快捷键"）。
3. **文档化**：在 CLI 的帮助文档或键绑定提示中，明确说明 macOS 用户需要启用 "Option as Meta" 以获得最佳体验，并解释当前映射表只是兼容性补丁。
4. **自动化测试**：可以编写单元测试覆盖 `isMacosOptionChar` 和映射查找，确保新增键绑定时不会破坏现有逻辑。
5. **与键绑定系统统一**：当前映射表是硬编码的，与 `src/keybindings/schema.ts` 等键绑定定义没有自动关联。可以考虑从键绑定 schema 中自动生成 `alt+{key}` 到 macOS 特殊字符的映射。
