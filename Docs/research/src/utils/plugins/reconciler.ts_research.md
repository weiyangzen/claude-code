# 研究报告：src/utils/plugins/reconciler.ts

## 场景与职责

`reconciler.ts` 是 Claude Code 插件市场的**声明式协调器（Marketplace Reconciler）**，负责将“用户意图层”（settings.json 中声明的 `extraKnownMarketplaces`）与“物化状态层”（`known_marketplaces.json` 及磁盘上的市场缓存）保持一致。

它是插件三层模型中的**第二层（Layer 2）**：
- **Layer 1**：意图（settings）
- **Layer 2**：物化（`~/.claude/plugins/`）— 即本文件
- **Layer 3**：活跃组件（AppState）— 由 `refresh.ts` 负责

## 功能点目的

### 1. `diffMarketplaces()` — 差异检测
- **目的**：比较“声明的市场列表”与“已物化的市场列表”，找出需要安装或更新的市场。
- **输出结构**：
  - `missing`：声明了但尚未物化的市场（需要安装）
  - `sourceChanged`：声明与物化的 `source` 不一致的市场（需要更新/替换）
  - `upToDate`：两者一致的市场

- **路径规范化**：
  - 对于 `directory` 和 `file` 类型的本地来源，若路径为相对路径（如 `./my-marketplace`），则解析为绝对路径。
  - **Git worktree 处理**：使用 `findCanonicalGitRoot()` 将相对路径解析到主仓库根目录，而非当前 worktree 的 cwd。这避免了多 worktree 场景下，不同会话反复覆盖共享的 `known_marketplaces.json` 条目，也防止删除 worktree 后留下死路径。

- **`sourceIsFallback` 特殊处理**：
  - 当声明带有 `sourceIsFallback: true` 时（如隐式官方市场），只要该市场已物化（无论来源是什么），就视为 `upToDate`。这保护了 seed dir 或镜像源已注册的市场不被强制覆盖为默认的 GitHub 源。

### 2. `reconcileMarketplaces()` — 协调执行
- **目的**：根据 `diffMarketplaces` 的结果，实际执行安装/更新操作，使 `known_marketplaces.json` 与意图一致。
- **特性**：
  - **幂等**：同一来源多次安装不会重复克隆（`addMarketplaceSource` 内部会检查 `alreadyMaterialized`）。
  - **只增不删**：永远不会删除 `known_marketplaces.json` 中的条目（即使 settings 中已移除声明）。这是为了避免用户手动添加的市场被意外清除。
  - **跳过机制**：支持 `opts.skip` 回调，用于 zip cache 模式跳过不支持的市场来源类型。
  - **本地路径保护**：对于 `sourceChanged` 的本地路径条目，若声明的路径不存在，则跳过更新并保留现有物化条目。这保护了多 checkout 场景下无法规范化的路径。

## 具体技术实现

### 关键数据结构

```ts
export type MarketplaceDiff = {
  missing: string[]
  sourceChanged: Array<{
    name: string
    declaredSource: MarketplaceSource
    materializedSource: MarketplaceSource
  }>
  upToDate: string[]
}

export type ReconcileOptions = {
  skip?: (name: string, source: MarketplaceSource) => boolean
  onProgress?: (event: ReconcileProgressEvent) => void
}

export type ReconcileResult = {
  installed: string[]
  updated: string[]
  failed: Array<{ name: string; error: string }>
  upToDate: string[]
  skipped: string[]
}
```

### `diffMarketplaces` 算法

```ts
for (const [name, intent] of Object.entries(declared)) {
  const state = materialized[name]
  const normalizedIntent = normalizeSource(intent.source, opts?.projectRoot)

  if (!state) {
    missing.push(name)
  } else if (intent.sourceIsFallback) {
    upToDate.push(name)
  } else if (!isEqual(normalizedIntent, state.source)) {
    sourceChanged.push({ name, declaredSource: normalizedIntent, materializedSource: state.source })
  } else {
    upToDate.push(name)
  }
}
```

### `reconcileMarketplaces` 执行流程

1. 读取 `declared = getDeclaredMarketplaces()`
2. 读取 `materialized = loadKnownMarketplacesConfig()`（失败则视为空对象）
3. 调用 `diffMarketplaces(declared, materialized, { projectRoot: getOriginalCwd() })`
4. 构建工作列表 `work = missing.map(...) + sourceChanged.map(...)`
5. 应用 `skip` 过滤和本地路径存在性检查
6. 串行遍历 `toProcess`，对每个条目调用 `addMarketplaceSource(source)`
   - `action === 'install'` → 推入 `installed`
   - `action === 'update'` → 推入 `updated`
7. 返回结果对象

### `normalizeSource` 实现

```ts
function normalizeSource(source: MarketplaceSource, projectRoot?: string): MarketplaceSource {
  if (
    (source.source === 'directory' || source.source === 'file') &&
    !isAbsolute(source.path)
  ) {
    const base = projectRoot ?? getOriginalCwd()
    const canonicalRoot = findCanonicalGitRoot(base)
    return {
      ...source,
      path: resolve(canonicalRoot ?? base, source.path),
    }
  }
  return source
}
```

### 文件引用

- `getDeclaredMarketplaces()` / `addMarketplaceSource()` / `loadKnownMarketplacesConfig()` → `src/utils/plugins/marketplaceManager.ts`
- `isLocalMarketplaceSource()` / `KnownMarketplacesFile` / `MarketplaceSource` → `src/utils/plugins/schemas.ts`
- `findCanonicalGitRoot()` → `src/utils/git.ts`
- `getOriginalCwd()` → `src/bootstrap/state.js`
- `pathExists()` → `src/utils/file.js`
- `errorMessage()` → `src/utils/errors.js`

## 依赖与外部交互

### 上游调用方
- `src/utils/plugins/headlessPluginInstall.ts`：`installPluginsForHeadless()` 的核心调用点，用于 headless/CCR 模式的市场协调。
- `src/services/plugins/PluginInstallationManager.ts`：交互式安装管理器在后台安装时调用。
- `src/ink/reconciler.ts`：与 Ink UI 的 reconciler 同名，但无直接关联（只是文件名巧合）。

### 下游依赖
- `marketplaceManager.ts`：获取声明的市场列表、加载已物化配置、执行实际的市场源添加。
- `schemas.ts`：市场来源类型和已物化配置的类型定义。
- `git.ts`：git worktree 的根目录规范化。

## 风险、边界与改进建议

### 已知风险

1. **`reconcileMarketplaces` 的串行执行瓶颈**
   - 市场安装/更新通过 `for` 循环串行执行，每个市场可能涉及 git clone、网络下载等 I/O 密集型操作。若用户声明了多个市场，启动时间会线性增长。
   - 当前设计可能是为了避免并发 clone 对同一目标目录的竞态，但对于来源独立的市场，完全可以并行化。

2. **`diffMarketplaces` 的 `.git` 读取开销**
   - `normalizeSource` 调用 `findCanonicalGitRoot(base)`，该函数会向上遍历目录树寻找 `.git` 目录。在深层嵌套的项目目录中，这可能涉及多次 `stat` 调用。
   - 虽然 `findCanonicalGitRoot` 内部可能有缓存，但注释未明确说明，且该操作在每次 `diffMarketplaces` 调用时都会执行。

3. **`sourceIsFallback` 的过度保护**
   - `sourceIsFallback: true` 时，任何已物化的来源都被视为有效。若用户显式在 settings 中修改了官方市场的来源（如从 GitHub 改为内部镜像），`sourceIsFallback` 会阻止更新到新的声明来源。
   - 实际上，隐式官方市场的声明带有 `sourceIsFallback: true`，而用户显式声明的市场不带此标志。因此正常场景下不会出问题，但若用户手动构造了带 `sourceIsFallback` 的 settings，可能产生意外行为。

4. **只增不删的累积问题**
   - `reconcileMarketplaces` 永远不会从 `known_marketplaces.json` 中删除条目。长期使用的 CLI 实例可能会累积大量已废弃的市场条目，占用磁盘空间并拖慢市场加载。
   - 当前缓解：市场缓存本身不大，且 `known_marketplaces.json` 只是元数据。但物化的市场目录（clone 的仓库）可能很大。

### 边界情况

- **`known_marketplaces.json` 损坏或不可读**：`reconcileMarketplaces` 捕获异常后将 `materialized` 设为空对象，所有声明的市场都会被视为 `missing` 并重新安装。
- **`skip` 回调过滤后的空工作列表**：直接返回 `upToDate` 和 `skipped`，不执行任何 I/O。
- **本地路径不存在**：`sourceChanged` 的本地路径条目会被跳过；`missing` 的本地路径条目不会被跳过（因为 `addMarketplaceSource` 会正常报错，用户应该看到这个错误）。

### 改进建议

1. **并行化市场安装**
   - 对于来源独立的市场，可将 `toProcess` 按来源分组后并行执行，或至少使用 `Promise.all` 对无 I/O 冲突的条目并行处理。需要确保 `addMarketplaceSource` 内部对 `known_marketplaces.json` 的写操作有文件锁或串行队列保护。

2. **缓存 `findCanonicalGitRoot` 结果**
   - 在 `normalizeSource` 或 `diffMarketplaces` 级别增加一个 session-scoped 的 `Map<basePath, canonicalRoot>` 缓存，避免对同一项目路径重复遍历 `.git`。

3. **增加市场清理机制**
   - 可考虑增加一个“孤儿市场”检测逻辑：若某个市场在 `known_marketplaces.json` 中已存在超过 N 天，且当前任何 settings 源均未声明它，则打印警告或提供 `/plugins cleanup` 命令供用户手动清理。
   - 自动删除风险较高，因为用户可能通过 `--add-dir` 或 managed settings 临时禁用市场，不希望数据被自动清除。

4. **增强 `diffMarketplaces` 的测试覆盖**
   - 建议增加以下场景的单元测试：
     - 相对路径在 git worktree 中的规范化
     - `sourceIsFallback` 对 `sourceChanged` 的抑制
     - 多个声明源对同一市场的覆盖优先级
     - Windows 路径分隔符的规范化行为
