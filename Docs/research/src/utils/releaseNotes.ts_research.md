# 研究文档：src/utils/releaseNotes.ts

## 场景与职责

`releaseNotes.ts` 负责 Claude Code 的**版本更新日志（Release Notes）生命周期管理**。由于 Ink 的静态渲染机制难以在首次渲染后动态插入新组件，模块采用“后台拉取 + 本地缓存 + 下次启动展示”的架构：

- 用户升级到新版本后，后台从 GitHub 拉取 `CHANGELOG.md`；
- 缓存到本地文件（`~/.claude/cache/changelog.md`）并记录拉取时间；
- 下次启动时，同步从内存缓存中读取并展示自上次已读版本以来的更新摘要。

该模块同时支持：
- **Ant 内部构建**：直接读取构建时注入的 `MACRO.VERSION_CHANGELOG`，不走网络。
- **迁移逻辑**：将旧版基于 `globalConfig.cachedChangelog` 的存储迁移到文件缓存。

---

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| `fetchAndStoreChangelog()` | 后台异步拉取 GitHub 原始 CHANGELOG，写入本地缓存文件，并更新 `config.changelogLastFetched` 时间戳。 |
| `getStoredChangelog()` / `getStoredChangelogFromMemory()` | 分别提供异步（首次加载）与同步（React render 路径）两种读取接口，确保 TUI 渲染时无需等待 I/O。 |
| `parseChangelog()` | 将 Markdown 格式的 CHANGELOG 解析为 `Record<version, notes[]>`，支持 `## X.X.X - YYYY-MM-DD` 格式。 |
| `getRecentReleaseNotes()` | 根据当前版本与上一次已读版本，筛选出“新版本”的 release notes，最多展示 5 条，按版本从新到旧排序。 |
| `getAllReleaseNotes()` | 返回所有版本的 release notes，按版本从旧到新排序，供 `/release-notes` 命令使用。 |
| `checkForReleaseNotes()` / `checkForReleaseNotesSync()` | 分别提供异步与同步版本检测入口，是 TUI 启动路径（`setup.ts`）与 headless 路径的调用点。 |
| `migrateChangelogFromConfig()` | 将旧版 config 中的 `cachedChangelog` 字段迁移到文件缓存，并删除已废弃字段，防止后续保存回写。 |

---

## 具体技术实现

### 1. 缓存架构：三层缓存

```
GitHub RAW (网络层)
    ↓ 拉取
~/.claude/cache/changelog.md (磁盘缓存)
    ↓ 首次 readFile
changelogMemoryCache (进程内存缓存)
    ↓ 后续同步读取
React render / headless sync check
```

- `changelogMemoryCache: string | null` 在模块加载时为 `null`。
- `getStoredChangelog()` 首次调用时读磁盘并填充内存缓存；后续直接返回内存值。
- `getStoredChangelogFromMemory()` 专供 React render 等不能 await 的场景使用。

### 2. 网络拉取：`fetchAndStoreChangelog`

```ts
const RAW_CHANGELOG_URL =
  'https://raw.githubusercontent.com/anthropics/claude-code/refs/heads/main/CHANGELOG.md'
```

- 使用 `axios.get(RAW_CHANGELOG_URL)` 拉取。
- **幂等写保护**：若返回内容与内存缓存完全一致，则跳过写盘，避免 `saveGlobalConfig` 的脏检查被 `Date.now()` 时间戳击穿。
- 写盘使用 `fs/promises.writeFile(cachePath, changelogContent, { encoding: 'utf-8' })`。
- 更新 `config.changelogLastFetched = Date.now()`。

**跳过条件**：
- `getIsNonInteractiveSession()` 为 true（非交互模式不需要展示 release notes）。
- `isEssentialTrafficOnly()` 为 true（用户关闭了非必要网络流量）。

### 3. CHANGELOG 解析：`parseChangelog`

解析逻辑：
1. `content.split(/^## /gm).slice(1)` 按二级标题切分版本块。
2. 每块第一行是版本号（取 ` - ` 前的部分）。
3. 后续行中过滤以 `- ` 开头的 bullet points，去掉前缀与首尾空格。
4. 返回 `Record<string, string[]>`。

容错：
- 空内容返回 `{}`。
- 任何异常被 catch 后 `logError` 并返回 `{}`，避免启动崩溃。

### 4. 版本比较：`getRecentReleaseNotes`

- 使用 `semver.coerce()` 将版本号（可能带 SHA 后缀）归一化为可比较的 semver 对象。
- 使用本地 `./semver.js` 的 `gt()` 做版本大小比较。
- 过滤出 `> previousVersion` 的所有版本 notes，flatMap 后取前 5 条。

### 5. Ant 构建特殊路径

```ts
if (process.env.USER_TYPE === 'ant') {
  const changelog = MACRO.VERSION_CHANGELOG
  // ...
}
```

Ant 构建时 CHANGELOG 在打包阶段通过 `MACRO.VERSION_CHANGELOG` 注入，无需网络即可立即展示。

---

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/utils/releaseNotes.ts:28-31` | CHANGELOG URL 定义。 |
| `src/utils/releaseNotes.ts:55-76` | 迁移逻辑 `migrateChangelogFromConfig`。 |
| `src/utils/releaseNotes.ts:82-119` | 后台拉取与存储 `fetchAndStoreChangelog`。 |
| `src/utils/releaseNotes.ts:126-149` | 缓存读取接口（异步 + 同步）。 |
| `src/utils/releaseNotes.ts:156-196` | Markdown 解析器 `parseChangelog`。 |
| `src/utils/releaseNotes.ts:207-240` | 最近 release notes 筛选 `getRecentReleaseNotes`。 |
| `src/utils/releaseNotes.ts:287-360` | 异步/同步检测入口 `checkForReleaseNotes` / `checkForReleaseNotesSync`。 |
| `src/setup.ts:386-393` | TUI 启动时调用 `checkForReleaseNotes` 并预取 `getRecentActivity`。 |
| `src/main.tsx:200` | `migrateChangelogFromConfig` 在 `runMigrations` 中被调用。 |
| `src/commands/release-notes/index.ts` | `/release-notes` 命令注册。 |
| `src/commands/release-notes/release-notes.ts` | 命令实现，调用 `getAllReleaseNotes`。 |
| `src/components/LogoV2/LogoV2.tsx` / `feedConfigs.tsx` | Logo 组件可能消费 release notes 数据。 |

---

## 依赖与外部交互

- **第三方库**：`axios`（HTTP 拉取）、`semver`（版本解析与比较）。
- **内部依赖**：
  - `./config.js`：`getGlobalConfig`, `saveGlobalConfig`
  - `./envUtils.js`：`getClaudeConfigHomeDir`
  - `./errors.js`：`toError`
  - `./log.js`：`logError`
  - `./privacyLevel.js`：`isEssentialTrafficOnly`
  - `./semver.js`：`gt`
  - `../bootstrap/state.js`：`getIsNonInteractiveSession`
- **文件系统**：`~/.claude/cache/changelog.md`。
- **网络**：`raw.githubusercontent.com`（GitHub CDN）。

---

## 风险、边界与改进建议

### 风险与边界

1. **网络拉取无超时**：`axios.get(RAW_CHANGELOG_URL)` 没有显式 `timeout` 配置，若 GitHub 连接异常，可能长时间挂起。虽然调用方 `checkForReleaseNotes` 中使用了 `.catch()` 做 fire-and-forget，但仍会占用一个 TCP 连接直到默认超时（通常 ~30-60s）。

2. **解析器脆弱性**：`parseChangelog` 依赖 `^## ` 正则切分，若 CHANGELOG 格式变化（如使用 `###` 做版本号、或没有 `- ` 前缀的 bullet）会导致解析为空或漏掉内容。当前没有 schema 校验。

3. **版本号比较依赖 `semver.coerce`**：若版本号包含非 semver 前缀（如 `v1.2.3-beta+build`），`coerce` 会丢失预发布信息，可能导致 `gt` 比较结果与预期不符。不过目前 CHANGELOG 版本号都是干净 semver。

4. **并发写风险**：`writeFile` 没有文件锁，若多进程同时启动并都触发 `fetchAndStoreChangelog`，可能产生 torn write。但由于写入的是纯文本且内容相同，实际风险极低。

5. **Ant 构建的 `MACRO` 宏**：`MACRO.VERSION_CHANGELOG` 是构建时注入的全局变量，TypeScript 类型系统中没有显式声明，依赖构建系统保证存在。

### 改进建议

1. **增加 axios timeout**：在 `fetchAndStoreChangelog` 中设置 `axios.get(url, { timeout: 10000 })`，防止网络抖动导致连接长期占用。

2. **解析器增强容错**：可引入更宽松的 markdown AST 解析（如 `marked` 的 lexer）替代正则切分，降低格式变更风险。但考虑到模块体积与启动性能，当前正则方案在受控 CHANGELOG 场景下可接受。

3. **缓存过期策略**：当前只记录 `changelogLastFetched` 但不用于任何过期判断。可考虑若距离上次拉取超过 7 天，则强制重新拉取，避免用户长期看到旧 CHANGELOG。

4. **迁移逻辑幂等性**：`migrateChangelogFromConfig` 使用 `writeFile(..., { flag: 'wx' })` 防止覆盖，但如果迁移后用户手动删除了缓存文件，旧 config 中的 `cachedChangelog` 已被清除，无法再次迁移。这是有意设计（一次性迁移），但可在文档中明确说明。

5. **减少 axios 依赖**：如果项目已全面使用 `fetch`（Bun/Node 18+），可考虑用原生 `fetch` 替换 `axios`，减少 bundle 体积。
