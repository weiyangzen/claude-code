# reservedShortcuts.ts 研究文档

## 场景与职责

`src/keybindings/reservedShortcuts.ts` 负责定义和维护**系统保留快捷键清单**，并提供平台相关的保留键查询与键名规范化能力。它是快捷键配置校验的"黑名单"数据源，确保用户不会将关键系统快捷键（如 `ctrl+c`、`cmd+q`）绑定到应用功能上，从而避免不可预期的行为或完全失效的绑定。

## 功能点目的

1. **定义不可重新绑定的快捷键**：如 `ctrl+c`（中断）、`ctrl+d`（退出）等硬编码在 Claude Code 中的快捷键。
2. **定义终端/操作系统保留快捷键**：如 `ctrl+z`（SIGTSTP）、`ctrl+\`（SIGQUIT）等，这些键通常被终端或 OS 拦截，应用层无法收到。
3. **平台差异化保留键**：如 macOS 特有的 `cmd+c`、`cmd+v`、`cmd+q` 等系统级快捷键。
4. **键名规范化比较**：提供 `normalizeKeyForComparison`，将不同写法（`control` vs `ctrl`、`option` vs `alt`）统一为可比较的 canonical 形式，用于后续冲突检测。

## 具体技术实现

### 关键数据结构

```ts
export type ReservedShortcut = {
  key: string        // 原始键字符串，如 "ctrl+c"
  reason: string     // 保留原因说明
  severity: 'error' | 'warning'
}
```

### 保留快捷键清单

#### `NON_REBINDABLE`（error 级别）
- `ctrl+c`：硬编码中断/退出
- `ctrl+d`：硬编码退出
- `ctrl+m`：终端中与 Enter 等价（都发送 CR），无法重新绑定

#### `TERMINAL_RESERVED`（混合级别）
- `ctrl+z`：Unix 进程挂起（SIGTSTP）— `warning`
- `ctrl+\`：终端退出信号（SIGQUIT）— `error`
- **注意**：`ctrl+s`（XOFF）和 `ctrl+q`（XON）**未被包含**，因为现代终端默认禁用流控制，且 Claude Code 将 `ctrl+s` 用于 stash 功能。

#### `MACOS_RESERVED`（error 级别）
- `cmd+c` / `cmd+v` / `cmd+x`：系统剪贴板操作
- `cmd+q`：退出应用
- `cmd+w`：关闭窗口/标签
- `cmd+tab`：应用切换器
- `cmd+space`：Spotlight

### 关键函数

#### `getReservedShortcuts(): ReservedShortcut[]`
- 调用 `getPlatform()`（来自 `src/utils/platform.js`）判断当前平台。
- 返回合并数组：`[...NON_REBINDABLE, ...TERMINAL_RESERVED, ...(macos ? MACOS_RESERVED : [])]`
- 非 rebindable 项排在最前，体现最高优先级。

#### `normalizeKeyForComparison(key: string): string`
- **按空格分割**处理 chord（如 `"ctrl+x ctrl+b"`），每段独立调用 `normalizeStep`。
- 这是关键设计：如果先按 `+` 分割再按空格处理，会把 `"x ctrl"` 误解析为后续步骤的修饰符，导致 chord 被压缩为最后一个键。

#### `normalizeStep(step: string): string`
- 按 `+` 分割，识别修饰符并统一别名：
  - `control` → `ctrl`
  - `option` / `opt` → `alt`
  - `command` / `cmd` → `cmd`
  - `shift` / `meta` 保持原样
- 修饰符按字母排序后拼接主键，生成 canonical 形式（如 `"ctrl+shift+k"`）。

## 关键代码路径与文件引用

| 导出项 | 被调用方 | 用途 |
|--------|----------|------|
| `getReservedShortcuts` | `validate.ts` (`checkReservedShortcuts`) | 校验用户绑定是否冲突 |
| `NON_REBINDABLE` | `template.ts` (`filterReservedShortcuts`) | 生成模板时过滤掉不可重新绑定的键 |
| `normalizeKeyForComparison` | `validate.ts` (`checkDuplicates`)、`template.ts` | 去重与过滤的键比较 |
| `MACOS_RESERVED` / `TERMINAL_RESERVED` | `src/skills/bundled/keybindings.ts` | 生成帮助文档中的保留快捷键列表 |

## 依赖与外部交互

- `../utils/platform.js`：`getPlatform()` 用于判断 macOS / Windows / Linux / WSL。
- 无 React 依赖，可在任意上下文（校验器、技能文档生成器）中安全使用。

## 风险、边界与改进建议

### 风险与边界

1. **平台检测的局限性**
   - `getPlatform()` 在 WSL 环境下返回 `'wsl'`，但 `getReservedShortcuts()` 仅在 `platform === 'macos'` 时追加 macOS 保留键。WSL 用户实际运行在 Windows 上，可能还受 Windows 终端保留键影响（如 `ctrl+v`），但当前未对 Windows 定义额外保留键列表。

2. **normalizeKeyForComparison 的 chord 处理**
   - 该函数正确避免了 `"ctrl+x ctrl+k"` 被错误压缩的问题，但对非法 chord（如连续多个空格、空步骤）没有显式报错，只是会原样传递空字符串给 `normalizeStep`。

3. **severity 级别的人为设定**
   - `ctrl+z` 被标记为 `warning`（因为现代终端可能配置不同），而 `ctrl+\` 是 `error`。这种区分基于经验判断，但不同终端/Shell 配置下实际行为可能有所差异。

4. **缺失 Windows 专用保留键**
   - Windows 终端有大量系统级快捷键（如 `ctrl+shift+c` 复制、`alt+f4` 关闭窗口、`win+d` 显示桌面等），当前未在保留列表中体现。虽然部分键可能通过 VT 序列到达应用层，但一致性上存在缺口。

### 改进建议

1. **增加 Windows 保留键列表**
   - 可参照 macOS 模式增加 `WINDOWS_RESERVED`，至少包含 `ctrl+shift+c`（终端复制，与 Scroll 上下文中的 `selection:copy` 冲突风险）、`alt+f4` 等。

2. **将 normalizeKeyForComparison 与 parser.ts 的解析逻辑进一步统一**
   - 当前 `parser.ts` 和 `reservedShortcuts.ts` 各自维护了一套修饰符别名映射表，存在重复和潜在不一致风险。建议将别名映射抽取到共享常量（如 `MODIFIER_ALIASES`）。

3. **支持用户自定义保留键白名单**
   - 某些高级用户可能确实希望在特定终端配置下使用 `ctrl+z`。可考虑在 `keybindings.json` 中增加 `"ignoreReservedWarnings": ["ctrl+z"]` 之类的配置项，但需权衡复杂度与收益。

4. **补充 `types.ts` 文件**
   - 与 parser.ts 相同，`ReservedShortcut` 类型应 ideally 定义在统一的 `types.ts` 中，当前文件内联定义了该类型。
