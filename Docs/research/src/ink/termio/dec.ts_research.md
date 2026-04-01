# dec.ts 研究报告

## 场景与职责

`dec.ts` 实现了 **DEC (Digital Equipment Corporation) 私有模式序列**。DEC 私有模式是 ANSI 标准的扩展，使用 `CSI ? <模式号> h`（设置）和 `CSI ? <模式号> l`（重置）格式，提供终端特定的增强功能。

该文件的核心职责：
1. **定义 DEC 私有模式号常量** —— 光标可见性、备用屏幕、鼠标跟踪、焦点事件等
2. **生成 DEC 序列** —— 提供 `decset()` 和 `decreset()` 函数
3. **预生成常用序列** —— 同步更新、粘贴括号、焦点事件、光标显示/隐藏等

## 功能点目的

### 1. DEC 私有模式号定义

```typescript
export const DEC = {
  CURSOR_VISIBLE: 25,        // 光标可见性
  ALT_SCREEN: 47,            // 备用屏幕（无清屏）
  ALT_SCREEN_CLEAR: 1049,    // 备用屏幕（带清屏）- 推荐
  MOUSE_NORMAL: 1000,        // 鼠标按钮按下/释放/滚轮
  MOUSE_BUTTON: 1002,        // 鼠标按钮拖动事件
  MOUSE_ANY: 1003,           // 鼠标任意移动（悬停）
  MOUSE_SGR: 1006,           // SGR 格式鼠标报告（推荐）
  FOCUS_EVENTS: 1004,        // 焦点进入/离开事件
  BRACKETED_PASTE: 2004,     // 粘贴括号模式
  SYNCHRONIZED_UPDATE: 2026, // 同步更新（原子渲染）
} as const
```

### 2. 序列生成函数

```typescript
// 生成设置模式序列: CSI ? <mode> h
export function decset(mode: number): string {
  return csi(`?${mode}h`)
}

// 生成重置模式序列: CSI ? <mode> l
export function decreset(mode: number): string {
  return csi(`?${mode}l`)
}
```

### 3. 预生成序列常量

| 常量 | 序列 | 说明 |
|------|------|------|
| `BSU` | `\x1b[?2026h` | 开始同步更新 (Begin Synchronized Update) |
| `ESU` | `\x1b[?2026l` | 结束同步更新 (End Synchronized Update) |
| `EBP` | `\x1b[?2004h` | 启用粘贴括号 (Enable Bracketed Paste) |
| `DBP` | `\x1b[?2004l` | 禁用粘贴括号 (Disable Bracketed Paste) |
| `EFE` | `\x1b[?1004h` | 启用焦点事件 (Enable Focus Events) |
| `DFE` | `\x1b[?1004l` | 禁用焦点事件 (Disable Focus Events) |
| `SHOW_CURSOR` | `\x1b[?25h` | 显示光标 |
| `HIDE_CURSOR` | `\x1b[?25l` | 隐藏光标 |
| `ENTER_ALT_SCREEN` | `\x1b[?1049h` | 进入备用屏幕（清屏）|
| `EXIT_ALT_SCREEN` | `\x1b[?1049l` | 退出备用屏幕 |
| `ENABLE_MOUSE_TRACKING` | 组合序列 | 启用完整鼠标跟踪 |
| `DISABLE_MOUSE_TRACKING` | 组合序列 | 禁用鼠标跟踪 |

### 4. 鼠标跟踪组合序列

```typescript
// 启用：组合 1000 + 1002 + 1003 + 1006
// - 1000: 按钮按下/释放/滚轮
// - 1002: 按钮拖动（按住移动）
// - 1003: 任意移动（悬停）
// - 1006: SGR 格式（CSI < btn;col;row M/m）替代传统 X10 字节
export const ENABLE_MOUSE_TRACKING =
  decset(DEC.MOUSE_NORMAL) +
  decset(DEC.MOUSE_BUTTON) +
  decset(DEC.MOUSE_ANY) +
  decset(DEC.MOUSE_SGR)

// 禁用：按相反顺序重置
export const DISABLE_MOUSE_TRACKING =
  decreset(DEC.MOUSE_SGR) +
  decreset(DEC.MOUSE_ANY) +
  decreset(DEC.MOUSE_BUTTON) +
  decreset(DEC.MOUSE_NORMAL)
```

## 具体技术实现

### 序列格式

DEC 私有模式序列使用 `?` 前缀区分于标准 CSI 序列：

```
标准 CSI: ESC [ <params> <final>
DEC 私有: ESC [ ? <params> <final>
          └─ 私有标记

示例:
  ESC [ ? 25 h  →  显示光标
  ESC [ ? 25 l  →  隐藏光标
  ESC [ ? 1049 h  →  进入备用屏幕
```

### 模式号说明

| 模式号 | 名称 | 详细说明 |
|--------|------|----------|
| 25 | DECTCEM | 文本光标启用模式 (Text Cursor Enable Mode) |
| 47 | 备用屏幕 | 切换到备用屏幕缓冲区，不清除内容 |
| 1049 | 备用屏幕+清屏 | 切换到备用屏幕并清除，退出时恢复 |
| 1000 | X10 鼠标 | 报告按钮按下和滚轮事件 |
| 1002 | 按钮事件 | 报告拖动事件（按住按钮移动）|
| 1003 | 任意事件 | 报告所有鼠标移动（包括悬停）|
| 1006 | SGR 鼠标 | 使用 SGR 格式报告（支持更大坐标）|
| 1004 | 焦点事件 | 终端获得/失去焦点时发送 CSI I / CSI O |
| 2004 | 粘贴括号 | 粘贴内容前后发送 CSI 200 ~ / CSI 201 ~ |
| 2026 | 同步更新 | 原子渲染，防止闪烁 |

## 关键代码路径与文件引用

### 依赖

| 被导入 | 来源 | 用途 |
|--------|------|------|
| `csi` | `./csi.js` | 构建 CSI 序列基础 |

### 被导入方

| 导入方 | 导入内容 | 用途 |
|--------|----------|------|
| `parser.ts` | `DEC` | 解析 DEC 私有模式序列 |
| `terminal.ts` | `BSU`, `ESU`, `HIDE_CURSOR`, `SHOW_CURSOR` | 终端输出同步和光标控制 |
| `AlternateScreen.tsx` | `ENTER_ALT_SCREEN`, `EXIT_ALT_SCREEN` | 备用屏幕切换 |
| `App.tsx` | `ENABLE_MOUSE_TRACKING`, `DISABLE_MOUSE_TRACKING` | 鼠标跟踪控制 |

### 导出内容

```typescript
// DEC 模式号常量
export const DEC

// 序列生成函数
export function decset(mode: number): string
export function decreset(mode: number): string

// 预生成序列
export const BSU, ESU                    // 同步更新
export const EBP, DBP                    // 粘贴括号
export const EFE, DFE                    // 焦点事件
export const SHOW_CURSOR, HIDE_CURSOR    // 光标可见性
export const ENTER_ALT_SCREEN, EXIT_ALT_SCREEN  // 备用屏幕
export const ENABLE_MOUSE_TRACKING, DISABLE_MOUSE_TRACKING  // 鼠标跟踪
```

## 依赖与外部交互

### 内部依赖

- `csi.ts`：使用 `csi()` 函数构建基础 CSI 序列

### 外部使用场景

1. **终端初始化**：`App.tsx` 启用鼠标跟踪和焦点事件
2. **备用屏幕**：`AlternateScreen.tsx` 切换主/备用屏幕
3. **渲染同步**：`terminal.ts` 使用 BSU/ESU 包装输出防止闪烁
4. **输入解析**：`parser.ts` 识别 DEC 模式设置/重置序列

### 终端兼容性

| 模式 | 支持终端 |
|------|----------|
| 25 (光标) | 几乎所有终端 |
| 47/1049 (备用屏幕) | 几乎所有终端 |
| 1000-1006 (鼠标) | iTerm2, kitty, GNOME Terminal, Windows Terminal 等 |
| 1004 (焦点) | iTerm2, kitty, tmux 等 |
| 2004 (粘贴括号) | 现代终端广泛支持 |
| 2026 (同步更新) | iTerm2, kitty, WezTerm, Ghostty, Alacritty 等 |

## 风险、边界与改进建议

### 边界情况

1. **tmux 特殊处理**：`terminal.ts` 中 `isSynchronizedOutputSupported()` 明确排除 tmux，因为 tmux 会解析 BSU/ESU 但不实现原子性
2. **鼠标模式堆叠**：启用多个鼠标模式（1000+1002+1003）是累加的，不是替换的
3. **备用屏幕恢复**：模式 1049 退出时会自动恢复光标位置和屏幕内容

### 风险

1. **序列顺序**：`DISABLE_MOUSE_TRACKING` 使用与启用相反的顺序，这是有意为之（先禁用高级功能）
2. **无验证**：`decset/decreset` 不验证模式号是否有效
3. **硬编码模式号**：模式号是 DEC 标准的一部分，但不同终端可能有扩展

### 改进建议

1. **添加模式验证**：
   ```typescript
   const VALID_MODES = new Set([25, 47, 1049, 1000, 1002, 1003, 1006, 1004, 2004, 2026])
   export function decset(mode: number): string {
     if (!VALID_MODES.has(mode)) console.warn(`Unknown DEC mode: ${mode}`)
     return csi(`?${mode}h`)
   }
   ```

2. **添加模式查询支持**：
   ```typescript
   // DECRQM - 请求模式状态
   export function decrqm(mode: number): string {
     return csi(`?${mode}$p`)
   }
   ```

3. **添加更多常用模式**：
   - 模式 7 (DECAWM)：自动换行
   - 模式 12 (DECSCLM)：本地回显
   - 模式 45 (DECNRCM)：国家替换字符集

4. **文档改进**：
   - 添加每个模式的终端兼容性矩阵
   - 添加模式交互说明（如 1006 需要 1000/1002/1003 之一）

### 测试建议

- 验证生成的序列格式正确（`ESC [ ? <n> h/l`）
- 测试组合序列的顺序
- 验证模式号常量值符合 DEC 标准
- 测试与各种终端的兼容性
