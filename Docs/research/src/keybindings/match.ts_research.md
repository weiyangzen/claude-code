# match.ts 研究文档

## 场景与职责

`match.ts` 是 Claude Code 键盘快捷键系统的核心匹配模块，负责将 Ink 输入事件（`input` + `Key`）与解析后的按键绑定（`ParsedKeystroke`）进行比对。它是按键解析链的最底层，直接处理终端输入的原始数据。

**核心职责：**
1. **键名提取**：从 Ink 的 `Key` 对象和 `input` 字符串提取规范化键名
2. **修饰符匹配**：比较 Ink 修饰符状态与绑定配置的修饰符要求
3. **按键匹配**：判断单个按键是否匹配目标 `ParsedKeystroke`
4. **绑定匹配**：判断按键是否匹配单键绑定的第一个（也是唯一一个）和弦元素

**架构位置：**
```
Ink Input Event
    ↓
useInput handler (useKeybinding.ts / ChordInterceptor)
    ↓
resolveKeyWithChordState (resolver.ts)
    ↓
matchesKeystroke / matchesBinding (本文件)
    ↓
Action Handler
```

---

## 功能点目的

### 1. 键名提取（getKeyName）

**功能：** 将 Ink 的 `Key` 对象映射为内部键名格式。

**映射表：**
| Ink Key 属性 | 内部键名 |
|--------------|----------|
| `key.escape` | `'escape'` |
| `key.return` | `'enter'` |
| `key.tab` | `'tab'` |
| `key.backspace` | `'backspace'` |
| `key.delete` | `'delete'` |
| `key.upArrow` | `'up'` |
| `key.downArrow` | `'down'` |
| `key.leftArrow` | `'left'` |
| `key.rightArrow` | `'right'` |
| `key.pageUp` | `'pageup'` |
| `key.pageDown` | `'pagedown'` |
| `key.wheelUp` | `'wheelup'` |
| `key.wheelDown` | `'wheeldown'` |
| `key.home` | `'home'` |
| `key.end` | `'end'` |
| 单字符 `input` | 小写字符 |
| 其他 | `null` |

**特殊处理：**
- 单字符输入转为小写，确保大小写不敏感匹配
- 未识别的键返回 `null`

### 2. 修饰符匹配（modifiersMatch）

**Ink 修饰符：**
```typescript
type InkModifiers = Pick<Key, 'ctrl' | 'shift' | 'meta' | 'super'>
```

**匹配规则：**

| 修饰符 | Ink 来源 | 匹配逻辑 |
|--------|----------|----------|
| `ctrl` | `key.ctrl` | 直接比较 |
| `shift` | `key.shift` | 直接比较 |
| `alt` / `meta` | `key.meta` | **合并检查** - 任一要求都匹配 `key.meta` |
| `super` | `key.super` | 直接比较 |

**Alt/Meta 合并原因：**
- Ink 历史上将 Alt/Option 键设置为 `key.meta`
- 终端无法区分 Alt 和 Meta（Option）
- 配置中的 `alt` 或 `meta` 都匹配 `key.meta === true`

**Super（Cmd/Win）：**
- 仅通过 Kitty 键盘协议到达
- 与 Alt/Meta 完全独立
- 在不支持 Kitty 协议的终端上永远不会触发

### 3. 按键匹配（matchesKeystroke）

**匹配流程：**
1. 提取键名（`getKeyName`）
2. 键名不匹配 → 返回 `false`
3. 提取 Ink 修饰符（`getInkModifiers`）
4. **特殊处理 Escape 键：**
   - Ink 设置 `key.meta = true` 当 Escape 被按下（遗留行为）
   - 匹配时忽略 `meta` 修饰符
5. 比较修饰符（`modifiersMatch`）

**Escape 键特殊处理：**
```typescript
if (key.escape) {
  return modifiersMatch({ ...inkMods, meta: false }, target)
}
```

**原因：**
- 如果不忽略 `meta`，纯 `"escape"` 绑定永远不会匹配
- 因为 Ink 总是为 Escape 设置 `meta: true`

### 4. 绑定匹配（matchesBinding）

**限制：**
- 仅支持单键绑定（`binding.chord.length === 1`）
- 用于 Phase 1 实现（单键绑定优先）

**流程：**
1. 检查和弦长度是否为 1
2. 获取第一个（唯一）和弦元素
3. 调用 `matchesKeystroke` 进行匹配

---

## 具体技术实现

### 1. 修饰符提取

```typescript
function getInkModifiers(key: Key): InkModifiers {
  return {
    ctrl: key.ctrl,
    shift: key.shift,
    meta: key.meta,
    super: key.super,
  }
}
```

**注意：** `fn` 修饰符被有意排除，因为它很少使用且终端应用通常不可配置。

### 2. 修饰符匹配实现

```typescript
function modifiersMatch(inkMods: InkModifiers, target: ParsedKeystroke): boolean {
  // 1. 检查 ctrl
  if (inkMods.ctrl !== target.ctrl) return false
  
  // 2. 检查 shift
  if (inkMods.shift !== target.shift) return false
  
  // 3. 检查 alt/meta（合并）
  const targetNeedsMeta = target.alt || target.meta
  if (inkMods.meta !== targetNeedsMeta) return false
  
  // 4. 检查 super
  if (inkMods.super !== target.super) return false
  
  return true
}
```

### 3. 按键匹配实现

```typescript
export function matchesKeystroke(
  input: string,
  key: Key,
  target: ParsedKeystroke,
): boolean {
  // 1. 键名匹配
  const keyName = getKeyName(input, key)
  if (keyName !== target.key) return false
  
  // 2. 提取修饰符
  const inkMods = getInkModifiers(key)
  
  // 3. Escape 键特殊处理
  if (key.escape) {
    return modifiersMatch({ ...inkMods, meta: false }, target)
  }
  
  // 4. 正常修饰符匹配
  return modifiersMatch(inkMods, target)
}
```

### 4. 绑定匹配实现

```typescript
export function matchesBinding(
  input: string,
  key: Key,
  binding: ParsedBinding,
): boolean {
  // 仅支持单键绑定
  if (binding.chord.length !== 1) return false
  
  const keystroke = binding.chord[0]
  if (!keystroke) return false
  
  return matchesKeystroke(input, key, keystroke)
}
```

---

## 关键代码路径与文件引用

### 核心类型定义

```typescript
// 来自 ../ink.js
type Key = {
  upArrow: boolean
  downArrow: boolean
  leftArrow: boolean
  rightArrow: boolean
  pageDown: boolean
  pageUp: boolean
  wheelUp: boolean
  wheelDown: boolean
  home: boolean
  end: boolean
  return: boolean
  escape: boolean
  ctrl: boolean
  shift: boolean
  fn: boolean
  tab: boolean
  backspace: boolean
  delete: boolean
  meta: boolean
  super: boolean
}

// 来自 ./types.js（推断）
interface ParsedKeystroke {
  key: string
  ctrl: boolean
  alt: boolean
  shift: boolean
  meta: boolean
  super: boolean
}

interface ParsedBinding {
  chord: ParsedKeystroke[]
  action: string | null
  context: KeybindingContextName
}
```

### 依赖文件

| 文件 | 导入内容 | 用途 |
|------|----------|------|
| `../ink.js` | `type Key` | Ink 键盘事件类型 |
| `./types.js` | `ParsedBinding`, `ParsedKeystroke` | 绑定类型定义 |

### 导出符号

```typescript
export { getKeyName }         // 键名提取
export { matchesKeystroke }   // 按键匹配
export { matchesBinding }     // 绑定匹配（单键）
```

### 消费方

| 文件 | 使用函数 | 用途 |
|------|----------|------|
| `resolver.ts` | `getKeyName`, `matchesBinding` | 按键解析 |
| `resolver.ts` | `matchesKeystroke`（通过 keystrokesEqual） | 和弦匹配 |

---

## 依赖与外部交互

### 1. 上游依赖（输入）

**来自 Ink：**
- `input: string` - 字符输入（如 `"a"`, `"\r"`, `"\u001b"`）
- `key: Key` - 修饰符和特殊键状态

**来自绑定配置：**
- `ParsedKeystroke` - 目标按键定义
- `ParsedBinding` - 完整绑定定义

### 2. 下游输出（消费）

**被 resolver.ts 消费：**
```typescript
import { getKeyName, matchesBinding } from './match.js'

// 在 resolveKey() 中使用
if (matchesBinding(input, key, binding)) {
  match = binding
}

// 在 buildKeystroke() 中使用
const keyName = getKeyName(input, key)
```

### 3. 与 Ink 的交互

**输入事件处理链：**
```
终端输入 → Ink parseKeypress → Key 对象 + input 字符串
                                      ↓
                              matchesKeystroke / matchesBinding
                                      ↓
                              匹配结果（true/false）
```

**特殊键处理：**
- Escape 键：`key.escape = true`, `key.meta = true`
- 方向键：`key.upArrow = true` 等
- 功能键：`key.pageUp = true` 等
- 修饰键：反映在 `key.ctrl`, `key.shift`, `key.meta`, `key.super`

---

## 风险、边界与改进建议

### 1. 已知风险

**Escape 键特殊处理：**
- 硬编码忽略 `meta` 修饰符
- 如果 Ink 行为改变，匹配逻辑需要更新

**Alt/Meta 合并：**
- 终端无法区分 Alt 和 Meta
- 配置 `"alt+k"` 和 `"meta+k"` 实际上是相同的
- 可能导致用户困惑

**大小写敏感：**
- `getKeyName` 将单字符转为小写
- `"Shift+A"` 和 `"Shift+a"` 都匹配 `"shift+a"`
- 但 `"A"` 单独输入（无 shift）也会匹配 `"a"`

### 2. 边界情况

**未识别键：**
- `getKeyName` 返回 `null`
- 上层调用者（resolver）需要处理

**空和弦：**
- `matchesBinding` 检查 `binding.chord[0]` 是否存在
- 防御性编程，理论上不应发生

**多键和弦：**
- `matchesBinding` 明确返回 `false`
- 多键和弦由 resolver 的 `chordExactlyMatches` 处理

**修饰符-only 输入：**
- 如纯 `ctrl` 按下（无字符）
- `input` 可能为空或控制字符
- 依赖 Ink 的 `key.ctrl` 标志

### 3. 改进建议

**代码清晰度：**
```typescript
// 建议：添加注释解释为什么忽略 meta
if (key.escape) {
  // Ink sets meta=true for escape (legacy terminal behavior)
  // We must ignore it to match plain "escape" bindings
  return modifiersMatch({ ...inkMods, meta: false }, target)
}
```

**类型安全：**
- 当前 `ParsedKeystroke` 类型从 `./types.js` 导入
- 建议内联定义或添加更严格的类型约束

**测试覆盖：**
- 添加单元测试覆盖所有键名映射
- 测试修饰符组合（ctrl+shift, alt+meta 等）
- 测试 Escape 键特殊处理

**性能优化：**
- 当前每次匹配都调用 `getKeyName` 和 `getInkModifiers`
- 考虑在调用方缓存结果（如果多次匹配同一按键）

### 4. 终端兼容性

**Kitty 键盘协议：**
- 支持 `super` 修饰符
- 更多键可以被识别

**传统终端：**
- `super` 永远不会被设置
- 某些组合键可能无法区分

**Windows Terminal：**
- VT 模式影响修饰键报告
- 参见 `defaultBindings.ts` 中的 `SUPPORTS_TERMINAL_VT_MODE`

### 5. 扩展考虑

**多键和弦支持：**
- 当前 `matchesBinding` 仅支持单键
- 多键和弦在 resolver 中单独处理
- 考虑统一匹配逻辑

**正则匹配：**
- 当前是精确匹配
- 未来可能支持通配符（如 `"ctrl+*"`）

**时序敏感：**
- 当前匹配是状态less的
- 未来可能需要考虑按键时序（双击等）
