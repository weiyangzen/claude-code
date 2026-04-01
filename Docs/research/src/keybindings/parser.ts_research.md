# parser.ts 研究文档

## 场景与职责

`src/keybindings/parser.ts` 是 Claude Code 快捷键系统的**解析与格式化核心**。它负责将人类可读的快捷键字符串（如 `"ctrl+shift+k"`）转换为结构化的内部表示（`ParsedKeystroke` / `Chord`），并提供反向的字符串格式化功能（用于 UI 展示）。该模块是纯函数集合，无状态、无副作用，可被配置加载器、校验器、UI 组件等任意调用。

## 功能点目的

1. **Keystroke 解析**：把 `"ctrl+k"` 这类单键组合解析为带修饰符标志的结构化对象。
2. **Chord 解析**：支持多段快捷键（chord），如 `"ctrl+k ctrl+s"`，按空格分隔为多个 `ParsedKeystroke`。
3. **Canonical 字符串化**：将解析后的对象还原为统一的显示字符串（如 `"ctrl+Space"`）。
4. **平台适配显示**：根据平台（macOS / 其他）生成不同的显示文本（如 macOS 显示 `"opt"`，其他显示 `"alt"`）。
5. **Binding 块解析**：将 `KeybindingBlock[]`（来自 JSON 配置）扁平化为 `ParsedBinding[]`，供解析器使用。

## 具体技术实现

### 关键数据结构

```ts
// 推断自代码使用方式（原 types.ts 在仓库快照中缺失）
type ParsedKeystroke = {
  key: string      // 主键，如 'k', 'enter', 'up'
  ctrl: boolean
  alt: boolean
  shift: boolean
  meta: boolean    // 终端中 alt 与 meta 通常等价
  super: boolean   // Cmd/Win，仅 kitty 协议终端支持
}

type Chord = ParsedKeystroke[]

type ParsedBinding = {
  chord: Chord
  action: string | null
  context: KeybindingContextName
}

type KeybindingBlock = {
  context: KeybindingContextName
  bindings: Record<string, string | null>
}
```

### 关键流程

#### 1. `parseKeystroke(input: string): ParsedKeystroke`
- 以 `"+"` 分割输入字符串。
- 遍历每个部分，通过 `switch(lower)` 识别修饰符别名：
  - `ctrl` / `control` → `ctrl = true`
  - `alt` / `opt` / `option` → `alt = true`
  - `meta` → `meta = true`
  - `cmd` / `command` / `super` / `win` → `super = true`
- 特殊键名映射：`esc` → `escape`，`return` → `enter`，`space` → `' '`，箭头符号 `↑↓←→` 映射为英文方向名。
- 未匹配的部分作为主键 `key`，统一转小写。

#### 2. `parseChord(input: string): Chord`
- 边界处理：若输入恰好是单个空格字符 `' '`，直接返回 `[parseKeystroke('space')]`，避免被 `trim().split(/\s+/)` 误处理为空数组。
- 否则 `trim().split(/\s+/)` 后逐个调用 `parseKeystroke`。

#### 3. `keystrokeToString(ks: ParsedKeystroke): string`
- 按固定顺序拼接修饰符：`ctrl` → `alt` → `shift` → `meta` → `cmd`（`super` 在 canonical 形式中显示为 `cmd`）。
- 主键通过 `keyToDisplayName` 做可读性转换：
  - `escape` → `Esc`
  - `' '` → `Space`
  - `up/down/left/right` → `↑/↓/←/→`
  - 其他保持原样。

#### 4. `keystrokeToDisplayString(ks, platform)`
- 与 `keystrokeToString` 类似，但平台敏感：
  - macOS 下 `alt||meta` 显示为 `opt`
  - macOS 下 `super` 显示为 `cmd`
  - 其他平台 `super` 显示为 `super`

#### 5. `parseBindings(blocks: KeybindingBlock[]): ParsedBinding[]`
- 双重循环遍历所有 block 及其 `bindings` 记录。
- 对每个键调用 `parseChord`，生成 `ParsedBinding` 并压入结果数组。
- 输出为扁平数组，后续由 `loadUserBindings.ts` 与默认绑定合并（`[...default, ...user]`，后入覆盖）。

## 关键代码路径与文件引用

| 函数 | 被调用方 |
|------|----------|
| `parseKeystroke` | `validate.ts` (`validateKeystroke`)、`resolver.ts` (`buildKeystroke` 间接对比)、`loadUserBindings.ts` (通过 `parseBindings`) |
| `parseChord` | `validate.ts` (voice push-to-talk 校验)、`resolver.ts` (chord 匹配逻辑) |
| `keystrokeToString` / `chordToString` | `resolver.ts` (`getBindingDisplayText`)、`validate.ts` (`checkReservedShortcuts`)、`KeybindingContext.tsx` (展示) |
| `keystrokeToDisplayString` / `chordToDisplayString` | `KeybindingContext.tsx`、`HelpV2.tsx` 等 UI 组件 |
| `parseBindings` | `loadUserBindings.ts` (默认与用户绑定加载)、`defaultBindings.ts` (初始化解析) |

## 依赖与外部交互

- **无外部运行时依赖**：纯本地字符串处理。
- **类型依赖**：从 `./types.js` 导入 `Chord`, `KeybindingBlock`, `ParsedBinding`, `ParsedKeystroke`。
  - ⚠️ **注意**：当前仓库快照中 `src/keybindings/types.ts` 文件缺失，但所有模块均通过 `./types.js` 引用；推断该文件在完整构建产物或上游源码中存在，定义了上述核心类型。

## 风险、边界与改进建议

### 风险与边界

1. **大小写敏感与别名一致性**
   - `parseKeystroke` 将所有非特殊键转小写，因此 `"Ctrl+K"` 与 `"ctrl+k"` 等价。但 `keystrokeToString` 输出固定格式，若用户配置与默认配置大小写不同，在展示层会统一为 canonical 形式。
   
2. **空格键的歧义处理**
   - `parseChord` 对单个空格 `' '` 做了特殊保护，这是正确的边界处理；但若用户写 `"space space"`（双空格 chord），`trim().split(/\s+/)` 会将其合并为单元素数组 `"space"`，无法表达 `"space space"` chord。当前产品未使用此类 chord，风险可控。

3. **alt / meta 终端等价性**
   - 解析层将 `alt` 和 `meta` 分别存入不同布尔字段，但匹配层（`match.ts` / `resolver.ts`）会将二者视为等价。这是有意设计，以兼容无法区分 Alt 与 Meta 的传统终端。

4. **super (cmd) 的可用性受限**
   - `super` 修饰符仅在支持 kitty keyboard protocol 的终端（kitty、WezTerm、ghostty、iTerm2）上才能被应用接收。解析层支持它，但匹配层注释明确说明：在不支持该协议的终端上，`cmd+` 绑定永远不会触发。

5. **缺失的 `types.ts` 文件**
   - 仓库快照缺少 `src/keybindings/types.ts`，导致无法直接查看类型的完整定义和文档注释。建议确认该文件是否被意外排除在快照外。

### 改进建议

1. **统一 chord 空格解析**
   - 若未来需要支持包含空格键的多段 chord，可考虑引入转义语法（如 `"space,space"` 或显式 `"space space"` 保护逻辑），但目前需求优先级低。

2. **解析错误增强**
   - `parseKeystroke` 在无法识别任何键/修饰符时，仅返回空 `key` 和全 `false` 修饰符，由调用方（`validate.ts`）二次判断。可考虑在解析器内直接抛出/返回结构化错误，减少调用方重复校验。

3. **类型文件补全**
   - 建议将 `types.ts` 纳入版本控制或文档快照，因为它是 9 个以上模块的共同依赖，缺失会导致静态分析和新人理解成本显著上升。
