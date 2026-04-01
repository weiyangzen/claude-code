# setupPortable.ts 深度研究文档

## 场景与职责

`setupPortable.ts` 是 **Claude in Chrome** 扩展安装检测的**可移植实现**。它被设计为能够在 **Claude Code TUI** 和 **VS Code 扩展** 两种宿主环境中复用，因此：

- **不依赖** `src/utils/platform.js` 的 `getPlatform()` 抽象（该抽象可能依赖 Node 运行时或特定全局状态）。
- **直接使用** `process.platform` 进行平台判断。
- **不依赖** `common.ts` 中的浏览器配置表，而是内联维护了一份精简但一致的镜像配置。

核心职责：
1. 扫描本地 Chromium 系列浏览器的数据目录，检测用户是否已安装 Claude 浏览器扩展。
2. 返回检测结果（布尔值）以及扩展所在浏览器（用于日志和诊断）。

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| `ChromiumBrowser` | 支持的浏览器类型联合类型：`chrome` \| `brave` \| `arc` \| `chromium` \| `edge` \| `vivaldi` \| `opera`。 |
| `BrowserPath` | 浏览器标识与数据路径的配对结构。 |
| `getAllBrowserDataPathsPortable()` | 返回所有支持浏览器在当前平台下的数据目录路径列表。 |
| `detectExtensionInstallationPortable()` | 遍历浏览器数据目录下的 `Default` 和 `Profile *` 目录，检查 `Extensions/<extensionId>` 是否存在。 |
| `isChromeExtensionInstalledPortable()` | `detectExtensionInstallationPortable()` 的便捷包装，仅返回 `boolean`。 |
| `isChromeExtensionInstalled()` | 最便捷的自动路径版，直接调用 `getAllBrowserDataPathsPortable()` 然后检测。 |

## 具体技术实现

### 1. 浏览器配置镜像

```typescript
const CHROMIUM_BROWSERS: Record<ChromiumBrowser, BrowserDataConfig> = {
  chrome:   { macos: ['Library', 'Application Support', 'Google', 'Chrome'],           linux: ['.config', 'google-chrome'],      windows: { path: ['Google', 'Chrome', 'User Data'] } },
  brave:    { macos: ['Library', 'Application Support', 'BraveSoftware', 'Brave-Browser'], linux: ['.config', 'BraveSoftware', 'Brave-Browser'], windows: { path: ['BraveSoftware', 'Brave-Browser', 'User Data'] } },
  arc:      { macos: ['Library', 'Application Support', 'Arc', 'User Data'],           linux: [],                                 windows: { path: ['Arc', 'User Data'] } },
  chromium: { macos: ['Library', 'Application Support', 'Chromium'],                  linux: ['.config', 'chromium'],            windows: { path: ['Chromium', 'User Data'] } },
  edge:     { macos: ['Library', 'Application Support', 'Microsoft Edge'],            linux: ['.config', 'microsoft-edge'],      windows: { path: ['Microsoft', 'Edge', 'User Data'] } },
  vivaldi:  { macos: ['Library', 'Application Support', 'Vivaldi'],                   linux: ['.config', 'vivaldi'],             windows: { path: ['Vivaldi', 'User Data'] } },
  opera:    { macos: ['Library', 'Application Support', 'com.operasoftware.Opera'],    linux: ['.config', 'opera'],               windows: { path: ['Opera Software', 'Opera Stable'], useRoaming: true } },
}
```

- 与 `common.ts` 中的 `CHROMIUM_BROWSERS` 保持**路径一致**。
- `Arc` 在 Linux 下为空数组（未提供）。
- `Opera` 在 Windows 下使用 `Roaming` AppData（`useRoaming: true`）。

### 2. 平台检测

```typescript
switch (process.platform) {
  case 'darwin':  dataPath = config.macos; break
  case 'linux':   dataPath = config.linux; break
  case 'win32':   // 拼接 AppData/Local 或 Roaming
}
```

- 使用原生 `process.platform` 而非 `getPlatform()`，确保在 VS Code 扩展宿主（可能经过 webpack/esbuild 且环境不同）中也能正确执行。

### 3. 扩展检测算法

```typescript
for (const { browser, path: browserBasePath } of browserPaths) {
  // 1. 读取浏览器数据目录
  const browserProfileEntries = await readdir(browserBasePath, { withFileTypes: true })

  // 2. 筛选 Profile 目录（Default 或 Profile *）
  const profileDirs = browserProfileEntries
    .filter(entry => entry.isDirectory())
    .filter(entry => entry.name === 'Default' || entry.name.startsWith('Profile '))
    .map(entry => entry.name)

  // 3. 在每个 Profile 下检查 Extensions/<id>
  for (const profile of profileDirs) {
    for (const extensionId of extensionIds) {
      const extensionPath = join(browserBasePath, profile, 'Extensions', extensionId)
      try {
        await readdir(extensionPath)
        return { isInstalled: true, browser }
      } catch {
        // 继续检查下一个
      }
    }
  }
}
```

#### Extension ID 列表

```typescript
function getExtensionIds(): string[] {
  return process.env.USER_TYPE === 'ant'
    ? [PROD_EXTENSION_ID, DEV_EXTENSION_ID, ANT_EXTENSION_ID]
    : [PROD_EXTENSION_ID]
}
```

- **Production**: `fcoeoabgfenejglbffodgkkbkcdhcgfn`
- **Dev** (ant-only): `dihbgbndebgnbjfmelmegjepbnkhlgni`
- **Ant** (ant-only): `dngcpimnedloihjnnfngkgjoidhnaolf`

### 4. 错误处理

- 对 `readdir(browserBasePath)` 的错误使用 `isFsInaccessible(e)` 判断：
  - 若是文件不存在（ENOENT）等不可访问错误，静默跳过该浏览器。
  - 其他错误（如权限拒绝 EACCES 且未被归类为 inaccessible）则**重新抛出**，避免掩盖系统级问题。

## 关键代码路径与文件引用

```
src/utils/claudeInChrome/setupPortable.ts
  ├── getAllBrowserDataPathsPortable()
  │   └── process.platform
  ├── detectExtensionInstallationPortable(browserPaths, log?)
  │   ├── getExtensionIds()
  │   ├── readdir(browserBasePath, { withFileTypes: true })
  │   └── readdir(extensionPath)
  └── isChromeExtensionInstalledPortable(browserPaths, log?)
      └── detectExtensionInstallationPortable()
```

### 调用方分布

| 调用方 | 使用的导出项 | 用途 |
|--------|-------------|------|
| `src/utils/claudeInChrome/setup.ts` | `isChromeExtensionInstalledPortable` | `isChromeExtensionInstalled()` 的核心实现委托。 |
| `src/utils/claudeInChrome/setup.ts`（间接） | `getAllBrowserDataPathsPortable` | 未被 setup.ts 直接使用，但 `isChromeExtensionInstalled()` 内部会调用。 |
| `src/utils/claudeInChrome/common.ts`（类型） | `ChromiumBrowser` | `common.ts` 通过 `export type { ChromiumBrowser } from './setupPortable.js'` 复用该类型。 |

## 依赖与外部交互

| 依赖 | 用途 |
|------|------|
| `fs/promises` | `readdir` |
| `os` | `homedir()` |
| `path` | `join()` |
| `src/utils/errors.js` | `isFsInaccessible()` |

### 零外部运行时状态依赖

- 不导入 `src/utils/platform.js`。
- 不导入 `src/utils/debug.js`（日志通过可选的 `log?: Logger` 回调参数注入）。
- 不依赖任何全局配置或环境变量（除 `process.env.USER_TYPE` 和 `process.platform`）。

这使得 `setupPortable.ts` 可以被安全地打包进 VS Code 扩展、测试环境、甚至浏览器端的 Web Worker（若提供 polyfill）。

## 风险、边界与改进建议

### 风险

1. **与 `common.ts` 的配置重复且需手动同步**：`CHROMIUM_BROWSERS` 和 `BROWSER_DETECTION_ORDER` 在 `setupPortable.ts` 与 `common.ts` 中几乎完全镜像。若某浏览器更新数据路径（如 Arc 新增 Linux 支持），开发者极易遗漏其中一处。
2. **Profile 目录名称的硬编码假设**：只检测 `Default` 和 `Profile ` 前缀的目录。某些企业环境或用户自定义的 profile 名称（如 `Work`）不会被扫描，导致漏检。
3. **Extension 目录存在 ≠ 扩展已启用**：文件系统检测只能证明扩展曾经安装过，无法判断用户是否在当前 profile 中禁用了它。
4. **`isFsInaccessible` 的语义边界**：该函数将某些权限错误归类为 "inaccessible" 并静默跳过，若企业策略导致目录不可读，检测会静默返回 false，用户无感知。

### 边界

- `Arc` 在 Linux 下返回空路径，因此 Linux 用户即使通过 Wine/容器运行 Arc，也不会被检测到。
- 检测是**顺序短路**的：按 `BROWSER_DETECTION_ORDER` 找到第一个含扩展的浏览器即返回，不会报告多个浏览器同时安装的情况。
- `isChromeExtensionInstalled()` 的自动路径获取仅在 Node 环境下有效（依赖 `process.platform` 和 `os.homedir()`）。

### 改进建议

1. **统一浏览器配置源**：将 `CHROMIUM_BROWSERS` 提取为独立的 JSON 文件（如 `chromiumBrowsers.json`），`common.ts` 和 `setupPortable.ts` 同时导入，消除重复维护。
2. **支持自定义 Profile 名称**：放宽 profile 目录过滤条件，改为检测所有目录（或增加常见自定义名称白名单），提升企业/高级用户场景的覆盖率。
3. **扩展启用状态校验（进阶）**：在检测到扩展目录后，尝试读取该 profile 的 `Preferences` 或 `Secure Preferences` JSON，检查扩展 ID 是否在启用列表中。这能显著降低"已安装但未启用"的误判。
4. **增加单元测试**：使用 `memfs` 或临时目录 mock 构建典型的 Chrome profile 结构，覆盖：
   - 扩展在 `Default` 中检测到。
   - 扩展在 `Profile 1` 中检测到。
   - 多浏览器同时安装时的优先返回顺序。
   - 目录不存在时的静默跳过行为。
5. **日志增强**：在 `detectExtensionInstallationPortable` 中增加扫描的浏览器总数、profile 总数、检测耗时等结构化日志，便于分析扩展安装率的统计偏差。
