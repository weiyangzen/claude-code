# terminalPreference.ts 研究文档

> 文件路径：`src/utils/deepLink/terminalPreference.ts`  
> 研究时间：2026-04-01  
> 执行器：kimi (k2p5)

---

## 场景与职责

`terminalPreference.ts` 是 deep link 终端偏好采集的**独立轻量模块**。它的唯一职责是：在 Claude Code 的**交互式启动路径**中，读取当前终端通过 `TERM_PROGRAM` 环境变量自报的身份，将其映射为 `terminalLauncher.ts` 内部使用的 `app` 名称，并持久化到全局配置中。

为什么要单独拆成一个文件？因为 `terminalLauncher.ts` 本身包含大量平台特定逻辑、AppleScript 模板、shell 转义函数以及 `child_process` 导入。如果 `interactiveHelpers.tsx`（热启动路径）直接导入 `terminalLauncher.ts`，会将整个 launcher 模块拉入启动依赖树，**破坏 LODESTONE 功能的 tree-shaking 优化**。因此，偏好采集逻辑被剥离到本模块，保持启动路径最小化。

---

## 功能点目的

| 功能 | 目的 |
|------|------|
| `updateDeepLinkTerminalPreference()` | 读取 `process.env.TERM_PROGRAM`，映射并存储到 `globalConfig.deepLinkTerminal`，供 headless deep link 启动时使用。 |

### 映射表 (`TERM_PROGRAM_TO_APP`)

| `TERM_PROGRAM`（小写） | 存储的 `app` 值 | 对应终端 |
|------------------------|-----------------|----------|
| `iterm` / `iterm.app` | `iTerm` | iTerm2 |
| `ghostty` | `Ghostty` | Ghostty |
| `kitty` | `kitty` | Kitty |
| `alacritty` | `Alacritty` | Alacritty |
| `wezterm` | `WezTerm` | WezTerm |
| `apple_terminal` | `Terminal` | Terminal.app |

- 注释说明：`TERM_PROGRAM` 的值有时与 `.app` bundle 名称不一致（例如 `iTerm.app` → `iTerm`，`Apple_Terminal` → `Terminal`），因此需要显式映射表做归一化。

---

## 具体技术实现

### 关键流程

#### `updateDeepLinkTerminalPreference(): void`
1. **平台过滤**：
   ```ts
   if (process.platform !== 'darwin') return
   ```
   只有 macOS 需要存储该偏好，因为 `detectMacosTerminal` 是唯一读取 `deepLinkTerminal` 配置的分支。

2. **读取环境变量**：
   ```ts
   const termProgram = process.env.TERM_PROGRAM
   if (!termProgram) return
   ```

3. **映射查找**：
   ```ts
   const app = TERM_PROGRAM_TO_APP[termProgram.toLowerCase()]
   if (!app) return
   ```
   未在映射表中的终端（如 Hyper、Tabby 等）直接忽略，不写入配置。

4. **幂等写入**：
   ```ts
   const config = getGlobalConfig()
   if (config.deepLinkTerminal === app) return
   
   saveGlobalConfig(current => ({ ...current, deepLinkTerminal: app }))
   logForDebugging(`Stored deep link terminal preference: ${app}`)
   ```
   若配置已相同则跳过，避免无意义的磁盘写入。

---

## 关键代码路径与文件引用

### 本文件导出
- `updateDeepLinkTerminalPreference()` → `src/interactiveHelpers.tsx` ~L177（`showSetupScreens` 函数内）

### 调用方详情

**`src/interactiveHelpers.tsx`**
```ts
if (feature('LODESTONE')) {
  updateDeepLinkTerminalPreference()
}
```
- 在 `showSetupScreens()` 中，于**信任对话框（TrustDialog）通过之后**执行。
- 与 `updateGithubRepoPathMapping()` 并列，均为 fire-and-forget 的后台任务。
- 选择此时机的原因：必须在用户已接受信任后才允许写入全局配置（防止不可信目录通过任何侧信道污染配置）。

### 被调用方详情

| 导入来源 | 符号 | 用途 |
|----------|------|------|
| `../config.js` | `getGlobalConfig`, `saveGlobalConfig` | 读取/写入全局配置 |
| `../debug.js` | `logForDebugging` | 调试日志 |

### 消费方详情

**`src/utils/deepLink/terminalLauncher.ts`**
- `detectMacosTerminal()` 函数首先检查：
  ```ts
  const stored = getGlobalConfig().deepLinkTerminal
  if (stored) {
    const match = MACOS_TERMINALS.find(t => t.app === stored)
    if (match) {
      return { name: match.name, command: match.app }
    }
  }
  ```
- 这是 macOS 终端检测链的**最高优先级信号**，因为在 headless LaunchServices / xdg-open 环境中，`TERM_PROGRAM` 通常不存在，而存储的配置是唯一能跨越进程保留的偏好信息。

---

## 依赖与外部交互

- **零外部 I/O（除配置读写）**：不调用子进程、不访问网络、不读取文件系统。
- **配置系统依赖**：依赖 `config.js` 提供的全局配置接口；`saveGlobalConfig` 内部会持久化到磁盘（通常是 `~/.claude.json` 或等价文件）。

---

## 风险、边界与改进建议

### 风险
1. **映射表遗漏常见终端**：当前仅覆盖 6 个终端。若用户使用未列出的终端（如 Warp、Tabby、Hyper、Rio 等），其偏好不会被记录，导致 deep link 启动时回退到 `mdfind` / `ls` 检测，可能选中错误终端。
2. **macOS 专属限制**：Linux 与 Windows 的 deep link 终端检测不读取该配置字段。虽然当前设计如此，但如果未来 Linux/Windows 也需要存储偏好，需要扩展平台判断或拆分映射表。
3. **`TERM_PROGRAM` 的可靠性**：某些终端模拟器可能不设置 `TERM_PROGRAM`，或设置非标准值；此外在 tmux/screen 等多路复用器内部运行时，`TERM_PROGRAM` 可能反映的是多路复用器而非底层终端，导致偏好记录失真。

### 边界
- **仅在交互式启动路径执行**：非交互模式（`-p` / `--print`）不会调用 `showSetupScreens`，因此不会更新终端偏好。对于主要使用 headless 模式的用户，其终端偏好可能长期保持旧值或为空。
- **幂等但非事务性**：`getGlobalConfig` 与 `saveGlobalConfig` 之间没有锁，极端并发场景下（如快速连续启动多个 Claude 实例）可能出现覆盖写竞态，但由于写入的是同一值，实际影响极小。

### 改进建议
1. **扩展终端映射表**：补充 Warp (`warp`)、Tabby (`tabby`)、Rio (`rio`) 等新兴终端的映射，减少遗漏。
2. **多路复用器感知**：可检测 `TERM_PROGRAM` 是否为 `tmux` / `screen`，并尝试通过其他环境变量（如 `TMUX_TERM`）推断真实终端，或干脆跳过写入以避免记录错误偏好。
3. **Linux/Windows 偏好存储**：评估是否需要在其他平台引入类似的存储机制。例如 Linux 用户可能频繁切换桌面环境，固定 `$TERMINAL` 或 `x-terminal-emulator` 的偏好意义不大；但 Windows 上存储 `wt.exe` vs `pwsh.exe` 的偏好可能有价值。
4. **测试覆盖**：当前无单元测试，建议补充：
   - 各 `TERM_PROGRAM` 值到 `app` 的映射测试；
   - `saveGlobalConfig` 被调用/跳过的幂等性测试（mock `getGlobalConfig`）；
   - 非 `darwin` 平台直接返回的边界测试。
