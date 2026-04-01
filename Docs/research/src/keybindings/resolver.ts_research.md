# resolver.ts 研究文档

## 场景与职责

`src/keybindings/resolver.ts` 是 Claude Code 快捷键系统的**核心解析引擎**。它负责将 Ink 输入事件（`input: string` + `key: Key`）与已解析的绑定配置进行匹配，返回对应的动作名称或状态信息。该模块设计为**纯函数**（无状态、无副作用），支持单键绑定和 chord（多段快捷键）序列的完整生命周期管理。

## 功能点目的

1. **单键解析**（`resolveKey`）：处理简单的单键绑定，返回匹配、未绑定或无匹配结果。
2. **Chord 解析**（`resolveKeyWithChordState`）：支持 `"ctrl+k ctrl+s"` 这类多段快捷键，管理 chord 的开始、进行中和取消状态。
3. **展示文本查询**（`getBindingDisplayText`）：根据动作名称反向查找绑定的快捷键字符串（用于 UI 提示）。
4. **Keystroke 相等性比较**（`keystrokesEqual`）：处理 alt/meta 等价性、super 区分等终端特性。

## 具体技术实现

### 关键类型定义

```ts
export type ResolveResult =
  | { type: 'match'; action: string }
  | { type: 'none' }
  | { type: 'unbound' }

export type ChordResolveResult =
  | { type: 'match'; action: string }
  | { type: 'none' }
  | { type: 'unbound' }
  | { type: 'chord_started'; pending: ParsedKeystroke[] }
  | { type: 'chord_cancelled' }
```

### 关键流程

#### 1. `resolveKey`（单键解析，Phase 1 兼容）
- 仅处理 `binding.chord.length === 1` 的单键绑定。
- 使用 `Set` 优化上下文过滤：`ctxSet = new Set(activeContexts)`，将 O(n·m) 降为 O(n)。
- 遍历顺序决定优先级：后出现的绑定覆盖先出现的（用户绑定在默认绑定之后，自然实现覆盖）。
- 返回 `unbound` 时，表示该键被显式设为 `null`（用户意图取消默认行为）。

#### 2. `resolveKeyWithChordState`（完整 chord 支持）
- **Escape 取消**：若按下 Escape 且处于 chord 中，返回 `chord_cancelled`。
- **构建 Keystroke**：调用 `buildKeystroke(input, key)` 将 Ink 事件转换为 `ParsedKeystroke`。
  - 关键边界处理：Ink 在 Escape 按下时会设置 `key.meta=true`（遗留终端行为），此处强制将 Escape 键的 `meta` 设为 `false`，避免 chord 匹配失败。
- **前缀匹配检测**：
  - 遍历所有 `contextBindings`，找出 `chord.length > testChord.length` 且前缀匹配的绑定。
  - 使用 `chordWinners` Map 按 chord 字符串分组，处理 `null` unbind 覆盖场景：若用户将 `"ctrl+x ctrl+k"` 设为 `null`，但保留了 `"ctrl+x"` 的单键绑定，不应因 `"ctrl+x"` 是更长 chord 的前缀而进入等待状态。
  - 仅当存在非 `null` 的更长 chord 时，才返回 `chord_started`。
- **精确匹配**：若无更长 chord 可能，遍历查找精确匹配，同样遵循"后入优先"原则。

#### 3. `keystrokesEqual`（相等性比较）
- 核心逻辑：`(a.alt || a.meta) === (b.alt || b.meta)` —— 将 alt 和 meta 视为等价。
- `super` 严格比较，体现 kitty protocol 的独立性。

#### 4. `getBindingDisplayText`
- 使用 `bindings.findLast` 查找最后定义的匹配绑定（用户绑定优先）。
- 通过 `chordToString`（来自 `parser.ts`）将 chord 转换为展示字符串。

## 关键代码路径与文件引用

| 函数/类型 | 被调用方 | 用途 |
|-----------|----------|------|
| `resolveKey` | 保留用于向后兼容，当前主入口为 `resolveKeyWithChordState` | 单键解析 |
| `resolveKeyWithChordState` | `KeybindingContext.tsx`（通过 Provider 的 `resolve` 函数） | 核心解析入口 |
| `getBindingDisplayText` | `KeybindingContext.tsx`、`shortcutFormat.ts` | 获取动作对应的快捷键展示文本 |
| `keystrokesEqual` | `useVoiceIntegration.tsx` | 语音快捷键匹配 |
| `ChordResolveResult` / `ResolveResult` | `KeybindingContext.tsx`、`useKeybinding.ts` | 类型契约 |

## 依赖与外部交互

- `../ink.js`：`Key` 类型（来自 `input-event.ts`）。
- `./match.js`：`matchesBinding`、`getKeyName` —— 底层匹配逻辑。
- `./parser.js`：`chordToString` —— chord 展示字符串生成。
- `./types.js`：`KeybindingContextName`、`ParsedBinding`、`ParsedKeystroke`（类型依赖，文件缺失于快照）。

## 风险、边界与改进建议

### 风险与边界

1. **chord_started 的激进触发**
   - 当前逻辑只要存在更长的 chord 就进入等待状态，即使当前按键本身也有精确匹配。这是有意设计（chord 优先于单键），但可能导致用户意外触发 chord 等待（如想按 `"ctrl+k"` 单键，但存在 `"ctrl+k ctrl+s"`，结果进入 1 秒等待）。
   - 缓解：`KeybindingProviderSetup.tsx` 设置了 1 秒超时（`CHORD_TIMEOUT_MS`），超时后自动取消。

2. **Escape 的 meta 修饰符特殊处理**
   - `buildKeystroke` 中强制设置 `effectiveMeta = key.escape ? false : key.meta`。这是针对 Ink 遗留行为的 workaround，若 Ink 未来版本改变此行为，此处需要同步更新。

3. **alt/meta 等价性的一致性问题**
   - `keystrokesEqual` 将 alt/meta 视为等价，但 `keystrokeToDisplayString`（parser.ts）在 macOS 上只检查 `alt || meta` 显示为 `opt`。两处逻辑需要保持一致，否则可能出现匹配成功但展示不一致的情况。

4. **性能考虑**
   - 每次按键都遍历所有 `contextBindings`（最坏情况 O(n)）。对于大量绑定的场景（如用户配置了数百条自定义绑定），这可能成为瓶颈。可考虑按上下文和首键建立索引，但目前绑定数量有限，实际影响可忽略。

### 改进建议

1. **索引优化**
   - 在 `KeybindingProviderSetup.tsx` 或 `loadUserBindings.ts` 中构建 `Map<context, Map<key, binding[]>>` 索引，将解析复杂度从 O(n) 降至 O(1) 或 O(chord_length)。

2. **chord 等待的视觉反馈增强**
   - 当前 chord 等待状态仅在 `KeybindingContext.tsx` 中维护，UI 层可通过 `pendingChord` 状态显示当前已输入的 chord 前缀（如底部状态栏显示 `"ctrl+k"...`），帮助用户理解当前处于 chord 输入中。

3. **Escape 行为可配置**
   - 某些用户可能希望 Escape 作为普通键（如在 vim 模式下），而非 chord 取消键。可考虑在 `keybindings.json` 中增加 `"chordCancelKey": "escape"` 配置项。

4. **补充单元测试**
   - 当前仓库快照中未发现测试文件。resolver.ts 作为纯函数模块，非常适合单元测试，建议补充覆盖：
     - chord 前缀匹配与精确匹配的优先级
     - `null` unbind 对 chord 前缀的影响
     - alt/meta 等价性边界
     - Escape 的 meta 清除逻辑
