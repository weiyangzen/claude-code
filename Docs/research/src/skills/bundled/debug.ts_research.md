# debug.ts 研究文档

## 场景与职责

`debug.ts` 实现了 `/debug` 内置技能，用于帮助用户诊断当前 Claude Code 会话中的问题。对于 Anthropic 内部员工（`USER_TYPE === 'ant'`），该技能直接读取当前会话的 debug 日志并展示最后若干行；对于普通用户，它会在调用时动态开启 debug 日志记录，并提示用户复现问题后再读取日志。

## 功能点目的

1. **动态开启调试日志**：非 ant 用户默认不记录 debug 日志，`/debug` 调用时通过 `enableDebugLogging()` 开启，后续操作才会被捕获。
2. **安全 tail 读取**：避免一次性读取可能无限增长的完整 debug 日志，仅读取尾部最多 64KB，并展示最后 20 行。
3. **设置文件定位**：在 prompt 中注入用户级、项目级、本地级 settings 文件的路径，方便排查配置问题。
4. **引导使用子 agent**：提示模型可考虑启动 `claude-code-guide` 子 agent 来深入理解相关功能。

## 具体技术实现

### 关键流程

- `registerDebugSkill()` → `registerBundledSkill({ name: 'debug', ... })`
- `getPromptForCommand(args)` 入口：
  1. `enableDebugLogging()` 开启运行时调试日志（返回之前是否已在记录）
  2. `getDebugLogPath()` 获取日志路径
  3. `fs.open` + `fd.read` 读取尾部最多 `TAIL_READ_BYTES = 64 * 1024` 字节
  4. 提取最后 `DEFAULT_DEBUG_LINES_READ = 20` 行
  5. 组装包含日志大小、尾部内容、设置文件路径、用户问题描述的 prompt

### 数据结构

```ts
const DEFAULT_DEBUG_LINES_READ = 20
const TAIL_READ_BYTES = 64 * 1024
```

### 核心逻辑片段

```ts
const stats = await stat(debugLogPath)
const readSize = Math.min(stats.size, TAIL_READ_BYTES)
const startOffset = stats.size - readSize
const fd = await open(debugLogPath, 'r')
const { buffer, bytesRead } = await fd.read({
  buffer: Buffer.alloc(readSize),
  position: startOffset,
})
const tail = buffer
  .toString('utf-8', 0, bytesRead)
  .split('\n')
  .slice(-DEFAULT_DEBUG_LINES_READ)
  .join('\n')
```

### 注册参数

| 字段 | 值 |
|------|-----|
| `name` | `'debug'` |
| `allowedTools` | `['Read', 'Grep', 'Glob']` |
| `disableModelInvocation` | `true`（必须由用户显式调用） |
| `userInvocable` | `true` |

## 关键代码路径与文件引用

- 源文件：`src/skills/bundled/debug.ts`
- 注册入口：`src/skills/bundled/index.ts`
- 核心注册器：`src/skills/bundledSkills.ts`
- 调试日志系统：`src/utils/debug.ts`（`enableDebugLogging`、`getDebugLogPath`、`logForDebugging`）
- 设置路径工具：`src/utils/settings/settings.ts`（`getSettingsFilePathForSource`）
- 错误处理工具：`src/utils/errors.ts`（`errorMessage`、`isENOENT`）
- 文件大小格式化：`src/utils/format.ts`（`formatFileSize`）
- 子 agent 类型：`src/tools/AgentTool/built-in/claudeCodeGuideAgent.ts`（`CLAUDE_CODE_GUIDE_AGENT_TYPE`）

## 依赖与外部交互

| 依赖 | 作用 |
|------|------|
| `enableDebugLogging` | 开启本会话的调试日志记录 |
| `getDebugLogPath` | 获取当前会话的日志文件绝对路径 |
| `getSettingsFilePathForSource` | 获取 settings.json 的各级路径 |
| `formatFileSize` | 将字节数格式化为人类可读字符串 |
| `isENOENT` | 判断文件不存在错误，给出友好提示 |

- **无网络调用**：纯本地文件读取与 prompt 组装。
- **权限敏感**：读取的 debug 日志可能包含系统路径、工具调用内容等，但仅用于当前会话的自我诊断，不会外发。

## 风险、边界与改进建议

1. **边界：日志尚未生成**：若用户首次调用 `/debug` 且日志文件尚不存在，`logInfo` 会提示 "No debug log exists yet — logging was just enabled."
2. **边界：非 ant 用户的信息缺口**：非 ant 用户调用 `/debug` 前的问题不会被记录，只能复现后再分析，可能丢失关键上下文。
3. **风险：UTF-8 截断问题**：`startOffset` 可能落在多字节 UTF-8 字符中间，导致尾部第一行出现乱码。当前实现未做字符边界对齐。
4. **改进建议**：
   - 在读取尾部前增加 UTF-8 字符边界对齐逻辑（向前搜索到最近的 `\n` 起始点）。
   - 对 ant 用户可考虑提供一键打包最近 N 个会话日志的功能，便于提交 bug 报告。
   - 增加对 `--debug-to-stderr` 模式的兼容提示：若用户以该模式启动，debug 内容不在文件而在 stderr，当前技能会显示 "No debug log exists"。
