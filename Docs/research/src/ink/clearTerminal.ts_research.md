# clearTerminal.ts 研究文档

## 场景与职责

`clearTerminal.ts` 提供跨平台的终端清屏功能，支持清除屏幕内容和滚动缓冲区（scrollback）。

### 核心职责

1. **跨平台清屏**：处理 Windows、macOS、Linux 等不同平台的终端差异
2. **滚动缓冲区清除**：在现代终端上清除滚动历史（scrollback）
3. **终端能力检测**：自动检测终端支持的清屏能力

### 使用场景

- 应用启动前清屏
- 全屏应用（Alternate Screen）退出后的清理
- 需要干净屏幕状态的用户交互场景

## 功能点目的

### 1. 终端类型检测
识别不同类型的终端以选择合适的清屏序列：

- **Windows Terminal**：现代 Windows 终端，支持完整 ANSI 序列
- **mintty**：Git Bash、MSYS2、Cygwin 使用的终端
- **VS Code 集成终端**：基于 xterm.js，支持现代序列
- **传统 Windows 控制台**：conhost，不支持滚动缓冲区清除

### 2. 清屏序列生成
根据终端能力生成适当的 ANSI 转义序列：

- **现代终端**：`ERASE_SCREEN + ERASE_SCROLLBACK + CURSOR_HOME`
- **传统 Windows**：`ERASE_SCREEN + CURSOR_HOME_WINDOWS`

### 3. 常量导出
提供预生成的清屏字符串常量 `clearTerminal`，便于直接使用。

## 具体技术实现

### 终端检测函数

```typescript
// Windows Terminal 检测
function isWindowsTerminal(): boolean {
  return process.platform === 'win32' && !!process.env.WT_SESSION
}

// mintty 检测（Git Bash/MSYS2/Cygwin）
function isMintty(): boolean {
  if (process.env.TERM_PROGRAM === 'mintty') return true
  if (process.platform === 'win32' && process.env.MSYSTEM) return true
  return false
}

// 现代终端检测
function isModernWindowsTerminal(): boolean {
  if (isWindowsTerminal()) return true
  if (process.platform === 'win32' && 
      process.env.TERM_PROGRAM === 'vscode' && 
      process.env.TERM_PROGRAM_VERSION) return true
  if (isMintty()) return true
  return false
}
```

### 清屏序列生成

```typescript
export function getClearTerminalSequence(): string {
  if (process.platform === 'win32') {
    if (isModernWindowsTerminal()) {
      // 现代 Windows 终端：清除屏幕 + 滚动缓冲区 + 光标归位
      return ERASE_SCREEN + ERASE_SCROLLBACK + CURSOR_HOME
    } else {
      // 传统 Windows 控制台：仅清除屏幕，无法清除滚动缓冲区
      return ERASE_SCREEN + CURSOR_HOME_WINDOWS
    }
  }
  // Unix/Linux/macOS：完整清屏
  return ERASE_SCREEN + ERASE_SCROLLBACK + CURSOR_HOME
}
```

### 使用的 CSI 序列

| 序列 | 名称 | 功能 |
|------|------|------|
| `CSI 2 J` | ERASE_SCREEN | 清除整个屏幕 |
| `CSI 3 J` | ERASE_SCROLLBACK | 清除滚动缓冲区 |
| `CSI H` | CURSOR_HOME | 光标移动到左上角 |
| `CSI 0 f` | CURSOR_HOME_WINDOWS | HVP 序列（传统 Windows） |

## 关键代码路径与文件引用

### 入口与导出
- **文件**：`src/ink/clearTerminal.ts`
- **导出常量**：`clearTerminal` - 预生成的清屏序列
- **导出函数**：`getClearTerminalSequence` - 动态生成清屏序列

### 依赖关系

**被导入**：
- `./termio/csi.js` - CSI 序列常量

**导入使用**：
```typescript
import {
  CURSOR_HOME,
  csi,
  ERASE_SCREEN,
  ERASE_SCROLLBACK,
} from './termio/csi.js'
```

### 关键函数

| 函数 | 职责 | 行号 |
|------|------|------|
| `isWindowsTerminal` | 检测 Windows Terminal | 16-18 |
| `isMintty` | 检测 mintty 终端 | 20-30 |
| `isModernWindowsTerminal` | 检测现代终端能力 | 32-53 |
| `getClearTerminalSequence` | 生成清屏序列 | 59-69 |

### 相关文件

- `src/ink/termio/csi.ts` - CSI 序列定义
  - `ERASE_SCREEN` (`CSI 2 J`)
  - `ERASE_SCROLLBACK` (`CSI 3 J`)
  - `CURSOR_HOME` (`CSI H`)
  - `csi()` 函数用于生成序列

## 依赖与外部交互

### 环境变量依赖

| 变量 | 用途 |
|------|------|
| `WT_SESSION` | Windows Terminal 标识 |
| `TERM_PROGRAM` | 终端程序识别（vscode、mintty） |
| `TERM_PROGRAM_VERSION` | VS Code 终端版本验证 |
| `MSYSTEM` | MSYS2/MINGW 环境标识 |
| `process.platform` | 操作系统平台检测 |

### 调用时机

```
应用启动
    ↓
需要清屏的场景
    ↓
getClearTerminalSequence() / clearTerminal
    ↓
stdout.write() 输出序列
    ↓
终端执行清屏操作
```

### 与 termio 的交互

依赖 `termio/csi.ts` 提供的标准 CSI 序列：
- 所有序列符合 ANSI/ECMA-48 标准
- 使用 `csi()` 辅助函数生成格式正确的序列

## 风险、边界与改进建议

### 已知风险

1. **终端检测可靠性**：
   - 依赖环境变量可能被用户修改
   - 某些终端可能模拟其他终端的环境变量
   - WSL 环境检测可能不够精确

2. **序列兼容性**：
   - `CSI 3 J`（清除滚动缓冲区）不是所有终端都支持
   - 不支持的终端会静默忽略该序列

3. **Windows 传统控制台**：
   - 无法清除滚动缓冲区是已知限制
   - 使用 HVP (`CSI 0 f`) 代替 CUP (`CSI H`) 作为光标归位

### 边界情况

1. **未知终端**：默认使用 Unix 风格完整清屏
2. **管道/重定向**：环境变量检测仍然有效，但序列可能被忽略
3. **SSH 会话**：依赖远程终端的环境变量设置

### 改进建议

1. **增强检测**：
   - 添加对更多终端的检测（Alacritty、WezTerm、Ghostty 等）
   - 使用终端查询序列（DA1）动态检测能力
   
   ```typescript
   // 示例：使用终端查询
   function queryTerminalCapabilities(): Promise<TerminalCaps> {
     // 发送 DA1 查询，解析响应
   }
   ```

2. **配置选项**：
   - 允许用户强制指定终端类型
   - 提供 `FORCE_CLEAR_SCROLLBACK` 环境变量覆盖

3. **错误处理**：
   - 添加对无效环境值的警告
   - 记录检测到的终端类型用于调试

4. **性能优化**：
   - 缓存检测结果（当前每次调用都重新检测）
   
   ```typescript
   // 示例：添加缓存
   let cachedSequence: string | undefined
   export function getClearTerminalSequence(): string {
     if (!cachedSequence) {
       cachedSequence = computeSequence()
     }
     return cachedSequence
   }
   ```

5. **测试覆盖**：
   - 添加各种终端模拟器的测试
   - 验证序列格式正确性
   - 测试环境变量边界情况
