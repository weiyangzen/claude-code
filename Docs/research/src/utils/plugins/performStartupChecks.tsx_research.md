# performStartupChecks.tsx 深度研究文档

## 场景与职责

`performStartupChecks.tsx` 是 Claude Code 启动时执行**插件系统初始化**的入口模块。它在用户信任当前目录后，启动后台插件市场注册和插件安装流程，确保插件系统就绪。

### 核心职责

1. **信任检查**：验证用户是否已接受信任对话框（安全前提）
2. **种子市场注册**：注册 `CLAUDE_CODE_PLUGIN_SEED_DIR` 中的预置市场
3. **缓存管理**：种子市场变更时清除相关缓存
4. **后台安装启动**：启动 `performBackgroundPluginInstallations` 进行插件安装

### 在系统架构中的位置

```
启动流程
    ├── cli.tsx
    │   └── 显示信任对话框，等待用户确认
    └── screens/REPL.tsx
        └── 初始化完成后
            └── performStartupChecks()  ← 本文件入口
                └── 插件系统初始化
```

---

## 功能点目的

### 1. 安全检查

```typescript
// SECURITY: 此函数只在用户信任当前目录后调用
if (!checkHasTrustDialogAccepted()) {
  logForDebugging('Trust not accepted - skipping plugin installations')
  return
}
```

这是关键的安全机制，防止恶意仓库自动安装插件。

### 2. 种子市场注册

```typescript
const seedChanged = await registerSeedMarketplaces()
```

种子市场（`CLAUDE_CODE_PLUGIN_SEED_DIR`）是预置的只读插件目录：
- 用于容器镜像中预装插件
- 避免重复克隆
- 支持多个种子目录（PATH 风格，按优先级）

### 3. 缓存一致性

```typescript
if (seedChanged) {
  clearMarketplacesCache()
  clearPluginCache('performStartupChecks: seed marketplaces changed')
  setAppState(prev => ({ ...prev, plugins: { ...prev.plugins, needsRefresh: true } }))
}
```

种子市场变更时：
1. 清除市场缓存
2. 清除插件缓存
3. 标记需要刷新（提示用户运行 `/reload-plugins`）

### 4. 后台安装

```typescript
await performBackgroundPluginInstallations(setAppState)
```

- 不阻塞启动流程
- 通过 `setAppState` 报告进度
- 错误通过 AppState 通知用户

---

## 具体技术实现

### 核心函数

```typescript
export async function performStartupChecks(setAppState: SetAppState): Promise<void>
```

### 完整流程

```
performStartupChecks(setAppState)
    │
    ├── 1. 信任检查
    │   └── checkHasTrustDialogAccepted()
    │       └── 未接受 → 跳过，返回
    │
    ├── 2. 种子市场注册
    │   └── registerSeedMarketplaces()  # marketplaceManager.ts
    │       └── 返回 seedChanged: boolean
    │
    ├── 3. 缓存处理（如果 seedChanged）
    │   ├── clearMarketplacesCache()    # 清除市场缓存
    │   ├── clearPluginCache()          # 清除插件缓存
    │   └── setAppState()               # 标记 needsRefresh: true
    │       # 提示用户运行 /reload-plugins
    │
    └── 4. 启动后台安装
        └── performBackgroundPluginInstallations(setAppState)
            └── PluginInstallationManager.ts
                └── 后台安装插件
```

### 错误处理

```typescript
try {
  // ... 上述流程 ...
} catch (error) {
  // 即使失败也不阻塞启动
  logForDebugging(`Error initiating background plugin installations: ${error}`)
}
```

---

## 关键代码路径与文件引用

### 导出函数

| 函数 | 用途 | 调用方 |
|------|------|--------|
| `performStartupChecks` | 启动插件初始化 | `screens/REPL.tsx` |

### 调用关系图

```
screens/REPL.tsx
    └── REPL 组件初始化
        └── useEffect 或初始化逻辑
            └── performStartupChecks(setAppState)
                ├── utils/config.ts
                │   └── checkHasTrustDialogAccepted()
                ├── marketplaceManager.ts
                │   ├── registerSeedMarketplaces()
                │   └── clearMarketplacesCache()
                ├── pluginLoader.ts
                │   └── clearPluginCache()
                └── services/plugins/PluginInstallationManager.ts
                    └── performBackgroundPluginInstallations()
```

---

## 依赖与外部交互

### 直接依赖

| 模块 | 用途 |
|------|------|
| `utils/config.ts` | 信任检查 (`checkHasTrustDialogAccepted`) |
| `utils/debug.ts` | 调试日志 (`logForDebugging`) |
| `marketplaceManager.ts` | 市场注册和缓存管理 |
| `pluginLoader.ts` | 插件缓存管理 |
| `services/plugins/PluginInstallationManager.ts` | 后台安装 |
| `state/AppState.ts` | AppState 类型 |

### 类型定义

```typescript
type SetAppState = (f: (prevState: AppState) => AppState) => void
```

---

## 风险、边界与改进建议

### 已知风险

1. **信任检查绕过**
   - 风险：如果其他路径直接调用本函数，可能绕过信任检查
   - 缓解：函数注释明确安全要求，调用方（REPL.tsx）确保顺序

2. **种子市场竞态**
   - 风险：种子市场注册和后台安装可能竞态
   - 缓解：`registerSeedMarketplaces` 是同步/原子操作，完成后才启动后台安装

3. **缓存清除范围**
   - 风险：清除所有缓存可能影响性能
   - 现状：种子市场变更相对罕见，全量清除简单可靠

4. **后台安装失败无通知**
   - 风险：用户可能不知道插件安装失败
   - 缓解：`performBackgroundPluginInstallations` 通过 AppState 报告状态

### 边界情况

| 场景 | 行为 |
|------|------|
| 未信任目录 | 跳过所有插件操作 |
| 无种子目录 | `registerSeedMarketplaces` 返回 false，无缓存操作 |
| 种子目录未变更 | 不清理缓存 |
| 后台安装抛出 | 捕获错误，记录日志，不阻断启动 |
| 重复调用 | 每次调用都执行完整流程（幂等但冗余） |

### 改进建议

1. **调用保护**
   - 建议：添加开发模式断言，确保只在信任后调用

2. **进度指示**
   - 建议：启动时显示插件初始化进度条

3. **延迟启动**
   - 建议：种子市场注册可延迟几秒，优先保证 UI 响应

4. **失败重试**
   - 建议：后台安装失败时自动重试或提示用户

5. **诊断信息**
   - 建议：添加更多调试信息，便于排查启动问题

---

## 测试要点

1. **信任检查**：验证未信任时跳过插件操作
2. **种子市场**：验证注册和缓存清除的正确性
3. **错误处理**：验证后台安装失败不阻断启动
4. **状态更新**：验证 `needsRefresh` 正确标记
5. **幂等性**：验证多次调用的行为一致性
