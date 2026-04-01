# registerProtocol.ts 研究文档

> 文件路径：`src/utils/deepLink/registerProtocol.ts`  
> 研究时间：2026-04-01  
> 执行器：kimi (k2p5)

---

## 场景与职责

`registerProtocol.ts` 负责将 `claude-cli://` 自定义 URI Scheme **注册到操作系统**，使得浏览器或其他应用点击该协议链接时能够唤起 Claude Code。它覆盖三大桌面平台：

- **macOS**：创建最小化的 `.app` trampoline bundle，利用 `CFBundleURLTypes` 声明 URL scheme；
- **Linux**：生成 `.desktop` 文件并通过 `xdg-mime` 注册；
- **Windows**：向当前用户注册表写入 `HKEY_CURRENT_USER\Software\Classes\claude-cli`。

此外，该模块还提供：
- **`isProtocolHandlerCurrent`**：读取注册产物（symlink / .desktop / registry）判断当前注册是否指向正确的 claude 二进制路径；
- **`ensureDeepLinkProtocolRegistered`**：每会话后台自动检查并修复注册（支持安装路径变更后的自愈）。

---

## 功能点目的

| 功能 | 目的 |
|------|------|
| `registerProtocolHandler(claudePath?)` | 显式注册协议处理器，按平台分发到对应实现。 |
| `registerMacos(claudePath)` | 在 `~/Applications/Claude Code URL Handler.app` 创建 bundle，内含指向 claude 二进制的符号链接。 |
| `registerLinux(claudePath)` | 写入 `~/.local/share/applications/claude-code-url-handler.desktop` 并调用 `xdg-mime`。 |
| `registerWindows(claudePath)` | 执行 `reg add` 写入三条注册表项。 |
| `resolveClaudePath()` | 解析用于注册的二进制路径：优先 `~/.local/bin/claude` 稳定符号链接，否则回退 `process.execPath`。 |
| `isProtocolHandlerCurrent(claudePath)` | 通过读取实际 OS 注册产物判断注册是否最新，避免依赖可能跨机同步的配置缓存。 |
| `ensureDeepLinkProtocolRegistered()` | 后台自动注册（fire-and-forget），带 24 小时失败退避。 |

---

## 具体技术实现

### 关键流程

#### 1. macOS 注册 (`registerMacos`)
```
rm -rf ~/Applications/Claude Code URL Handler.app
mkdir -p .../Contents/MacOS
write Info.plist (含 CFBundleURLTypes 声明 claude-cli)
symlink claudePath → .../Contents/MacOS/claude
lsregister -R <appDir>
```
- **安全设计**：bundle 的 `CFBundleExecutable` 是一个**符号链接**，指向已安装且已签名的 `claude` 二进制，而非引入新的可执行文件。这样可避免被 Santa 等终端安全工具拦截。
- **原子性**：符号链接是最后一个写入操作，充当“提交标记”。若 `Info.plist` 写入失败，下次 `isProtocolHandlerCurrent` 会因读不到 symlink 而返回 `false`，从而触发重新注册。

#### 2. Linux 注册 (`registerLinux`)
- 生成 `.desktop` 文件：
  ```ini
  [Desktop Entry]
  Name=Claude Code URL Handler
  Comment=Handle claude-cli:// deep links for Claude Code
  Exec="<claudePath>" --handle-uri %u
  Type=Application
  NoDisplay=true
  MimeType=x-scheme-handler/claude-cli;
  ```
- 若系统存在 `xdg-mime`，则执行：
  ```
  xdg-mime default claude-code-url-handler.desktop x-scheme-handler/claude-cli
  ```
- **headless 容错**：在 WSL/Docker/CI 等无桌面环境中，`xdg-mime` 可能不存在，此时仅保留 `.desktop` 文件即可（部分应用直接读取该文件）。

#### 3. Windows 注册 (`registerWindows`)
执行三条 `reg add` 命令：
1. `HKEY_CURRENT_USER\Software\Classes\claude-cli` 默认值设为 `URL:Claude Code URL Handler`
2. 同键下新建 `URL Protocol` 值（空字符串）
3. `...\shell\open\command` 默认值设为 `"<claudePath>" --handle-uri "%1"`

#### 4. 自动注册 (`ensureDeepLinkProtocolRegistered`)
```
if disableDeepLinkRegistration === 'disable' → 跳过
if feature('tengu_lodestone_enabled') 为 false → 跳过
resolveClaudePath()
if isProtocolHandlerCurrent(claudePath) → 跳过
if 失败标记文件存在且 < 24h → 跳过（退避）
registerProtocolHandler(claudePath)
logEvent('tengu_deep_link_registered', { success: true })
```
- 失败标记路径：`~/.claude/.deep-link-register-failed`
- 仅对 `EACCES` / `ENOSPC` 写入失败标记（确定性失败）；其他错误（如 `xdg-mime` 非零退出）不标记，以便下次重试。

### 数据结构 / 常量

| 常量 | 值 | 说明 |
|------|-----|------|
| `MACOS_BUNDLE_ID` | `com.anthropic.claude-code-url-handler` | 用于 LaunchServices 识别与 `__CFBundleIdentifier` 校验。 |
| `APP_NAME` | `Claude Code URL Handler` | 注册表 / bundle 显示名称。 |
| `DESKTOP_FILE_NAME` | `claude-code-url-handler.desktop` | Linux 桌面入口文件名。 |
| `MACOS_APP_NAME` | `Claude Code URL Handler.app` | macOS bundle 目录名。 |
| `FAILURE_BACKOFF_MS` | `24 * 60 * 60 * 1000` | 失败退避时长（24 小时）。 |

---

## 关键代码路径与文件引用

### 本文件导出
- `MACOS_BUNDLE_ID` → `src/utils/deepLink/protocolHandler.ts` ~L23（`handleUrlSchemeLaunch` 检测）
- `registerProtocolHandler()` → 未在本批次发现显式 CLI 命令调用，推测供未来手动注册命令或测试使用。
- `isProtocolHandlerCurrent()` → 本文件内部 `ensureDeepLinkProtocolRegistered` ~L307
- `ensureDeepLinkProtocolRegistered()` → `src/utils/backgroundHousekeeping.ts` ~L40（后台 housekeeping 调用）

### 调用方详情

**`src/utils/backgroundHousekeeping.ts`**
```ts
if (feature('LODESTONE') && getIsInteractive()) {
  void registerProtocolModule!.ensureDeepLinkProtocolRegistered()
}
```
- 在 `startBackgroundHousekeeping()` 中 fire-and-forget 执行，确保每会话检查一次注册状态。

### 被调用方详情

| 导入来源 | 符号 | 用途 |
|----------|------|------|
| `fs/promises` | `promises as fs` | 文件/目录/符号链接操作 |
| `os` | `*` | 获取 home 目录 |
| `path` | `*` | 路径拼接 |
| `src/services/analytics/growthbook.js` | `getFeatureValue_CACHED_MAY_BE_STALE` | 功能开关 `tengu_lodestone_enabled` |
| `src/services/analytics/index.js` | `logEvent` | 注册成功/失败埋点 |
| `../debug.js` | `logForDebugging` | 调试日志 |
| `../envUtils.js` | `getClaudeConfigHomeDir` | 失败标记目录 |
| `../errors.js` | `getErrnoCode` | 提取错误码用于退避判断 |
| `../execFileNoThrow.js` | `execFileNoThrow` | 执行 `lsregister`、`xdg-mime`、`reg` |
| `../settings/settings.js` | `getInitialSettings` | 读取 `disableDeepLinkRegistration` |
| `../which.js` | `which` | 查找 `xdg-mime` |
| `../xdg.js` | `getUserBinDir`, `getXDGDataHome` | 解析 Linux 桌面路径与稳定二进制路径 |
| `./parseDeepLink.js` | `DEEP_LINK_PROTOCOL` | 协议名常量复用 |

---

## 依赖与外部交互

### 外部系统命令
| 平台 | 命令 | 作用 |
|------|------|------|
| macOS | `lsregister -R <app>` | 向 LaunchServices 注册/刷新 URL scheme。 |
| Linux | `xdg-mime default <desktop> <mimetype>` | 将 `.desktop` 设为默认 handler。 |
| Windows | `reg add/query` | 读写注册表。 |

### 文件系统产物
- **macOS**：`~/Applications/Claude Code URL Handler.app/Contents/MacOS/claude`（symlink）
- **Linux**：`~/.local/share/applications/claude-code-url-handler.desktop`
- **Windows**：`HKEY_CURRENT_USER\Software\Classes\claude-cli\shell\open\command`

---

## 风险、边界与改进建议

### 风险
1. **`lsregister` 路径硬编码**：`/System/Library/Frameworks/CoreServices.framework/.../lsregister` 是 macOS 私有实现细节，未来系统更新可能移动该路径，导致注册静默失败。
2. **Windows 注册表无撤销逻辑**：模块提供注册与检查，但**未提供卸载/反注册**功能。用户卸载 Claude Code 后，注册表残留可能导致点击 `claude-cli://` 链接时系统尝试调用已不存在的路径。
3. **Linux `xdg-mime` 非零退出即抛错**：某些 Linux 发行版中 `xdg-mime` 即使成功也可能返回非零，或需要特定桌面会话环境；当前实现会将其视为失败并触发 analytics 失败事件。
4. **并发注册竞态**：若用户同时启动多个 Claude 实例，`registerMacos` 的 `fs.rm` + `fs.symlink` 序列存在竞态窗口，可能导致 bundle 处于不完整状态。

### 边界
- **仅注册当前用户**：Windows 使用 `HKEY_CURRENT_USER`，Linux 使用 `~/.local/share`，macOS 使用 `~/Applications`，均不涉及系统级权限。
- **`isProtocolHandlerCurrent` 的读取与写入强耦合**：注释明确警告 "Keep the writer and reader in lockstep — drift here means the check returns a perpetual false"。
- **失败标记存储在 `~/.claude`**（`getClaudeConfigHomeDir`），该目录按设计是 per-machine 不同步的，适合存放此类本地状态。

### 改进建议
1. **macOS `lsregister` 路径弹性化**：可先尝试 `which lsregister` 或检查文件存在性，再回退到已知路径。
2. **增加反注册 API**：为未来的卸载流程提供 `unregisterProtocolHandler()`，清理 `.app`、`.desktop` 和注册表项。
3. **Linux 注册容错增强**：对 `xdg-mime` 失败增加更细粒度的错误分类（如区分命令不存在 vs 执行失败）。
4. **并发安全**：对 macOS bundle 目录使用临时目录写入 + 原子重命名，避免竞态。
5. **测试覆盖**：当前无单元测试，建议补充：
   - 各平台 `isProtocolHandlerCurrent` 的 mock 测试（mock `fs.readlink`、`fs.readFile`、`execFileNoThrow`）；
   - `ensureDeepLinkProtocolRegistered` 的功能开关、退避逻辑、成功与失败路径测试。
