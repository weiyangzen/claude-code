# detection.ts 深度研究文档

## 场景与职责

detection.ts 是 Agent Swarm 后端系统的 **环境检测模块**，负责检测当前运行环境（tmux/iTerm2）和后端可用性。它是整个后端选择逻辑的基础，为 registry.ts 提供决策依据。

**核心定位：**
- 环境检测的单一可信源
- 模块加载时捕获原始环境状态（避免后续被覆盖）
- 提供同步和异步两种检测接口
- 缓存检测结果以提高性能

**检测目标：**
1. 是否在 tmux 会话中
2. 是否在 iTerm2 终端中
3. tmux 是否已安装
4. it2 CLI 是否可用

---

## 功能点目的

### 1. Tmux 环境检测
- `isInsideTmuxSync()`: 同步检测是否在 tmux 中
- `isInsideTmux()`: 异步检测（带缓存）
- `getLeaderPaneId()`: 获取 leader 的 tmux pane ID

### 2. iTerm2 环境检测
- `isInITerm2()`: 检测是否在 iTerm2 终端中（带缓存）
- 多指标检测：TERM_PROGRAM、ITERM_SESSION_ID、env.terminal

### 3. 后端可用性检测
- `isTmuxAvailable()`: 检测 tmux 是否已安装
- `isIt2CliAvailable()`: 检测 it2 CLI 是否可用且 Python API 已启用

### 4. 缓存管理
- `resetDetectionCache()`: 重置所有缓存（用于测试）

---

## 具体技术实现

### 关键数据结构

```typescript
// 模块加载时捕获原始环境（关键！）
const ORIGINAL_USER_TMUX = process.env.TMUX           // 原始 TMUX 值
const ORIGINAL_TMUX_PANE = process.env.TMUX_PANE      // 原始 pane ID

// 缓存
let isInsideTmuxCached: boolean | null = null
let isInITerm2Cached: boolean | null = null
```

### 核心流程

#### 1. Tmux 内部检测（isInsideTmux）

```typescript
export async function isInsideTmux(): Promise<boolean> {
  if (isInsideTmuxCached !== null) {
    return isInsideTmuxCached
  }
  
  // 只检查模块加载时捕获的 TMUX 值
  // 不使用 tmux display-message 作为回退！
  isInsideTmuxCached = !!ORIGINAL_USER_TMUX
  return isInsideTmuxCached
}
```

**重要设计决策：**

为什么不使用 `tmux display-message` 作为回退？

```typescript
// 这段代码故意不存在：
// const result = await execFileNoThrow('tmux', ['display-message', '-p', '#{pane_id}'])
// if (result.code === 0) return true
```

原因：`tmux display-message` 在**任何 tmux 服务器运行时**都会成功，即使当前进程不在 tmux 会话中。这会导致错误地判断为"在 tmux 中"。

**正确检测逻辑：**
- TMUX 环境变量仅在进程在 tmux 会话中时才设置
- 这是检测当前进程是否在 tmux 中的唯一可靠方法

#### 2. Leader Pane ID 获取

```typescript
export function getLeaderPaneId(): string | null {
  return ORIGINAL_TMUX_PANE || null
}
```

**为什么使用模块加载时捕获的值？**

`Shell.ts` 可能在初始化 Claude socket 时覆盖 `TMUX` 环境变量。使用模块加载时捕获的值确保：
- 始终获取 leader 启动时的原始 pane ID
- 即使用户在 tmux 中切换了窗口，也能正确定位 leader

#### 3. iTerm2 检测（isInITerm2）

```typescript
export function isInITerm2(): boolean {
  if (isInITerm2Cached !== null) {
    return isInITerm2Cached
  }
  
  // 多指标检测
  const termProgram = process.env.TERM_PROGRAM        // "iTerm.app"
  const hasItermSessionId = !!process.env.ITERM_SESSION_ID
  const terminalIsITerm = env.terminal === 'iTerm.app'
  
  isInITerm2Cached = termProgram === 'iTerm.app' || 
                     hasItermSessionId || 
                     terminalIsITerm
  
  return isInITerm2Cached
}
```

**检测指标：**

| 指标 | 来源 | 可靠性 |
|-----|------|-------|
| `TERM_PROGRAM` | 环境变量 | 高 |
| `ITERM_SESSION_ID` | 环境变量 | 高 |
| `env.terminal` | utils/env.ts | 中（依赖检测逻辑） |

#### 4. Tmux 可用性检测（isTmuxAvailable）

```typescript
export async function isTmuxAvailable(): Promise<boolean> {
  const result = await execFileNoThrow(TMUX_COMMAND, ['-V'])
  return result.code === 0
}
```

简单检测 tmux 是否在 PATH 中且可执行。

#### 5. it2 CLI 可用性检测（isIt2CliAvailable）

```typescript
export async function isIt2CliAvailable(): Promise<boolean> {
  // 使用 'session list' 而非 '--version'
  // 因为 --version 在 Python API 禁用时也会成功
  const result = await execFileNoThrow(IT2_COMMAND, ['session', 'list'])
  return result.code === 0
}
```

**重要设计决策：**

为什么使用 `it2 session list` 而非 `it2 --version`？

- `it2 --version` 即使 Python API 被禁用时也会成功
- `it2 session list` 需要实际连接到 iTerm2，因此可以检测 Python API 是否启用
- 这避免了后续 `session split` 命令失败的尴尬

---

## 关键代码路径与文件引用

### 内部依赖

| 文件路径 | 用途 |
|---------|------|
| `src/utils/env.ts` | `env.terminal` 检测 |
| `src/utils/execFileNoThrow.ts` | 安全执行外部命令 |
| `src/utils/swarm/constants.ts` | `TMUX_COMMAND` |

### 外部命令

| 命令 | 用途 |
|-----|------|
| `tmux -V` | 检测 tmux 是否安装 |
| `it2 session list` | 检测 it2 CLI 和 Python API 可用性 |

### 关键代码位置

- **原始环境捕获**: 行 9-19
- **Tmux 检测**: 行 36-60
- **Leader Pane ID**: 行 66-68
- **Tmux 可用性**: 行 73-76
- **iTerm2 检测**: 行 90-104
- **it2 可用性**: 行 117-120
- **缓存重置**: 行 125-128

---

## 依赖与外部交互

### 与 Registry 的交互

detection.ts 被 `registry.ts` 大量调用：

```typescript
// registry.ts
const insideTmux = await isInsideTmux()
const inITerm2 = isInITerm2()
const it2Available = await isIt2CliAvailable()
const tmuxAvailable = await isTmuxAvailable()
```

### 与 TmuxBackend 的交互

```typescript
// TmuxBackend.ts
import { getLeaderPaneId, isInsideTmux, isTmuxAvailable } from './detection.js'

// 使用 getLeaderPaneId 确保始终定位到 leader 的原始 pane
const leaderPane = getLeaderPaneId()
```

### 与 ITermBackend 的交互

```typescript
// ITermBackend.ts
import { isInITerm2, isIt2CliAvailable } from './detection.js'
```

---

## 风险、边界与改进建议

### 已知风险

1. **环境变量被覆盖**
   - `Shell.ts` 可能覆盖 `TMUX` 环境变量
   - **缓解**: 模块加载时立即捕获原始值

2. **缓存不一致**
   - 环境在进程运行时理论上不会改变
   - 但如果发生（如 tmux 附加/分离），缓存会过时
   - **缓解**: 环境检测通常只在启动时进行

3. **误检测风险**
   - `isInITerm2` 使用多个指标，但仍有误检测可能
   - 例如：在 tmux 中运行，但 tmux 在 iTerm2 中
   - **缓解**: registry.ts 的优先级逻辑（tmux 优先于 iTerm2）

4. **it2 检测延迟**
   - `it2 session list` 需要连接到 iTerm2
   - 如果 iTerm2 繁忙，可能超时
   - **影响**: 后端检测延迟

### 边界情况

| 场景 | 处理方式 |
|-----|---------|
| TMUX 为空字符串 | 视为不在 tmux 中（`!!'' === false`） |
| TMUX_PANE 未设置 | 返回 null |
| it2 未安装 | 返回 false |
| Python API 禁用 | `session list` 失败，返回 false |
| 多指标冲突 | iTerm2 检测使用 OR 逻辑 |

### 改进建议

1. **更精确的 iTerm2 检测**
   - 添加更多指标（如 `LC_TERMINAL`）
   - 考虑检测是否在 tmux 中运行的 iTerm2

2. **超时控制**
   - 为 `it2 session list` 添加超时
   - 防止检测过程阻塞

3. **缓存策略优化**
   - 考虑添加缓存过期机制
   - 或提供强制刷新接口

4. **检测顺序优化**
   - 当前并行检测可能更高效
   - 但顺序检测有助于调试

5. **更好的错误信息**
   - 区分 "it2 未安装" 和 "Python API 禁用"
   - 帮助用户快速定位问题

6. **单元测试覆盖**
   - 测试各种环境变量组合
   - 模拟 execFileNoThrow 的不同返回值
