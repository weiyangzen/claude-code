# fullscreen.ts 深度研究文档

## 场景与职责

`fullscreen.ts` 负责 Claude Code CLI 的全屏模式（alt-screen）检测和配置，主要解决以下问题：

1. **终端环境检测**：检测是否运行在 tmux -CC（iTerm2 集成模式）等特殊终端环境
2. **全屏模式开关**：根据环境变量和终端类型决定是否启用全屏模式
3. **鼠标支持配置**：控制鼠标追踪和点击处理的启用/禁用
4. **tmux 用户提示**：为 tmux 用户提供鼠标配置建议

该模块是终端 UI 渲染的关键配置源，影响 React 组件的渲染模式选择。

## 功能点目的

### 1. tmux -CC 模式检测
- 通过环境变量启发式检测（`TMUX` + `TERM_PROGRAM` + `TERM`）
- 通过 `tmux display-message -p '#{client_control_mode}'` 同步探测确认
- 在 tmux -CC 模式下自动禁用全屏（alt-screen + 鼠标追踪会导致终端状态损坏）

### 2. 全屏模式配置
- `CLAUDE_CODE_NO_FLICKER` 环境变量控制
- Ant 用户默认启用，外部用户默认禁用
- 支持显式启用/禁用覆盖

### 3. 鼠标支持配置
- `CLAUDE_CODE_DISABLE_MOUSE` - 禁用鼠标追踪（保留键盘滚动）
- `CLAUDE_CODE_DISABLE_MOUSE_CLICKS` - 禁用鼠标点击（保留滚轮）

### 4. tmux 鼠标提示
- 检测 tmux 的 `mouse` 选项状态
- 在鼠标禁用时提供配置建议

## 具体技术实现

### 关键数据结构

```typescript
// 模块级缓存状态
let tmuxControlModeProbed: boolean | undefined  // undefined = 未探测
let loggedTmuxCcDisable = false  // 防止重复日志
let checkedTmuxMouseHint = false  // 单次提示标志
```

### 关键流程

#### tmux -CC 检测流程
1. **环境启发式** (`isTmuxControlModeEnvHeuristic`):
   - 检查 `TMUX` 环境变量是否存在
   - 检查 `TERM_PROGRAM` 是否为 `iTerm.app`
   - 检查 `TERM` 是否不以 `screen` 或 `tmux` 开头

2. **同步探测** (`probeTmuxControlModeSync`):
   - 使用 `spawnSync` 执行 `tmux display-message -p '#{client_control_mode}'`
   - 超时 2000ms
   - 缓存结果避免重复探测
   - 仅在 `TMUX` 设置且 `TERM_PROGRAM` 未设置时探测（SSH 场景）

#### 全屏启用判断流程
1. 检查 `CLAUDE_CODE_NO_FLICKER` 是否为 falsy（显式禁用）
2. 检查 `CLAUDE_CODE_NO_FLICKER` 是否为 truthy（显式启用）
3. 检查是否为 tmux -CC 模式（自动禁用）
4. 根据 `USER_TYPE` 决定默认值

### 环境变量

| 变量 | 值 | 行为 |
|------|-----|------|
| `CLAUDE_CODE_NO_FLICKER` | `0`, `false`, `no`, `off` | 显式禁用全屏 |
| `CLAUDE_CODE_NO_FLICKER` | `1`, `true`, `yes`, `on` | 显式启用全屏 |
| `CLAUDE_CODE_NO_FLICKER` | 未设置 | 自动检测 |
| `CLAUDE_CODE_DISABLE_MOUSE` | truthy | 禁用鼠标追踪 |
| `CLAUDE_CODE_DISABLE_MOUSE_CLICKS` | truthy | 禁用鼠标点击 |

## 关键代码路径与文件引用

### 核心导出
- `isTmuxControlMode()` - 检测是否在 tmux -CC 模式
- `isFullscreenEnvEnabled()` - 检查全屏是否启用
- `isMouseTrackingEnabled()` - 检查鼠标追踪是否启用
- `isMouseClicksDisabled()` - 检查鼠标点击是否禁用
- `isFullscreenActive()` - 检查全屏是否实际激活（需要交互式 REPL）
- `maybeGetTmuxMouseHint()` - 获取 tmux 鼠标提示（异步）
- `_resetTmuxControlModeProbeForTesting()` - 测试重置
- `_resetForTesting()` - 测试重置

### 依赖关系

**被以下模块导入**（20+ 个）：
- `src/ink/styles.ts` - Ink 样式配置
- `src/ink/output.ts` - 输出渲染
- `src/screens/REPL.tsx` - 主 REPL 界面
- `src/ink/components/App.tsx` - 应用组件
- `src/hooks/useTextInput.ts` - 文本输入处理
- `src/components/StructuredDiff.tsx` - 差异显示
- `src/components/FullscreenLayout.tsx` - 全屏布局
- `src/components/StatusLine.tsx` - 状态栏
- `src/components/PromptInput/PromptInput.tsx` - 提示输入
- ... 等

**依赖的模块**：
- `src/bootstrap/state.ts` - `getIsInteractive()`
- `src/utils/debug.ts` - `logForDebugging()`
- `src/utils/envUtils.ts` - `isEnvDefinedFalsy()`, `isEnvTruthy()`
- `src/utils/execFileNoThrow.ts` - `execFileNoThrow()`

### 文件位置
- 源码：`src/utils/fullscreen.ts` (202 行)

## 依赖与外部交互

### Node.js 内置模块
- `child_process` - `spawnSync` 用于 tmux 探测

### 项目内部依赖
- `src/bootstrap/state.ts` - 获取交互式状态
- `src/utils/debug.ts` - 调试日志
- `src/utils/envUtils.ts` - 环境变量解析
- `src/utils/execFileNoThrow.ts` - 安全的命令执行

### 外部依赖
- 无直接外部依赖

## 风险、边界与改进建议

### 已知风险

1. **同步探测阻塞**：`probeTmuxControlModeSync` 使用 `spawnSync`，可能阻塞事件循环约 5ms
2. **环境变量竞争**：`USER_TYPE` 是构建时定义的，无法在运行时更改
3. **tmux 版本兼容**：`#{client_control_mode}` 格式变量需要 tmux 2.4+

### 边界情况

1. **SSH 场景**：`TERM_PROGRAM` 通常不会通过 SSH 传播，需要依赖同步探测
2. **多 tmux 会话**：`display-message` 可能查询到错误的会话
3. **tmux 未安装**：`spawnSync` 可能抛出 ENOENT，已被捕获处理

### 改进建议

1. **异步初始化**：考虑将 tmux 探测改为异步，避免启动时的同步阻塞
2. **配置持久化**：考虑将用户选择持久化到配置文件
3. **更多终端支持**：扩展对其他终端集成模式的支持（如 kitty、wezterm）
4. **动态切换**：支持运行时通过命令动态切换全屏模式
5. **测试覆盖**：当前没有专门的测试文件，建议添加单元测试

### 相关 Issue

- 注释中提到 `coder-tmux` 场景：SSH → tmux -CC 时 `TERM_PROGRAM` 未传播
- 注释中提到 iTerm2 集成模式的鼠标滚轮问题
- GitHub issue 参考：`anthropics/claude-code#30924`（Bun/Windows 相关问题）
