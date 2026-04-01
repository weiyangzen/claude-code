# ShellDetailDialog.tsx 研究文档

## 场景与职责

`ShellDetailDialog.tsx` 是 Claude Code 终端 UI 中用于展示**本地 Shell 任务详情**的模态对话框组件。它属于 `src/components/tasks/` 目录下的任务详情视图家族，与 `RemoteSessionDetailDialog.tsx`、`AsyncAgentDetailDialog.tsx`、`InProcessTeammateDetailDialog.tsx` 等并列。

该组件的核心职责包括：
1. **展示 Shell 任务的元信息**：状态（status）、运行时长（runtime）、执行的命令/脚本（command/script）。
2. **展示任务输出**：从磁盘读取任务输出文件的最后 8KB（tail），避免加载大文件导致内存爆炸。
3. **提供交互控制**：关闭对话框、返回上一级、终止正在运行的 Shell 任务。
4. **实时刷新输出**：对于 `running` 状态的任务，通过 `setInterval` 每秒重新读取输出尾部，保证用户看到最新日志。

该组件同时支持普通 `bash` 任务和 `monitor` 任务（`kind === "monitor"`），在 UI 文案上会做区分（如 `"Monitor details"` vs `"Shell details"`、`"Script:"` vs `"Command:"`）。

---

## 功能点目的

### 1. `ShellDetailDialog`（主组件）
接收以下 props：
```ts
type Props = {
  shell: DeepImmutable<LocalShellTaskState>;
  onDone: (result?: string, options?: { display?: CommandResultDisplay }) => void;
  onKillShell?: () => void;
  onBack?: () => void;
};
```

渲染流程：
- 使用 `useTerminalSize()` 获取终端列宽，用于后续输出框的宽度计算。
- 使用 `useState(() => getTaskOutput(shell))` 创建输出 Promise，并通过 `useDeferredValue` 进行延迟更新，避免快速轮询时 UI 闪烁。
- 使用 `useEffect` 在 `shell.status === "running"` 时启动 1 秒定时器，持续刷新输出 Promise。
- 使用 `useKeybindings` 绑定 `"confirm:yes"` 到 `handleClose`（Enter 确认关闭）。
- 自定义 `handleKeyDown` 处理：
  - `Space` → 关闭对话框
  - `Left Arrow` → 返回（`onBack`）
  - `x` → 终止运行中的 Shell（`onKillShell`）

### 2. `getTaskOutput`
异步函数，负责读取任务输出文件的尾部：
- 通过 `getTaskOutputPath(shell.id)` 获取输出文件路径。
- 调用 `tailFile(path, SHELL_DETAIL_TAIL_BYTES)` 读取最后 8KB。
- 若文件不存在或读取失败，返回 `{ content: '', bytesTotal: 0 }`。

### 3. `ShellOutputContent`
内部子组件，利用 React 19 的 `use(outputPromise)` 在 Suspense 边界内解包 Promise：
- 若 `content` 为空，显示 `"No output available"`。
- 否则，从内容末尾向前提取最多 **10 行**（通过 `lastIndexOf("\n")` 手动解析），并渲染为带边框的垂直列表。
- 若 `bytesTotal > content.length`，在底部显示 `"Showing N lines of {formatFileSize(bytesTotal)}"`，提示用户输出被截断。
- 输出框高度固定为 12 行，宽度为 `columns - 6`，采用 `borderStyle="round"` 的 `Box` 包裹。

---

## 具体技术实现（关键流程/数据结构/协议/命令）

### 关键常量
```ts
const SHELL_DETAIL_TAIL_BYTES = 8192; // 8KB
```

### 输出轮询机制
```ts
useEffect(() => {
  if (shell.status !== "running") return;
  const timer = setInterval(() => {
    setOutputPromise(() => getTaskOutput(shell));
  }, 1000);
  return () => clearInterval(timer);
}, [shell.id, shell.status]);
```
- 仅在 `running` 时轮询，状态变为 `completed`/`failed`/`killed` 时自动清理定时器。
- `useDeferredValue(outputPromise)` 让 React 在后台准备新 Promise，避免阻塞当前 UI 帧。

### 行解析算法（ShellOutputContent）
```ts
const starts = [];
let pos = content.length;
for (let i = 0; i < 10 && pos > 0; i++) {
  const prev = content.lastIndexOf("\n", pos - 1);
  starts.push(prev + 1);
  pos = prev;
}
starts.reverse();
```
- 从字符串末尾向前找最多 10 个换行符，记录每行起始索引。
- 然后按顺序 `slice(start, end)` 提取每行内容。
- 该算法避免了 `split("\n")` 可能产生的大量临时字符串（当 8KB 内容很长时）。

### 状态颜色映射
```ts
shell.status === "running"
  ? <Text color="background">...</Text>
  : shell.status === "completed"
    ? <Text color="success">...</Text>
    : <Text color="error">...</Text>
```
- `background` 色在终端中通常是高亮/主题主色，用于表示运行中。

### 磁盘 I/O 路径
- `getTaskOutputPath` → `src/utils/task/diskOutput.ts`
- `tailFile` → `src/utils/fsOperations.ts`
- 输出文件实际存储在 `getProjectTempDir() / getSessionId() / tasks / {taskId}.output`

---

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/components/tasks/ShellDetailDialog.tsx` | 本文件 |
| `src/components/tasks/BackgroundTasksDialog.tsx` | 可能的调用方（打开详情弹窗） |
| `src/tasks/LocalShellTask/guards.ts` | `LocalShellTaskState` 类型定义 |
| `src/utils/task/diskOutput.ts` | `getTaskOutputPath` |
| `src/utils/fsOperations.ts` | `tailFile` |
| `src/utils/format.ts` | `formatDuration`, `formatFileSize`, `truncateToWidth` |
| `src/hooks/useTerminalSize.ts` | 获取终端尺寸 |
| `src/keybindings/useKeybinding.ts` | `useKeybindings` |
| `src/components/design-system/Dialog.tsx` | 对话框容器 |
| `src/components/design-system/Byline.tsx` | 底部快捷键提示栏 |
| `src/components/design-system/KeyboardShortcutHint.tsx` | 快捷键标签 |

---

## 依赖与外部交互

### 运行时依赖
- **Ink**：`Box`, `Text`, `Suspense`, `use`（React 19 特性）。
- **文件系统**：通过 `tailFile` 异步读取任务输出，不经过主进程 IPC，直接 Node.js `fs/promises`。
- **键盘交互**：
  - `useKeybindings` 提供声明式快捷键绑定。
  - `onKeyDown` 处理方向键和 `x` 键的自定义行为。

### 数据流
1. `LocalShellTask.tsx`（或相关执行器）将 Shell 的 stdout/stderr 写入 `diskOutput.ts` 管理的输出文件。
2. 用户通过背景任务列表（`BackgroundTaskStatus.tsx` / `BackgroundTasksDialog.tsx`）选中某个 Shell 任务，打开详情。
3. `ShellDetailDialog` 挂载时创建 `getTaskOutput(shell)` Promise。
4. `ShellOutputContent` 在 `Suspense` 边界内通过 `use(promise)` 读取内容并渲染。
5. 若任务仍在运行，每秒重新创建 Promise，触发 `Suspense` 重新获取数据。

---

## 风险、边界与改进建议

### 风险
1. **编译产物与源码不一致**：与 `RemoteSessionProgress.tsx` 类似，该文件也是 React Compiler 编译输出，包含大量 `$[n]` memo cache 代码。直接修改此文件而不重新编译源码会导致运行时与源码不匹配。
2. **轮询带来的磁盘压力**：对于高频输出（如 `tail -f` 风格的 monitor 任务），每秒读取 8KB 磁盘文件虽然不大，但在大量 Shell 任务同时运行时可能累积成可观的 I/O。
3. **`useDeferredValue` + `Suspense` 的闪烁问题**：虽然 `useDeferredValue` 旨在减少闪烁，但每秒重新创建 Promise 仍可能导致旧内容与新内容之间出现短暂的白屏或 fallback（`"Loading output…"`）。
4. **硬编码的 8KB / 10 行限制**：如果一行非常长（如 JSON 单行输出），8KB 可能只包含 1-2 行，而算法仍尝试找 10 行，实际渲染行数可能远少于预期。

### 边界
- 输出文件可能不存在（任务刚启动尚未写入），此时静默返回空字符串，UI 显示 `"No output available"`。
- `truncateToWidth(shell.command, 280)` 对命令长度做硬截断，防止超长命令撑爆对话框。
- `monitor` 任务与普通 `bash` 任务在 UI 文案上区分，但底层数据结构相同（`LocalShellTaskState`）。

### 改进建议
1. **增量读取替代 tail**：当前每秒重新读取最后 8KB，对于持续增长的日志是冗余的。可以改为维护一个 `fromOffset`，仅读取新增内容并追加到已有内容前（或维护一个环形缓冲区），将 I/O 从 O(8KB) 降到 O(增量)。
2. **提取行解析工具函数**：`ShellOutputContent` 中的 10 行提取逻辑可以抽离为 `getLastNLines(content, n)` 并补充单元测试。
3. **支持更大的 tail 或用户配置**：8KB 对于某些编译日志可能只显示几十行。可以考虑根据终端高度动态调整 `SHELL_DETAIL_TAIL_BYTES`（例如 `rows * 平均行宽`）。
4. **增加复制输出快捷键**：详情弹窗目前只能查看，无法复制。可以考虑增加 `c` 键将输出复制到剪贴板（通过系统剪贴板工具或 OSC 52）。
5. **错误状态展示优化**：当前读取失败仅返回空内容，没有向用户提示 `"输出文件读取失败"`。对于权限问题或磁盘故障，静默失败可能让用户困惑。
