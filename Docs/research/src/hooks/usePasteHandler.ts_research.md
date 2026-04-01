# Research: src/hooks/usePasteHandler.ts

## 场景与职责

`usePasteHandler` 是一个 React Hook，专门用于在终端文本输入组件中处理**粘贴事件**。它解决的核心问题是：在基于 Ink 的 TUI（终端用户界面）中，终端模拟器通过 stdin 发送的粘贴内容与普通按键输入是混在一起的，需要可靠地区分"用户正在粘贴"和"用户正在逐键输入"，并在此基础上支持：

1. **普通文本粘贴**：将多字符内容作为一次性输入注入。
2. **图片文件路径粘贴**：当用户从 Finder/文件管理器拖拽或粘贴图片路径时，自动读取图片文件并转换为 base64，供对话使用。
3. **剪贴板图片粘贴**（macOS 为主）：当用户直接粘贴图片（如截图）时，从系统剪贴板读取图片数据。

该 Hook 被 `BaseTextInput.tsx`（以及 `VimTextInput.tsx`）调用，是 PromptInput 输入链路的关键一环。

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| **Bracketed Paste 检测** | 现代终端支持 bracketed paste mode（转义序列 `\x1b[200~` ... `\x1b[201~`）。Hook 通过 `event.keypress.isPasted` 标志判断输入是否来自粘贴，避免与真实按键混淆。 |
| **大文本阈值检测** | 当单次输入字符数超过 `PASTE_THRESHOLD`（800）时，即使终端未发送 bracketed paste，也视为粘贴行为。 |
| **图片路径识别** | 对输入文本进行拆分（按空格+绝对路径特征或换行符），用正则 `IMAGE_EXTENSION_REGEX` 识别图片路径，调用 `tryReadImageFromPath` 读取。 |
| **macOS 剪贴板图片读取** | 当粘贴内容为空（用户用 Cmd+V 粘贴图片）或路径读取失败且匹配临时截图路径时，回退到 `getImageFromClipboard()` 读取 PNG 数据。 |
| **粘贴状态反馈** | 暴露 `isPasting` 状态，供 UI 显示 "Pasting text…" 等反馈；同时通过 `pastePendingRef` 解决 React 批量更新导致的竞态问题（粘贴+回车同时到达时防止提前提交）。 |

## 具体技术实现

### 关键流程

#### 1. 粘贴检测入口：`wrappedOnInput`
```ts
const isFromPaste = event.keypress.isPasted
```
- 优先依赖 Ink/keypress 解析器设置的 `isPasted` 标志。
- 若 `input.length > PASTE_THRESHOLD` 或包含图片路径，也视为粘贴。
- 空粘贴（`isFromPaste && input.length === 0`）在 macOS 下触发剪贴板图片检查。

#### 2. 粘贴聚合与去尾
粘贴内容可能分多个 stdin chunk 到达。Hook 使用 `pasteState.chunks` 数组累积输入，并通过 `resetPasteTimeout` 设置 100ms 超时。超时后：
- 拼接所有 chunks。
- 过滤掉终端焦点事件残留的 `[I`、`[O` 尾部。
- 拆分路径并识别图片文件。

#### 3. 图片路径处理
```ts
const lines = pastedText
  .split(/ (?=\/|[A-Za-z]:\\)/)
  .flatMap(part => part.split('\n'))
  .filter(line => line.trim())
```
- 支持 Unix 绝对路径（`/` 开头）和 Windows 绝对路径（`C:\` 开头）。
- 对 macOS 临时截图路径（`/TemporaryItems/…screencaptureui…/Screenshot`）做特殊处理：若文件已不存在，尝试从剪贴板回退读取。

#### 4. 剪贴板图片回退
```ts
void getImageFromClipboard()
  .then(imageData => onImagePaste(imageData.base64, ...))
```
- `getImageFromClipboard` 在 macOS 上优先使用原生 NSPasteboard 模块（`image-processor-napi`），失败则回退到 `osascript`。
- 读取后经过 `maybeResizeAndDownsampleImageBuffer` 压缩，确保不超过 API 5MB 限制。

### 数据结构

```ts
type PasteHandlerProps = {
  onPaste?: (text: string) => void
  onInput: (input: string, key: Key) => void
  onImagePaste?: (
    base64Image: string,
    mediaType?: string,
    filename?: string,
    dimensions?: ImageDimensions,
    sourcePath?: string,
  ) => void
}
```

返回：
```ts
{
  wrappedOnInput: (input: string, key: Key, event: InputEvent) => void
  pasteState: { chunks: string[]; timeoutId: ReturnType<typeof setTimeout> | null }
  isPasting: boolean
}
```

### 关键常量
- `CLIPBOARD_CHECK_DEBOUNCE_MS = 50`：剪贴板检查防抖。
- `PASTE_COMPLETION_TIMEOUT_MS = 100`：粘贴完成判定超时。
- `PASTE_THRESHOLD = 800`：大文本粘贴阈值（定义在 `imagePaste.ts`）。

## 关键代码路径与文件引用

| 文件 | 作用 |
|------|------|
| `src/hooks/usePasteHandler.ts` | 本 Hook：粘贴检测、聚合、图片路径/剪贴板处理。 |
| `src/components/BaseTextInput.tsx` | 调用方：将 `wrappedOnInput` 传给 `useInput`，并消费 `isPasting` 状态。 |
| `src/utils/imagePaste.ts` | 图片路径解析、剪贴板读取、平台命令封装（osascript/xclip/wl-paste/powershell）。 |
| `src/utils/imageResizer.ts` | 图片压缩/缩放（`maybeResizeAndDownsampleImageBuffer`、格式检测）。 |
| `src/utils/platform.ts` | 平台检测（macOS/Linux/WSL/Windows），用于决定剪贴板读取策略。 |
| `src/ink.js` | 提供 `InputEvent`、`Key`、`useInput` 等底层 TUI 输入抽象。 |

## 依赖与外部交互

### 运行时依赖
- **React**：`useState`、`useCallback`、`useEffect`、`useRef`、`useMemo`。
- **usehooks-ts**：`useDebounceCallback` 用于剪贴板检查防抖。
- **Node.js 子进程**：`imagePaste.ts` 内部通过 `execa` 调用 osascript/xclip 等。
- **可选原生模块**：`image-processor-napi`（仅在 macOS + GrowthBook gate 开启时动态 import）。

### 与调用方的契约
- `onPaste`：普通文本粘贴的最终回调。
- `onImagePaste`：图片粘贴的最终回调，携带 base64、mediaType、filename、dimensions、sourcePath。
- `onInput`：非粘贴场景的原样透传。

## 风险、边界与改进建议

### 风险与边界
1. **竞态条件（已修复）**：注释中明确提到，早期版本使用独立的 stdin `on('data')` 监听器，与 `App.tsx` 的 `readable` 监听器竞争，导致丢字符。当前版本完全依赖 `event.keypress.isPasted`，消除了该竞态。
2. **React 批量更新竞态**：`pastePendingRef` 同步 ref 的存在是为了解决"同一批次中第二次 `wrappedOnInput` 调用读到过期的 `pasteState.timeoutId`"的问题——如果第二次是回车键，会导致旧输入被提前提交。
3. **macOS 剪贴板依赖外部脚本**：当原生 NSPasteboard 模块不可用时，回退到 `osascript`，冷启动可能耗时 ~1.5s。
4. **空图片文件**：`tryReadImageFromPath` 对空文件做了显式检查并返回 null，避免 API 报错。
5. **Windows/WSL 剪贴板图片支持有限**：`hasImageInClipboard` 和 `getImageFromClipboard` 的快速路径仅在 `process.platform === 'darwin'` 时启用；其他平台依赖 shell 命令，体验可能不一致。

### 改进建议
1. **统一跨平台剪贴板能力**：目前 Linux/Windows 的剪贴板图片读取依赖外部命令链，可考虑引入轻量级跨平台剪贴板库（如 `clipboardy` 的扩展）或补充更多平台的原生绑定。
2. **粘贴超时动态调整**：100ms 的 `PASTE_COMPLETION_TIMEOUT_MS` 在远程/高延迟终端场景下可能偏紧，可根据输入 chunk 到达间隔动态延长。
3. **图片路径批量读取的并发控制**：`Promise.all(imagePaths.map(...))` 在拖拽大量图片时可能同时发起过多文件 IO，建议增加并发限制（如 `p-map`）。
4. **进一步减少 osascript 依赖**：`image-processor-napi` 已覆盖 macOS 主路径，可考虑将其作为必选依赖以彻底消除 osascript 延迟和潜在的安全/权限弹窗问题。
