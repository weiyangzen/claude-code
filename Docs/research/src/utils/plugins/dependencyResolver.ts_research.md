# dependencyResolver.ts 深度研究文档

## 场景与职责

`dependencyResolver.ts` 实现 Claude Code 插件系统的依赖解析引擎，采用 **apt-style 语义**：依赖是"存在性保证"而非模块图。即插件A依赖插件B意味着"B的命名空间组件（MCP服务器、命令、Agent）在A运行时必须可用"。

核心应用场景：
1. **安装时依赖解析**：DFS遍历传递依赖，检测循环依赖和跨市场边界
2. **加载时依赖验证**：固定点检查，降级不满足依赖的插件（会话本地，不写设置）
3. **卸载前反向依赖检查**：警告用户哪些插件将因卸载而损坏

## 功能点目的

### 1. 依赖闭包解析 (`resolveDependencyClosure`)
- **目的**：安装插件前计算完整依赖树
- **关键行为**：
  - 根插件永不被跳过（支持重新安装已启用插件）
  - 已启用依赖被跳过（避免意外设置写入）
  - 跨市场依赖默认被阻止（安全边界）
  - 支持根市场的 `allowCrossMarketplaceDependenciesOn` 白名单

### 2. 加载时降级验证 (`verifyAndDemote`)
- **目的**：会话启动时确保所有启用插件的依赖满足
- **机制**：
  - 固定点迭代：降级A可能破坏依赖A的B，需循环至稳定
  - 区分错误原因：`'not-enabled'`（存在但禁用）vs `'not-found'`（完全不存在）
  - 裸依赖支持：`@inline` 插件的裸名依赖通过名称匹配

### 3. 反向依赖查找 (`findReverseDependents`)
- **目的**：卸载/禁用插件前警告用户影响范围
- **匹配逻辑**：
  - 全限定依赖：`qualified === pluginId`
  - 裸依赖：`qualified === targetName`（名称匹配）

### 4. 辅助功能
- `qualifyDependency`：将裸依赖规范化为 `name@marketplace` 格式
- `getEnabledPluginIdsForScope`：获取指定作用域的已启用插件ID集合
- `formatDependencyCountSuffix` / `formatReverseDependentsSuffix`：CLI消息格式化

## 具体技术实现

### 核心类型定义
```typescript
export type ResolutionResult =
  | { ok: true; closure: PluginId[] }
  | { ok: false; reason: 'cycle'; chain: PluginId[] }
  | { ok: false; reason: 'not-found'; missing: PluginId; requiredBy: PluginId }
  | { ok: false; reason: 'cross-marketplace'; dependency: PluginId; requiredBy: PluginId }
```

### 依赖规范化算法 (`qualifyDependency`)
```
qualifyDependency(dep, declaringPluginId)
  ├─ dep 已包含 @ → 直接返回（已是全限定）
  ├─ 解析 declaringPluginId 获取其 marketplace
  │   └─ 无 marketplace 或为 'inline' → 返回裸名（无法推断）
  └─ 返回 `${dep}@${marketplace}`（继承声明插件的市场）
```

### DFS闭包解析算法 (`resolveDependencyClosure`)
```
resolveDependencyClosure(rootId, lookup, alreadyEnabled, allowedCrossMarketplaces)
  ├─ 初始化：closure=[], visited=Set(), stack=[]
  └─ walk(rootId, rootId)
      ├─ 跳过检查：id !== rootId && alreadyEnabled.has(id) → return null
      ├─ 跨市场检查：
      │   ├─ 解析 idMarketplace
      │   ├─ idMarketplace !== rootMarketplace && !allowedCrossMarketplaces.has(idMarketplace)
      │   └─ 是 → return {ok: false, reason: 'cross-marketplace', ...}
      ├─ 循环检测：stack.includes(id) → return {ok: false, reason: 'cycle', chain}
      ├─ 去重：visited.has(id) → return null
      ├─ 标记：visited.add(id)
      ├─ 查询：entry = await lookup(id)
      │   └─ !entry → return {ok: false, reason: 'not-found', ...}
      ├─ 递归：stack.push(id)
      │   for each rawDep in entry.dependencies
      │       dep = qualifyDependency(rawDep, id)
      │       err = await walk(dep, id)
      │       if err return err
      └─ 收集：closure.push(id), stack.pop(), return null
```

### 固定点降级算法 (`verifyAndDemote`)
```
verifyAndDemote(plugins)
  ├─ 构建索引：
  │   ├─ known = Set(所有插件source)
  │   ├─ enabled = Set(启用插件source)
  │   ├─ knownByName = Set(所有插件name)
  │   └─ enabledByName = Map<name, count>（多市场支持的多重集）
  ├─ 固定点循环：
  │   changed = true
  │   while changed
  │       changed = false
  │       for each 启用插件 p
  │           for each rawDep in p.manifest.dependencies
  │               dep = qualifyDependency(rawDep, p.source)
  │               isBare = !parsePluginIdentifier(dep).marketplace
  │               satisfied = isBare ? enabledByName.get(dep) > 0 : enabled.has(dep)
  │               if !satisfied
  │                   enabled.delete(p.source)
  │                   更新 enabledByName 计数
  │                   errors.push({type: 'dependency-unsatisfied', ...})
  │                   changed = true
  │                   break
  └─ 返回：demoted = 原启用但现禁用的插件, errors
```

## 关键代码路径与文件引用

### 内部依赖
```
dependencyResolver.ts
  ├─ types/plugin.ts: LoadedPlugin, PluginError
  ├─ settings/constants.ts: EditableSettingSource
  ├─ settings/settings.ts: getSettingsForSource
  ├─ pluginIdentifier.ts: parsePluginIdentifier
  └─ schemas.ts: PluginId
```

### 外部调用方
| 调用方 | 调用函数 | 用途 |
|--------|----------|------|
| `pluginInstallationHelpers.ts` | `resolveDependencyClosure` | 安装前解析依赖树 |
| `pluginLoader.ts` | `verifyAndDemote` | 加载时验证依赖 |
| `marketplaceManager.ts` | `findReverseDependents` | 卸载前检查影响 |
| `pluginOperations.ts` | `getEnabledPluginIdsForScope`, `formatDependencyCountSuffix` | 安装操作与消息 |
| `doctor.ts` | `verifyAndDemote` | 诊断依赖问题 |

## 依赖与外部交互

### 输入依赖
- **PluginManifest.dependencies**: `string[]` 格式的依赖声明
- **Settings.enabledPlugins**: 各作用域的已启用插件配置
- **Marketplace lookup**: 异步查询插件元数据

### 安全边界
1. **跨市场依赖阻止**：防止从可信市场安装时静默拉取不可信市场内容
2. **白名单机制**：仅根市场的 `allowCrossMarketplaceDependenciesOn` 生效，不传递信任

## 风险、边界与改进建议

### 已知风险

1. **循环依赖检测的栈深度**
   - 风险：极深依赖链可能导致栈溢出
   - 现状：使用显式 `stack` 数组而非递归，风险可控

2. **裸依赖的歧义性**
   - 风险：`@inline` 插件的裸依赖可能匹配多个市场的同名插件
   - 现状：使用 `enabledByName` 多重集，只要有一个市场的同名插件启用即满足

3. **lookup 函数的外部依赖**
   - 风险：异步 lookup 可能在网络请求中挂起
   - 现状：调用方控制 lookup 实现，可添加超时

### 边界条件

| 场景 | 行为 |
|------|------|
| 根插件已在 enabled 集合 | 仍解析（支持重新安装） |
| 依赖已在 enabled 集合 | 跳过，不加入 closure |
| 跨市场依赖 + 白名单匹配 | 允许 |
| 跨市场依赖 + 无白名单 | 返回 cross-marketplace 错误 |
| 循环依赖 A→B→C→A | 返回 cycle 错误，chain 包含完整环 |
| lookup 返回 null | 返回 not-found 错误 |
| 降级后新产生的不满足 | 固定点循环处理直至稳定 |

### 改进建议

1. **依赖版本约束**
   - 当前：仅检查存在性，无版本语义
   - 建议：支持 semver 范围约束（如 `"plugin@marketplace": ">=1.0.0 <2.0.0"`）

2. **可选依赖支持**
   - 当前：所有依赖都是强依赖
   - 建议：支持 `optionalDependencies`，不满足时不降级

3. **依赖冲突检测**
   - 当前：不检测同一插件多版本冲突
   - 建议：检测并警告 diamond dependency 问题

4. **性能优化**
   - 当前：每次加载全量遍历所有插件
   - 建议：增量验证，仅检查变更插件的依赖子图
