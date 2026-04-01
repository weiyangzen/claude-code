# src/utils/jetbrains.ts 研究文档

## 场景与职责

`jetbrains.ts` 负责检测 JetBrains 系列 IDE（IntelliJ IDEA、PyCharm、WebStorm 等）中是否安装了 Claude Code 插件（`claude-code-jetbrains-plugin`）。这是 Claude Code IDE 集成功能的一部分——当用户在 JetBrains IDE 的终端中运行 `claude` 时，CLI 需要知道 IDE 插件是否可用，以决定是否启用 IDE 相关功能（如文件打开、diff 展示、代码补全等）。

该模块与 `src/utils/ide.ts` 紧密协作，`ide.ts` 中的 `isIDEExtensionInstalled` 函数在检测到 JetBrains IDE 时会调用本模块。

## 功能点目的

### 1. `isJetBrainsPluginInstalled`
核心检测函数，接受 `ideType: IdeType`，返回 `Promise<boolean>`。检测逻辑：
1. 根据 IDE 名称（如 `pycharm`、`intellij`）和当前操作系统，构建可能的插件目录候选路径。
2. 遍历这些候选路径，查找名为 `claude-code-jetbrains-plugin` 的子目录。
3. 若任一目录存在，则判定插件已安装。

### 2. `isJetBrainsPluginInstalledCached` / `isJetBrainsPluginInstalledCachedSync`
提供带缓存的异步和同步版本：
- **异步缓存**：使用 `Promise` 缓存避免并发重复检测，同时用 `Map<IdeType, boolean>` 缓存最终结果。
- **同步缓存**：返回已解析的缓存值，若尚未解析则返回 `false`。用于必须在同步上下文（如 React `isActive` 检查）中快速判断的场景。
- `forceRefresh` 参数允许清除缓存并重新检测。

## 具体技术实现

### 插件目录构建
`buildCommonPluginDirectoryPaths` 根据 `os.platform()` 返回不同路径：

**macOS (`darwin`)**：
- `~/Library/Application Support/JetBrains`
- `~/Library/Application Support`
- Android Studio 额外检查 `~/Library/Application Support/Google`

**Windows (`win32`)**：
- `%APPDATA%\JetBrains`
- `%LOCALAPPDATA%\JetBrains`
- `%APPDATA%`
- Android Studio 额外检查 `%LOCALAPPDATA%\Google`

**Linux (`linux`)**：
- `~/.config/JetBrains`
- `~/.local/share/JetBrains`
- `~/.{IDE名称}`（如 `~/.PyCharm`、`~/.IdeaIC`）
- Android Studio 额外检查 `~/.config/Google`

### IDE 名称到目录前缀映射
```typescript
const ideNameToDirMap: { [key: string]: string[] } = {
  pycharm: ['PyCharm'],
  intellij: ['IntelliJIdea', 'IdeaIC'],
  webstorm: ['WebStorm'],
  // ... 其他 IDE
}
```

### 目录探测逻辑
`detectPluginDirectories` 执行以下步骤：
1. 获取候选基础目录列表。
2. 将 `idePatterns` 预编译为正则表达式（`^PyCharm` 等）。
3. 对每个基础目录调用 `fs.readdir`。
4. 匹配目录名前缀，同时接受 `isDirectory()` 和 `isSymbolicLink()`（注释说明：GNU stow 用户可能使用符号链接管理配置目录）。
5. **Linux 特殊处理**：Linux 的 JetBrains 配置目录结构不同，匹配到的目录直接视为插件目录，不再拼接 `plugins` 子目录。
6. 非 Linux 系统：拼接 `/plugins` 后通过 `fs.stat` 确认存在，才加入结果。
7. 最后去重（`filter((dir, index) => foundDirectories.indexOf(dir) === index)`）。

### 缓存策略
```typescript
const pluginInstalledCache = new Map<IdeType, boolean>()
const pluginInstalledPromiseCache = new Map<IdeType, Promise<boolean>>()
```

缓存逻辑：
- 若 `forceRefresh = false` 且存在进行中的 Promise，直接返回该 Promise。
- 否则创建新的检测 Promise，完成后写入 `pluginInstalledCache`。
- `isJetBrainsPluginInstalledCachedSync` 只读 `pluginInstalledCache`，不触发异步检测。

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/utils/jetbrains.ts:29-82` | `buildCommonPluginDirectoryPaths` 路径构建 |
| `src/utils/jetbrains.ts:85-132` | `detectPluginDirectories` 目录探测 |
| `src/utils/jetbrains.ts:134-148` | `isJetBrainsPluginInstalled` 核心检测 |
| `src/utils/jetbrains.ts:150-180` | `isJetBrainsPluginInstalledMemoized` / `isJetBrainsPluginInstalledCached` 异步缓存 |
| `src/utils/jetbrains.ts:187-191` | `isJetBrainsPluginInstalledCachedSync` 同步缓存 |
| `src/utils/ide.ts` | 调用方，`isIDEExtensionInstalled` |
| `src/utils/fsOperations.ts` | `getFsImplementation` 文件系统抽象 |

## 依赖与外部交互

### 内部依赖
- `./ide.js`：`IdeType` 类型
- `./fsOperations.js`：`getFsImplementation`
- `os`：`homedir`, `platform`
- `path`：`join`

### 外部依赖
- 无第三方依赖。

### 调用方
- `src/utils/ide.ts`：`isIDEExtensionInstalled` 在 `isJetBrainsIde(ideType)` 为真时调用
- `src/utils/statusNoticeDefinitions.tsx`：状态通知定义

## 风险、边界与改进建议

### 风险与边界
1. **路径硬编码的维护成本**：JetBrains 的配置目录路径在不同版本和安装方式下可能变化（如 Toolbox App 安装 vs 独立安装）。当前路径列表基于官方文档和常见实践，但可能无法覆盖所有边缘情况。
2. **Linux 目录结构假设**：Linux 上直接认为匹配到的目录就是插件目录，不再检查 `plugins` 子目录。这一假设虽然基于 JetBrains 在 Linux 上的实际行为，但若未来 JetBrains 改变目录结构，检测会失效。
3. **符号链接的 TOCTOU**：`detectPluginDirectories` 中通过 `entry.isSymbolicLink()` 接受符号链接，但后续的 `fs.stat(pluginPath)` 才确认它是否指向真实目录。在这两个操作之间，符号链接目标可能被修改（竞态条件）。
4. **缓存过期问题**：`pluginInstalledCache` 在进程生命周期内不会自动过期。如果用户在 Claude 运行期间安装/卸载 JetBrains 插件，缓存不会反映这一变化，除非调用 `forceRefresh`。
5. **Android Studio 的 Google 路径**：Android Studio 的配置目录可能位于 `Google` 而非 `JetBrains` 下，代码已特殊处理，但只覆盖了 macOS/Windows/Linux 的常见路径，可能遗漏某些发行版（如 JetBrains Toolbox 自定义路径）。

### 改进建议
1. **增加 Toolbox 路径支持**：JetBrains Toolbox 允许用户自定义 IDE 配置目录位置（`~/.config/JetBrains/Toolbox` 中有 `settings.json`）。可以解析 Toolbox 配置来获取更准确的 IDE 安装路径和配置路径。
2. **定期缓存刷新**：在 `isJetBrainsPluginInstalledCached` 中加入 TTL（如 5 分钟），避免缓存长期陈旧。
3. **统一 IDE 检测抽象**：JetBrains 插件检测和 VS Code 扩展检测（`ide.ts` 中）的实现风格差异较大。可以考虑抽象出一个统一的 `IDEExtensionDetector` 接口，让两种 IDE 的检测逻辑更一致。
4. **增强错误日志**：当前 `detectPluginDirectories` 在 `fs.readdir` 失败时完全静默忽略。对于非 `ENOENT` 错误（如 `EACCES` 权限拒绝），可以记录 debug 日志，帮助排查检测失败原因。
5. **支持 Fleet 的检测**：`ideNameToDirMap` 中 `fleet: ['Fleet']`，但 `supportedIdeConfigs` 中 Fleet 的 `processKeywordsMac` 和 `processKeywordsLinux` 为空（不自动检测）。这导致 Fleet 的插件检测虽然存在路径逻辑，但实际上很少被触发。
