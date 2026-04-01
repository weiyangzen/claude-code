# useLspPluginRecommendation.tsx 深度研究文档

## 场景与职责

`useLspPluginRecommendation` 是一个智能的 LSP（Language Server Protocol）插件推荐钩子。它通过检测用户编辑的文件类型，自动推荐已安装市场中的 LSP 插件，前提是系统已安装对应的 LSP 二进制文件。

### 核心场景

1. **文件类型检测**：监听用户编辑的文件历史
2. **LSP 插件匹配**：根据文件扩展名匹配合适的 LSP 插件
3. **二进制可用性检查**：确保推荐的 LSP 服务器可在系统上运行
4. **智能推荐**：每会话只显示一次推荐，避免打扰

### 功能限制

- **仅检测内联配置**：无法检测引用外部 `.lsp.json` 文件的插件
- **每会话一次**：避免重复推荐
- **官方市场优先**：排序时优先显示官方市场的插件

---

## 功能点目的

### 1. 文件编辑监听

- 通过 `useAppState` 监听 `fileHistory.trackedFiles`
- 使用 `checkedFilesRef` 跟踪已检查的文件，避免重复处理
- 只处理新文件，提高性能

### 2. LSP 插件匹配

匹配条件：
1. 文件扩展名匹配插件声明的扩展名
2. LSP 二进制文件已安装在系统上
3. 插件尚未安装
4. 插件不在用户的"永不推荐"列表中

### 3. 用户响应处理

支持四种用户响应：

| 响应 | 行为 |
|-----|------|
| `yes` | 安装插件并启用 |
| `no` | 关闭推荐，超时则增加忽略计数 |
| `never` | 将插件加入永不推荐列表 |
| `disable` | 全局禁用 LSP 推荐功能 |

### 4. 超时检测

- 菜单在 30 秒后自动关闭
- 如果用户未在 28 秒内响应，视为超时
- 超时 5 次后自动禁用推荐

---

## 具体技术实现

### 关键数据结构

```typescript
export type LspRecommendationState = {
  pluginId: string           // 插件 ID，如 "typescript@anthropic"
  pluginName: string         // 显示名称
  pluginDescription?: string // 描述
  fileExtension: string      // 匹配的文件扩展名
  shownAt: number           // 显示时间戳（用于超时检测）
} | null

// 超时阈值
const TIMEOUT_THRESHOLD_MS = 28_000  // 28 秒
```

### 核心流程

#### 1. 插件检测 Effect

```typescript
React.useEffect(() => {
  tryResolve(async () => {
    // 检查是否已显示过
    if (hasShownLspRecommendationThisSession()) return null
    
    // 收集新文件
    const newFiles = []
    for (const file of trackedFiles) {
      if (!checkedFilesRef.current.has(file)) {
        checkedFilesRef.current.add(file)
        newFiles.push(file)
      }
    }
    
    // 检查每个新文件
    for (const filePath of newFiles) {
      const matches = await getMatchingLspPlugins(filePath)
      const match = matches[0]
      if (match) {
        setLspRecommendationShownThisSession(true)
        return {
          pluginId: match.pluginId,
          pluginName: match.pluginName,
          pluginDescription: match.description,
          fileExtension: extname(filePath),
          shownAt: Date.now()
        }
      }
    }
    return null
  })
}, [trackedFiles, tryResolve])
```

#### 2. 用户响应处理

```typescript
const handleResponse = (response: 'yes' | 'no' | 'never' | 'disable') => {
  if (!recommendation) return
  
  switch (response) {
    case 'yes':
      installPluginAndNotify(pluginId, pluginName, 'lsp-plugin', addNotification, 
        async pluginData => {
          const localSourcePath = typeof pluginData.entry.source === 'string' 
            ? join(pluginData.marketplaceInstallLocation, pluginData.entry.source) 
            : undefined
          await cacheAndRegisterPlugin(pluginId, pluginData.entry, 'user', undefined, localSourcePath)
          
          // 启用插件
          const settings = getSettingsForSource('userSettings')
          updateSettingsForSource('userSettings', {
            enabledPlugins: { ...settings?.enabledPlugins, [pluginId]: true }
          })
        })
      break
      
    case 'no':
      const elapsed = Date.now() - shownAt
      if (elapsed >= TIMEOUT_THRESHOLD_MS) {
        incrementIgnoredCount()
      }
      break
      
    case 'never':
      addToNeverSuggest(pluginId)
      break
      
    case 'disable':
      saveGlobalConfig(current => ({ ...current, lspRecommendationDisabled: true }))
      break
  }
  
  clearRecommendation()
}
```

### 依赖函数

#### `getMatchingLspPlugins`

位于 `src/utils/plugins/lspRecommendation.ts`：

```typescript
export async function getMatchingLspPlugins(filePath: string): Promise<LspPluginRecommendation[]> {
  // 检查是否全局禁用
  if (isLspRecommendationsDisabled()) return []
  
  const ext = extname(filePath).toLowerCase()
  if (!ext) return []
  
  // 获取所有 LSP 插件
  const allLspPlugins = await getLspPluginsFromMarketplaces()
  const config = getGlobalConfig()
  const neverPlugins = config.lspRecommendationNeverPlugins ?? []
  
  // 过滤匹配插件
  const matchingPlugins = []
  for (const [pluginId, info] of allLspPlugins) {
    if (!info.extensions.has(ext)) continue
    if (neverPlugins.includes(pluginId)) continue
    if (isPluginInstalled(pluginId)) continue
    matchingPlugins.push({ info, pluginId })
  }
  
  // 检查二进制可用性
  const pluginsWithBinary = []
  for (const { info, pluginId } of matchingPlugins) {
    if (await isBinaryInstalled(info.command)) {
      pluginsWithBinary.push({ info, pluginId })
    }
  }
  
  // 排序：官方市场优先
  pluginsWithBinary.sort((a, b) => {
    if (a.info.isOfficial && !b.info.isOfficial) return -1
    if (!a.info.isOfficial && b.info.isOfficial) return 1
    return 0
  })
  
  return pluginsWithBinary.map(...)
}
```

---

## 关键代码路径与文件引用

```
src/hooks/useLspPluginRecommendation.tsx
├── LspRecommendationState 类型    # 行 30-36
├── TIMEOUT_THRESHOLD_MS           # 行 29: 28 秒超时阈值
├── useLspPluginRecommendation()   # 行 41-181: 主钩子
│   ├── trackedFiles 监听          # 行 43
│   ├── checkedFilesRef            # 行 54
│   ├── usePluginRecommendationBase # 行 59
│   ├── useEffect - 检测逻辑       # 行 62-108
│   └── handleResponse             # 行 111-159
└── 辅助函数                       # 行 182-193

src/utils/plugins/lspRecommendation.ts
├── getMatchingLspPlugins()        # 行 222-309: 核心匹配逻辑
├── getLspPluginsFromMarketplaces() # 行 160-206
├── extractLspInfoFromManifest()   # 行 67-100
├── isLspRecommendationsDisabled() # 行 351-357
├── addToNeverSuggest()            # 行 316-328
└── incrementIgnoredCount()        # 行 334-343

src/hooks/usePluginRecommendationBase.tsx
├── usePluginRecommendationBase()  # 基础状态管理
└── installPluginAndNotify()       # 安装辅助函数
```

### 依赖文件

```
src/bootstrap/state.js
├── hasShownLspRecommendationThisSession()
└── setLspRecommendationShownThisSession()

src/utils/plugins/pluginInstallationHelpers.ts
└── cacheAndRegisterPlugin()

src/utils/plugins/installedPluginsManager.ts
└── isPluginInstalled()

src/utils/plugins/marketplaceManager.ts
├── getMarketplace()
└── loadKnownMarketplacesConfig()

src/utils/binaryCheck.ts
└── isBinaryInstalled()

src/utils/config.ts
├── getGlobalConfig()
└── saveGlobalConfig()

src/utils/settings/settings.ts
├── getSettingsForSource()
└── updateSettingsForSource()
```

---

## 依赖与外部交互

### React Hooks 使用

- `useAppState`: 获取 trackedFiles
- `useNotifications`: 添加通知
- `useRef`: 跟踪已检查文件
- `usePluginRecommendationBase`: 共享推荐逻辑

### 与插件系统的交互

```typescript
// 安装流程
await cacheAndRegisterPlugin(pluginId, pluginData.entry, 'user', undefined, localSourcePath)

// 启用插件
updateSettingsForSource('userSettings', {
  enabledPlugins: { ...settings?.enabledPlugins, [pluginId]: true }
})
```

### 与配置系统的交互

```typescript
// 全局配置
saveGlobalConfig(current => ({
  ...current,
  lspRecommendationDisabled: true
}))

// 永不推荐列表
addToNeverSuggest(pluginId)  // 内部调用 saveGlobalConfig

// 忽略计数
incrementIgnoredCount()  // 内部调用 saveGlobalConfig
```

---

## 风险、边界与改进建议

### 已知风险

1. **无法检测外部 LSP 配置**
   - 插件使用字符串路径引用 `.lsp.json` 时无法检测
   - 缓解：文档说明限制，鼓励内联配置

2. **二进制检查开销**
   - 每个匹配插件都要检查二进制是否存在
   - 缓解：可以添加缓存机制

3. **误推荐**
   - 用户可能不想为某些文件类型安装 LSP
   - 缓解：提供 "never" 和 "disable" 选项

### 边界情况

| 场景 | 行为 |
|-----|------|
| 无文件扩展名 | 跳过检测 |
| 二进制不存在 | 不显示推荐 |
| 已安装插件 | 跳过 |
| 在永不列表中 | 跳过 |
| 已显示过 | 不再显示 |
| 超时响应 | 增加忽略计数 |

### 改进建议

1. **缓存二进制检查结果**
   - 避免重复检查相同的二进制
   - 添加 TTL 机制

2. **更智能的推荐时机**
   - 基于文件打开频率推荐
   - 只在用户多次编辑某类型文件后推荐

3. **批量推荐**
   - 一次显示多个匹配的插件
   - 允许用户选择性安装

4. **LSP 健康检查**
   - 安装后验证 LSP 是否正常工作
   - 提供故障排除指引

5. **遥测**
   - 记录推荐接受率
   - 分析用户偏好

### 测试建议

1. **单元测试**：
   - 各种文件扩展名匹配
   - 二进制存在性检查
   - 用户响应处理

2. **集成测试**：
   - 完整推荐流程
   - 安装流程

3. **边界测试**：
   - 超时场景
   - 多次快速编辑
