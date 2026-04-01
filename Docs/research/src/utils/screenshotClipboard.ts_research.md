# 研究文档：src/utils/screenshotClipboard.ts

## 场景与职责

`screenshotClipboard.ts` 负责将 **ANSI 转义序列格式的终端文本** 转换为 PNG 图片并写入系统剪贴板。它是 Claude Code `/stats` 或类似截图功能的底层实现，使用户能够将终端输出以图片形式分享到 Slack、GitHub Issues 等不支持 ANSI 渲染的平台。

模块的核心职责：
1. 接收 ANSI 文本，调用 `ansiToPng` 生成 PNG 缓冲区。
2. 将 PNG 写入临时文件。
3. 根据操作系统（macOS / Linux / Windows）调用对应的系统命令将图片写入剪贴板。
4. 清理临时文件，返回操作结果。

---

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| `copyAnsiToClipboard(ansiText, options?)` | 公共入口：ANSI → PNG → 临时文件 → 剪贴板 → 清理，全程异常捕获并返回结构化结果。 |
| `copyPngToClipboard(pngPath)` | 平台适配层：分别使用 AppleScript（macOS）、xclip/xsel（Linux）、PowerShell（Windows）将 PNG 写入剪贴板。 |

---

## 具体技术实现

### 1. `copyAnsiToClipboard`

流程：
1. `mkdir(tempDir, { recursive: true })` 创建 `~/.tmp/claude-code-screenshots/`（或系统 tmpdir 下的子目录）。
2. 生成带时间戳的临时 PNG 路径：`screenshot-${Date.now().png`。
3. 调用 `ansiToPng(ansiText, options)` 生成 `Buffer`。
4. `writeFile(pngPath, pngBuffer)`。
5. 调用 `copyPngToClipboard(pngPath)`。
6. `unlink(pngPath)` 清理临时文件（错误被静默吞掉）。
7. 返回 `{ success: boolean, message: string }`。

异常处理：任何步骤出错都会被 catch，`logError` 后返回 `success: false` 与错误消息。

### 2. `copyPngToClipboard`

#### macOS (`platform === 'macos'`)
- 使用 `osascript -e` 执行 AppleScript：
  ```applescript
  set the clipboard to (read (POSIX file "<path>") as «class PNGf»)
  ```
- 路径中的 `\` 与 `"` 做转义：`pngPath.replace(/\\/g, '\\\\').replace(/"/g, '\\"')`。
- 超时 5 秒。

#### Linux (`platform === 'linux'`)
- **首选 xclip**：
  ```bash
  xclip -selection clipboard -t image/png -i <pngPath>
  ```
- **回退 xsel**：
  ```bash
  xsel --clipboard --input --type image/png
  ```
  注意：xsel 的 `--type` 参数实际上并不被所有版本支持；代码中通过 `execFileNoThrowWithCwd` 执行，若返回 code !== 0 则视为失败。
- 若两者均失败，返回提示用户安装 xclip 的消息。

#### Windows (`platform === 'windows'`)
- 使用 PowerShell 加载 .NET 程序集并调用 Clipboard API：
  ```powershell
  Add-Type -AssemblyName System.Windows.Forms;
  [System.Windows.Forms.Clipboard]::SetImage(
    [System.Drawing.Image]::FromFile('<pngPath>')
  )
  ```
- 路径中的单引号做转义：`pngPath.replace(/'/g, "''")`。
- 超时 5 秒。

#### 其他平台
- 返回 `success: false, message: "Screenshot to clipboard is not supported on ${platform}"`。

---

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/utils/screenshotClipboard.ts:16-44` | `copyAnsiToClipboard` 主流程。 |
| `src/utils/screenshotClipboard.ts:46-121` | `copyPngToClipboard` 平台适配实现。 |
| `src/utils/ansiToPng.ts` | 被依赖：ANSI → PNG 渲染引擎。 |
| `src/utils/execFileNoThrow.ts` | 被依赖：跨平台安全执行外部命令。 |
| `src/utils/platform.ts` | 被依赖：`getPlatform()` 判断操作系统。 |
| `src/components/Stats.tsx` | 主要调用方：/stats 截图到剪贴板功能。 |

---

## 依赖与外部交互

- **Node.js 内置模块**：
  - `fs/promises`：`mkdir`, `writeFile`, `unlink`
  - `os`：`tmpdir`
  - `path`：`join`
- **内部依赖**：
  - `./ansiToPng.js`：`ansiToPng`, `AnsiToPngOptions`
  - `./execFileNoThrow.js`：`execFileNoThrowWithCwd`
  - `./log.js`：`logError`
  - `./platform.js`：`getPlatform`
- **外部系统命令**：
  - macOS：`osascript`
  - Linux：`xclip`, `xsel`
  - Windows：`powershell`
- **调用方**：
  - `src/components/Stats.tsx`

---

## 风险、边界与改进建议

### 风险与边界

1. **Linux `xsel --type` 兼容性问题**：`xsel` 的 `--type` 参数在部分发行版中并不存在（xsel 1.2.0 及更早版本不支持）。代码中通过检查 exit code 来判定失败，这是合理的，但回退消息只提示安装 xclip，没有说明 xsel 版本问题，可能让用户困惑。

2. **Windows PowerShell 依赖 .NET WinForms**：`System.Windows.Forms` 与 `System.Drawing` 在 Windows Server Core 或某些精简版 Windows 上可能缺失，导致 `Add-Type` 失败。当前没有针对这种情况的 fallback。

3. **临时目录清理不彻底**：若 `copyPngToClipboard` 抛异常或进程崩溃，`unlink` 可能未执行，导致临时 PNG 文件残留。虽然位于系统 tmpdir，长期积累仍可能占用磁盘。

4. **AppleScript 路径转义不完整**：当前只转义了反斜杠与双引号。若路径中包含 AppleScript 的其他特殊字符（如回车符），可能导致脚本语法错误。不过 POSIX 路径通常不含这些字符。

5. **无剪贴板内容校验**：写入剪贴板后没有“读回验证”步骤，无法 100% 确认图片确实已进入剪贴板。某些桌面环境（如 Wayland 下的 xclip）可能存在剪贴板所有权问题，写入后若程序立即退出，剪贴板内容可能丢失。

### 改进建议

1. **Linux 增加 `wl-copy` fallback**：Wayland 桌面环境下 `xclip` 经常不可用或行为异常，应优先尝试 `wl-copy --type image/png < <pngPath>`，再回退到 xclip/xsel。

2. **Windows 增加 `Set-Clipboard` fallback**：PowerShell 5+ 内置 `Set-Clipboard` cmdlet，可尝试：
   ```powershell
   Set-Clipboard -Path '<pngPath>'
   ```
   作为 WinForms 失败后的回退。

3. **使用 `fs.mkdtemp` 创建隔离临时目录**：相比在公共 `tmpdir` 下直接写文件，使用 `fs/promises.mkdtemp` 创建一次性子目录，并在成功或失败后 `rm(tempDir, { recursive: true })`，可更可靠地清理残留。

4. **增加剪贴板读回验证（可选）**：在 macOS 上可通过 `osascript -e 'clipboard info'` 验证剪贴板中是否存在 `PNGf` 类型；在 Windows 上可通过 `[Windows.Forms.Clipboard]::ContainsImage()`。这会增加一次系统调用，但可显著提升用户信任度。

5. **路径转义使用 `JSON.stringify` 或 base64 编码**：对于 AppleScript 与 PowerShell，与其手动转义，不如将路径通过环境变量或 base64 传递，彻底消除注入风险。例如：
   ```ts
   const b64 = Buffer.from(pngPath).toString('base64')
   const script = `[System.Convert]::FromBase64String('${b64}') | ...`
   ```
   虽然当前路径来自受控的临时目录生成逻辑，但 base64 传递是更 robust 的工程实践。
