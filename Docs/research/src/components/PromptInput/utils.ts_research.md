# utils.ts 研究文档

## 场景与职责

PromptInput 目录的通用工具函数，为输入行为、终端特性提供纯函数支持。包含三个导出：
- `isVimModeEnabled()`：判断是否启用 Vim 编辑模式
- `getNewlineInstructions()`：返回当前终端的换行输入提示
- `isNonSpacePrintable(input, key)`：判断按键是否为非空白可打印字符，用于控制图片 pill 后自动空格的插入

## 功能点目的

- 统一 Vim 模式开关判断
- 根据终端类型告诉用户如何输入换行
- 过滤控制键，避免图片 pill 后的预插入空格被错误保留

## 具体技术实现

### 关键流程

#### `isVimModeEnabled()`
- `getGlobalConfig().editorMode === 'vim'`

#### `getNewlineInstructions()`
- Apple Terminal on macOS → `shift + ⏎ for newline`
- iTerm2 / VSCode 等且 `isShiftEnterKeyBindingInstalled()` → `shift + ⏎ for newline`
- 否则：`hasUsedBackslashReturn()` ? `\⏎ for newline` : `backslash (\) + return (⏎) for newline`

#### `isNonSpacePrintable(input, key)`
- 排除控制键：`ctrl`/`meta`/`escape`/`return`/`tab`/`backspace`/`delete`/方向键/`pageUp/Down`/`home`/`end`
- 要求 `input.length > 0 && !/^\s/.test(input) && !input.startsWith('\x1b')`

### 数据结构

- `key: Key` 来自 `src/ink.js`

### 协议/命令

- `getGlobalConfig()` → `src/utils/config.ts`
- `isShiftEnterKeyBindingInstalled()` / `hasUsedBackslashReturn()` → `src/commands/terminalSetup/terminalSetup.tsx`
- `env.terminal` → `src/utils/env.ts`

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/components/PromptInput/utils.ts` | 本文件 |
| `src/components/PromptInput/PromptInput.tsx` | 调用 `isVimModeEnabled`、`isNonSpacePrintable` |
| `src/components/PromptInput/PromptInputHelpMenu.tsx` | 调用 `getNewlineInstructions` |
| `src/components/PromptInput/PromptInputFooterLeftSide.tsx` | 调用 `isVimModeEnabled` |
| `src/components/StatusLine.tsx` | 调用 `isVimModeEnabled` |
| `src/hooks/useCancelRequest.ts` | 调用 `isVimModeEnabled` |
| `src/commands/terminalSetup/terminalSetup.tsx` | 提供绑定状态与 backslash 记录 |
| `src/utils/env.ts` | 终端环境 |
| `src/utils/config.ts` | 全局配置 |

## 依赖与外部交互

纯函数，但依赖全局配置和终端环境检测结果。`getNewlineInstructions` 与 `terminalSetup.tsx` 深度耦合。

## 风险、边界与改进建议

1. **平台检测硬编码**：`env.terminal === 'Apple_Terminal'` 与 `process.platform === 'darwin'` 耦合较紧
2. **Vim 模式扩展性**：目前仅判断 `'vim'`，若增加 `helix` 等模式需扩展为集合判断
3. **`isNonSpacePrintable` 排除列表**：未覆盖 `fn` 键、多媒体键等，但 ink 通常不会对这些键发送 input 字符串
4. **测试建议**：mock `getGlobalConfig` 验证 Vim 模式；mock 终端与绑定状态验证换行提示各分支；构造多种 `Key`+`input` 组合验证可打印字符判断
