# useOfficialMarketplaceNotification.tsx 深度研究文档

## 场景与职责

`useOfficialMarketplaceNotification` 是一个用于处理官方市场（Official Marketplace）自动安装通知的 React 钩子。它在启动时检查官方市场的安装状态，并向用户显示相应的成功或失败通知。

### 核心场景

1. **自动安装检查**：启动时检查是否需要安装官方市场
2. **成功通知**：安装成功时显示提示
3. **失败处理**：安装失败时显示重试提示
4. **配置保存失败**：特殊处理配置保存失败的情况

### 与其他组件的关系

- 使用 `useStartupNotification` 基础钩子
- 依赖 `checkAndInstallOfficialMarketplace` 执行实际安装
- 与通知系统集成显示结果

---

## 功能点目的

### 1. 官方市场自动安装

检查并安装 Anthropic 官方插件市场：
- 首次启动时自动尝试安装
- 支持 GCS 镜像和 Git 回退
- 企业策略限制检查

### 2. 通知场景

| 场景 | 通知内容 | 优先级 |
|-----|---------|-------|
| 配置保存失败 | "Failed to save marketplace retry info · Check ~/.claude.json permissions" | immediate |
| 安装成功 | "✓ Anthropic marketplace installed · /plugin to see available plugins" | immediate |
| 安装失败 | "Failed to install Anthropic marketplace · Will retry on next startup" | immediate |

### 3. 错误处理

- 配置保存失败单独显示（权限问题）
- 其他失败显示通用重试提示
- 成功时显示可用插件提示

---

## 具体技术实现

### 关键数据结构

```typescript
// checkAndInstallOfficialMarketplace 返回结果
type OfficialMarketplaceCheckResult = {
  installed: boolean      // 是否成功安装
  skipped: boolean        // 是否跳过
  reason?: OfficialMarketplaceSkipReason  // 跳过原因
  configSaveFailed?: boolean  // 配置保存是否失败
}

type OfficialMarketplaceSkipReason =
  | 'already_attempted'
  | 'already_installed'
  | 'policy_blocked'
  | 'git_unavailable'
  | 'gcs_unavailable'
  | 'unknown'
```

### 核心实现

```typescript
export function useOfficialMarketplaceNotification() {
  useStartupNotification(async () => {
    const result = await checkAndInstallOfficialMarketplace()
    const notifs: Notification[] = []
    
    // 1. 配置保存失败（最高优先级）
    if (result.configSaveFailed) {
      logForDebugging('Showing marketplace config save failure notification')
      notifs.push({
        key: 'marketplace-config-save-failed',
        jsx: <Text color="error">Failed to save marketplace retry info · Check ~/.claude.json permissions</Text>,
        priority: 'immediate',
        timeoutMs: 10000
      })
    }
    
    // 2. 安装成功
    if (result.installed) {
      logForDebugging('Showing marketplace installation success notification')
      notifs.push({
        key: 'marketplace-installed',
        jsx: <Text color="success">✓ Anthropic marketplace installed · /plugin to see available plugins</Text>,
        priority: 'immediate',
        timeoutMs: 7000
      })
    } else {
      // 3. 安装失败（未知原因）
      if (result.skipped && result.reason === 'unknown') {
        logForDebugging('Showing marketplace installation failure notification')
        notifs.push({
          key: 'marketplace-install-failed',
          jsx: <Text color="warning">Failed to install Anthropic marketplace · Will retry on next startup</Text>,
          priority: 'immediate',
          timeoutMs: 8000
        })
      }
    }
    
    return notifs
  })
}
```

### 安装检查流程

```typescript
// checkAndInstallOfficialMarketplace 内部逻辑
async function checkAndInstallOfficialMarketplace(): Promise<OfficialMarketplaceCheckResult> {
  // 1. 检查是否应该重试
  if (!shouldRetryInstallation(config)) {
    return { installed: false, skipped: true, reason: ... }
  }
  
  // 2. 检查环境变量禁用
  if (isOfficialMarketplaceAutoInstallDisabled()) {
    return { installed: false, skipped: true, reason: 'policy_blocked' }
  }
  
  // 3. 检查是否已安装
  if (knownMarketplaces[OFFICIAL_MARKETPLACE_NAME]) {
    return { installed: false, skipped: true, reason: 'already_installed' }
  }
  
  // 4. 检查企业策略
  if (!isSourceAllowedByPolicy(OFFICIAL_MARKETPLACE_SOURCE)) {
    return { installed: false, skipped: true, reason: 'policy_blocked' }
  }
  
  // 5. 尝试 GCS 安装
  const gcsSha = await fetchOfficialMarketplaceFromGcs(installLocation, cacheDir)
  if (gcsSha !== null) {
    return { installed: true, skipped: false }
  }
  
  // 6. Git 回退（如果启用）
  if (gitAvailable) {
    await addMarketplaceSource(OFFICIAL_MARKETPLACE_SOURCE)
    return { installed: true, skipped: false }
  }
  
  // 7. 失败处理
  return { installed: false, skipped: true, reason: 'unknown' }
}
```

---

## 关键代码路径与文件引用

```
src/hooks/useOfficialMarketplaceNotification.tsx
├── useOfficialMarketplaceNotification()  # 行 12-14: 主钩子
└── _temp() 辅助函数                   # 行 15-47: 通知逻辑
    ├── checkAndInstallOfficialMarketplace()  # 行 16
    ├── configSaveFailed 处理           # 行 18-26
    ├── installed 成功处理              # 行 27-35
    └── skipped + unknown 失败处理       # 行 36-45

src/hooks/notifs/useStartupNotification.ts
└── useStartupNotification()         # 基础启动通知钩子

src/utils/plugins/officialMarketplaceStartupCheck.ts
├── checkAndInstallOfficialMarketplace()  # 安装检查逻辑
├── shouldRetryInstallation()        # 重试判断
├── RETRY_CONFIG                     # 重试配置
└── OfficialMarketplaceCheckResult   # 结果类型

src/utils/plugins/officialMarketplaceGcs.ts
└── fetchOfficialMarketplaceFromGcs()  # GCS 下载

src/utils/plugins/marketplaceManager.ts
├── addMarketplaceSource()           # 添加市场源
├── loadKnownMarketplacesConfig()    # 加载已知市场
└── saveKnownMarketplacesConfig()    # 保存市场配置
```

---

## 依赖与外部交互

### React Hooks 使用

- `useStartupNotification`: 基础启动通知钩子

### 依赖函数

```typescript
import { checkAndInstallOfficialMarketplace } from '../utils/plugins/officialMarketplaceStartupCheck.js'
import { useStartupNotification } from './notifs/useStartupNotification.js'
import { logForDebugging } from '../utils/debug.js'
```

### 通知系统

```typescript
// 成功通知
addNotification({
  key: 'marketplace-installed',
  jsx: <Text color="success">...</Text>,
  priority: 'immediate',
  timeoutMs: 7000
})

// 失败通知
addNotification({
  key: 'marketplace-install-failed',
  jsx: <Text color="warning">...</Text>,
  priority: 'immediate',
  timeoutMs: 8000
})
```

---

## 风险、边界与改进建议

### 已知风险

1. **网络依赖**
   - 需要访问 GCS 或 GitHub
   - 企业防火墙可能阻止

2. **权限问题**
   - 配置保存可能因权限失败
   - 需要检查 `~/.claude.json` 权限

3. **重试风暴**
   - 每次启动都尝试，可能产生大量请求
   - 缓解：指数退避重试机制

### 边界情况

| 场景 | 行为 |
|-----|------|
| 已安装 | 跳过，显示已安装 |
| 企业策略阻止 | 跳过，记录原因 |
| Git 不可用 | 使用 GCS，失败则重试 |
| GCS 不可用 | 如果启用 Git 回退则尝试 Git |
| 配置保存失败 | 单独显示权限错误 |
| 达到最大重试次数 | 停止尝试 |

### 改进建议

1. **离线模式**
   - 检测无网络时跳过安装
   - 提供手动安装指引

2. **代理支持**
   - 支持 HTTP_PROXY 等环境变量
   - 企业代理场景

3. **进度显示**
   - 显示下载进度
   - 大文件下载时用户体验更好

4. **镜像选择**
   - 允许用户配置镜像源
   - 支持私有镜像

5. **版本锁定**
   - 支持锁定特定版本
   - 避免自动更新带来的问题

### 测试建议

1. **单元测试**：
   - 各种跳过条件
   - 重试逻辑
   - 通知生成

2. **集成测试**：
   - 完整安装流程
   - 失败恢复流程

3. **网络测试**：
   - 无网络场景
   - 慢网络场景
   - 代理场景
