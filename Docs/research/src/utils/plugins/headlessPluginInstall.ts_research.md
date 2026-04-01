# headlessPluginInstall.ts 深度研究文档

## 场景与职责

`headlessPluginInstall.ts` 为 Claude Code 的无头模式（Headless/CCR 模式）提供插件安装能力。与交互式模式不同，无头模式没有 UI 需要更新，因此该模块跳过 AppState 更新，专注于核心安装逻辑。

核心场景：
1. **CCR (Claude Code Runner) 环境**：CI/CD 流水线中的自动化插件安装
2. **容器化部署**：ZIP 缓存模式支持挂载卷存储插件
3. **种子市场初始化**：`CLAUDE_CODE_PLUGIN_SEED_DIR` 预置市场支持

## 功能点目的

### 1. 无头插件安装 (`installPluginsForHeadless`)
- **入口函数**：无头模式插件安装的主入口
- **返回值**：`boolean` - 是否有插件被安装（调用方应据此刷新 MCP）
- **ZIP 缓存模式适配**：
  - 创建 ZIP 缓存目录结构
  - 跳过不支持的源类型
  - 注册会话清理回调

### 2. 种子市场注册
- **目的**：支持 `CLAUDE_CODE_PLUGIN_SEED_DIR` 预置市场
- **行为**：
  - 幂等注册（重复调用无影响）
  - 若注册改变状态，清除市场/插件缓存
  - 确保首次启动时种子插件可见

### 3. 市场对账 (`reconcileMarketplaces`)
- **用途**：将声明的市场意图与实际状态同步
- **ZIP 缓存适配**：通过 `skip` 回调过滤不支持的源类型
- **进度回调**：记录安装成功/失败的调试日志

### 4. 下架插件检测 (`detectAndUninstallDelistedPlugins`)
- **用途**：执行黑名单策略，卸载被下架的插件
- **返回值**：新下架的插件列表

### 5. 遥测上报
- **事件**：`tengu_headless_plugin_install`
- **指标**：
  - `marketplaces_installed`: 安装/更新的市场数量
  - `delisted_count`: 下架插件数量

## 具体技术实现

### 主流程算法
```
installPluginsForHeadless()
  ├─ 检测 ZIP 缓存模式
  ├─ 注册种子市场 (registerSeedMarketplaces)
  │   └─ 若改变状态 → clearMarketplacesCache(), clearPluginCache()
  ├─ 确保 ZIP 缓存目录存在
  ├─ 获取声明的市场 (getDeclaredMarketplaces)
  │   └─ 包含隐式官方市场（当启用插件引用时）
  ├─ 初始化指标和 pluginsChanged 标志
  ├─ 若无可声明市场 → 记录调试日志
  └─ 否则执行市场对账
      ├─ reconcileMarketplaces({skip, onProgress})
      │   ├─ skip: ZIP模式下过滤不支持源
      │   └─ onProgress: 记录安装/失败日志
      ├─ 若有跳过项 → 记录调试日志
      ├─ 若有市场变更 → clearMarketplacesCache(), clearPluginCache()
      ├─ ZIP模式 → syncMarketplacesToZipCache()（离线访问）
      ├─ 检测下架插件 (detectAndUninstallDelistedPlugins)
      │   └─ 若有下架 → pluginsChanged = true
      ├─ 若 pluginsChanged → clearPluginCache()
      ├─ ZIP模式 → registerCleanup(cleanupSessionPluginCache)
      └─ 返回 pluginsChanged
  └─ finally: 上报遥测事件
```

### ZIP 缓存模式集成
```typescript
// 目录结构创建
if (zipCacheMode) {
  await getFsImplementation().mkdir(getZipCacheMarketplacesDir())
  await getFsImplementation().mkdir(getZipCachePluginsDir())
}

// 不支持源跳过
const reconcileResult = await reconcileMarketplaces({
  skip: zipCacheMode
    ? (_name, source) => !isMarketplaceSourceSupportedByZipCache(source)
    : undefined,
  // ...
})

// 会话清理注册
if (zipCacheMode) {
  registerCleanup(cleanupSessionPluginCache)
}
```

## 关键代码路径与文件引用

### 内部依赖
```
headlessPluginInstall.ts
  ├─ services/analytics/index.ts: logEvent
  ├─ cleanupRegistry.ts: registerCleanup
  ├─ debug.ts: logForDebugging
  ├─ diagLogs.ts: withDiagnosticsTiming
  ├─ fsOperations.ts: getFsImplementation
  ├─ log.ts: logError
  ├─ marketplaceManager.ts: 
  │   ├─ clearMarketplacesCache
  │   ├─ getDeclaredMarketplaces
  │   └─ registerSeedMarketplaces
  ├─ pluginBlocklist.ts: detectAndUninstallDelistedPlugins
  ├─ pluginLoader.ts: clearPluginCache
  ├─ reconciler.ts: reconcileMarketplaces
  └─ zipCache.ts / zipCacheAdapters.ts:
      ├─ cleanupSessionPluginCache
      ├─ getZipCacheMarketplacesDir
      ├─ getZipCachePluginsDir
      ├─ isMarketplaceSourceSupportedByZipCache
      ├─ isPluginZipCacheEnabled
      └─ syncMarketplacesToZipCache
```

### 外部调用方
| 调用方 | 用途 |
|--------|------|
| `print.ts` | 无头模式启动时调用，根据返回值决定是否刷新插件状态 |
| `headless.ts` | 主无头入口初始化插件 |

### 调用链
```
print.ts / headless.ts
  └─ installPluginsForHeadless()
      └─ 若返回 true → refreshPluginState()
          └─ clearCommandsCache(), clearAgentDefinitionsCache(), etc.
```

## 依赖与外部交互

### 环境变量
| 变量 | 用途 |
|------|------|
| `CLAUDE_CODE_PLUGIN_USE_ZIP_CACHE` | 启用 ZIP 缓存模式 |
| `CLAUDE_CODE_PLUGIN_SEED_DIR` | 种子市场目录路径 |

### 核心依赖模块
| 模块 | 用途 |
|------|------|
| `marketplaceManager.ts` | 市场声明获取、种子注册、缓存清理 |
| `reconciler.ts` | 市场对账核心逻辑 |
| `zipCache.ts` | ZIP 缓存模式检测与目录管理 |
| `zipCacheAdapters.ts` | 市场数据同步到 ZIP 缓存 |
| `pluginBlocklist.ts` | 下架插件检测与卸载 |
| `cleanupRegistry.ts` | 会话清理回调注册 |

## 风险、边界与改进建议

### 已知风险

1. **种子市场状态变更检测**
   - 风险：`seedChanged` 仅检测注册是否改变状态，不检测种子内容变更
   - 现状：若种子目录内容更新但配置未变，可能使用旧缓存
   - 缓解：手动清除缓存或重启进程

2. **ZIP 缓存跳过逻辑**
   - 风险：某些源类型被跳过可能导致插件不完整
   - 现状：记录调试日志，但无用户可见警告
   - 建议：增加警告或错误上报

3. **异步清理回调**
   - 风险：`cleanupSessionPluginCache` 在进程退出时可能未执行
   - 现状：依赖 `registerCleanup` 的退出钩子机制
   - 缓解：确保 `registerCleanup` 可靠处理进程退出

### 边界条件

| 场景 | 行为 |
|------|------|
| 无可声明市场 | 记录日志，返回 false |
| 市场对账全部失败 | 清除缓存（若有变更），返回 false |
| ZIP 模式 + 不支持的源 | 跳过并记录，继续处理其他 |
| 种子注册失败 | 抛出错误，被捕获后返回 false |
| 下架检测抛出错误 | 被 try-catch 捕获，返回 false |

### 改进建议

1. **失败重试机制**
   - 当前：市场对账失败即记录，无自动重试
   - 建议：对网络相关失败添加指数退避重试

2. **安装报告输出**
   - 当前：仅调试日志和遥测
   - 建议：返回结构化安装报告，供调用方展示或记录

3. **部分成功处理**
   - 当前：布尔返回值无法表达部分成功
   - 建议：返回 `{success: boolean, installed: string[], failed: string[]}`

4. **并发控制**
   - 当前：市场对账内部并发由 `reconciler.ts` 控制
   - 建议：暴露并发限制参数，适应不同环境资源限制
