# AddMarketplace.tsx 深度研究文档

## 1. 场景与职责

### 1.1 组件定位

`AddMarketplace.tsx` 是 Claude Code 插件系统的交互式 UI 组件，负责**添加新的插件市场源（Marketplace Source）**。它是 `/plugin marketplace add` 命令的交互式实现，支持：

- **交互式模式**：在 TUI（Terminal UI）中通过表单输入市场源信息
- **CLI 模式**：支持非交互式命令行调用（`claude plugin marketplace add <source>`）

### 1.2 使用场景

| 场景 | 触发方式 | 行为 |
|------|----------|------|
| 交互式添加 | `/plugin` → "Manage marketplaces" → "Add Marketplace" | 显示输入表单，用户手动输入源地址 |
| CLI 直接添加 | `claude plugin marketplace add <source>` | 自动解析并添加，完成后退出 |
| 自动添加（带初始值） | `PluginSettings` 传入 `initialValue` | 组件挂载后自动尝试添加 |

### 1.3 职责边界

- **不负责**：市场源的持久化存储（由 `marketplaceManager.ts` 处理）
- **不负责**：市场源的验证和缓存（由 `marketplaceManager.ts` 处理）
- **负责**：用户输入的收集、解析、错误展示、进度反馈、结果通知

---

## 2. 功能点目的

### 2.1 核心功能

| 功能 | 目的 |
|------|------|
| 输入解析 | 支持多种市场源格式：GitHub 简写、SSH URL、HTTPS URL、本地路径 |
| 自动添加 | 支持 CLI 模式下的非交互式批量添加 |
| 进度反馈 | 在长时间操作（git clone/pull）期间显示进度消息 |
| 错误处理 | 优雅处理网络错误、权限错误、格式错误 |
| 缓存清理 | 添加成功后清理所有插件缓存，确保新市场立即可用 |

### 2.2 支持的输入格式

```
# GitHub 简写
owner/repo
owner/repo#ref
owner/repo@ref

# SSH URL
git@github.com:owner/repo.git
git@github.com:owner/repo.git#ref
deploy@gitlab.com:group/project.git

# HTTPS URL
https://github.com/owner/repo.git
https://example.com/marketplace.json

# 本地路径
./path/to/marketplace
../relative/path
~/home/dir/marketplace
/path/to/marketplace.json
```

---

## 3. 具体技术实现

### 3.1 组件接口定义

```typescript
// 文件: src/commands/plugin/AddMarketplace.tsx

type Props = {
  inputValue: string;                    // 当前输入值
  setInputValue: (value: string) => void; // 输入值更新回调
  cursorOffset: number;                  // 光标位置
  setCursorOffset: (offset: number) => void;
  error: string | null;                  // 错误消息
  setError: (error: string | null) => void;
  result: string | null;                 // 结果消息
  setResult: (result: string | null) => void;
  setViewState: (state: ViewState) => void;  // 视图状态切换
  onAddComplete?: () => void | Promise<void>; // 添加完成回调
  cliMode?: boolean;                     // 是否为 CLI 模式
};
```

### 3.2 核心处理流程

```
┌─────────────────────────────────────────────────────────────────┐
│                        handleAdd()                               │
│                    (用户按 Enter 触发)                            │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 1. 输入验证                                                     │
│    - 检查非空: input.trim() !== ''                              │
│    - 空输入时: setError('Please enter a marketplace source')    │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. 解析输入                                                     │
│    - 调用: parseMarketplaceInput(input)                         │
│    - 返回: MarketplaceSource | { error: string } | null         │
│    - 失败时: 显示格式错误提示                                    │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. 执行添加                                                     │
│    - 设置 loading 状态                                          │
│    - 调用: addMarketplaceSource(parsed, onProgress)             │
│    - 返回: { name, resolvedSource }                             │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. 保存到设置                                                   │
│    - 调用: saveMarketplaceToSettings(name, { source })          │
│    - 写入: userSettings.extraKnownMarketplaces                  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 5. 清理缓存                                                     │
│    - 调用: clearAllCaches()                                     │
│    - 清理: 插件命令、Agent、Hook、输出样式等缓存                  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 6. 上报分析事件                                                 │
│    - 事件: tengu_marketplace_added                              │
│    - 元数据: source_type (github/url/git/directory/file)        │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 7. 完成处理                                                     │
│    - CLI 模式: setResult('Successfully added...')               │
│    - 交互模式: setViewState({ type: 'browse-marketplace', ... }) │
└─────────────────────────────────────────────────────────────────┘
```

### 3.3 自动添加机制

```typescript
// 组件挂载时检查是否需要自动添加
useEffect(() => {
  if (inputValue && !hasAttemptedAutoAdd.current && !error && !result) {
    hasAttemptedAutoAdd.current = true;
    void handleAdd();
  }
}, []);
```

此机制支持 CLI 模式的自动执行：当 `PluginSettings` 通过 `initialValue` 传入值时，组件会自动尝试添加，无需用户交互。

### 3.4 关键数据结构

#### MarketplaceSource（市场源类型）

```typescript
// 文件: src/utils/plugins/schemas.ts

type MarketplaceSource =
  | { source: 'url'; url: string; headers?: Record<string, string> }
  | { source: 'github'; repo: string; ref?: string; path?: string; sparsePaths?: string[] }
  | { source: 'git'; url: string; ref?: string; path?: string; sparsePaths?: string[] }
  | { source: 'npm'; package: string }
  | { source: 'file'; path: string }
  | { source: 'directory'; path: string }
  | { source: 'hostPattern'; hostPattern: string }
  | { source: 'pathPattern'; pathPattern: string }
  | { source: 'settings'; name: string; plugins: SettingsMarketplacePlugin[]; owner?: PluginAuthor };
```

#### ViewState（视图状态）

```typescript
// 文件: src/commands/plugin/types.js (编译后)
// 原始: src/commands/plugin/types.ts

type ViewState =
  | { type: 'menu' }
  | { type: 'help' }
  | { type: 'discover-plugins'; targetPlugin?: string }
  | { type: 'browse-marketplace'; targetMarketplace: string; targetPlugin?: string }
  | { type: 'manage-plugins'; targetPlugin?: string; targetMarketplace?: string; action?: 'uninstall' | 'enable' | 'disable' }
  | { type: 'manage-marketplaces'; targetMarketplace?: string; action?: 'remove' | 'update' }
  | { type: 'add-marketplace'; initialValue?: string }
  | { type: 'marketplace-list' }
  | { type: 'marketplace-menu' }
  | { type: 'validate'; path: string };
```

---

## 4. 关键代码路径与文件引用

### 4.1 文件依赖图

```
AddMarketplace.tsx
├── 直接依赖
│   ├── react (useEffect, useRef, useState)
│   ├── src/services/analytics/index.js (logEvent)
│   ├── src/components/ConfigurableShortcutHint.js
│   ├── src/components/design-system/Byline.js
│   ├── src/components/design-system/KeyboardShortcutHint.js
│   ├── src/components/Spinner.js
│   ├── src/components/TextInput.js
│   ├── src/ink.js (Box, Text)
│   ├── src/utils/errors.js (toError)
│   ├── src/utils/log.js (logError)
│   ├── src/utils/plugins/cacheUtils.js (clearAllCaches)
│   ├── src/utils/plugins/marketplaceManager.js (addMarketplaceSource, saveMarketplaceToSettings)
│   ├── src/utils/plugins/parseMarketplaceInput.js (parseMarketplaceInput)
│   └── ./types.js (ViewState)
│
└── 间接依赖（核心实现）
    ├── src/utils/plugins/marketplaceManager.ts
    │   ├── loadAndCacheMarketplace()      // 加载并缓存市场
    │   ├── cacheMarketplaceFromGit()      // Git 克隆/拉取
    │   ├── cacheMarketplaceFromUrl()      // URL 下载
    │   ├── gitClone() / gitPull()         // Git 操作
    │   └── saveKnownMarketplacesConfig()  // 保存配置
    │
    ├── src/utils/plugins/parseMarketplaceInput.ts
    │   └── parseMarketplaceInput()        // 输入解析
    │
    ├── src/utils/plugins/schemas.ts
    │   ├── MarketplaceSourceSchema        // 源验证
    │   ├── PluginMarketplaceSchema        // 市场验证
    │   └── validateOfficialNameSource()   // 官方名称验证
    │
    └── src/utils/plugins/cacheUtils.ts
        └── clearAllCaches()               // 缓存清理
```

### 4.2 关键代码路径

#### 路径 1: 输入解析

```
AddMarketplace.tsx:51
  └── parseMarketplaceInput(input)
      └── src/utils/plugins/parseMarketplaceInput.ts:23
          ├── 正则匹配 SSH URL: /^([a-zA-Z0-9._-]+@[^:]+:.+?(?:\.git)?)(#(.+))?$/
          ├── 检测 HTTP(S) URL
          ├── 检测本地路径 (./, ../, /, ~)
          └── 检测 GitHub 简写 (owner/repo)
```

#### 路径 2: 市场添加

```
AddMarketplace.tsx:69
  └── addMarketplaceSource(parsed, onProgress)
      └── src/utils/plugins/marketplaceManager.ts:1782
          ├── 策略检查: isSourceAllowedByPolicy()
          ├── 源去重检查: 检查 known_marketplaces.json
          ├── loadAndCacheMarketplace()
          │   ├── cacheMarketplaceFromGit()  // git 源
          │   │   ├── gitClone() / gitPull()
          │   │   └── 验证 marketplace.json
          │   └── cacheMarketplaceFromUrl()  // url 源
          │       ├── axios.get()
          │       └── 验证 marketplace.json
          ├── 官方名称验证: validateOfficialNameSource()
          ├── 冲突处理: 检查同名不同源的市场
          └── saveKnownMarketplacesConfig()
```

#### 路径 3: 缓存清理

```
AddMarketplace.tsx:75
  └── clearAllCaches()
      └── src/utils/plugins/cacheUtils.ts:44
          ├── clearAllPluginCaches()
          │   ├── clearPluginCache()
          │   ├── clearPluginCommandCache()
          │   ├── clearPluginAgentCache()
          │   ├── clearPluginHookCache()
          │   ├── pruneRemovedPluginHooks()
          │   └── clearPluginOutputStyleCache()
          ├── clearCommandsCache()
          ├── clearAgentDefinitionsCache()
          └── clearPromptCache()
```

---

## 5. 依赖与外部交互

### 5.1 核心依赖模块

| 模块 | 路径 | 用途 |
|------|------|------|
| marketplaceManager | `src/utils/plugins/marketplaceManager.ts` | 市场源的添加、加载、缓存、配置管理 |
| parseMarketplaceInput | `src/utils/plugins/parseMarketplaceInput.ts` | 用户输入解析为结构化源对象 |
| cacheUtils | `src/utils/plugins/cacheUtils.ts` | 缓存清理 |
| schemas | `src/utils/plugins/schemas.ts` | 类型定义和验证 |
| analytics | `src/services/analytics/index.ts` | 事件上报 |

### 5.2 外部系统交互

```
┌─────────────────────────────────────────────────────────────────┐
│                     AddMarketplace.tsx                          │
└─────────────────────────────────────────────────────────────────┘
                              │
          ┌───────────────────┼───────────────────┐
          ▼                   ▼                   ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│   Git 操作      │  │   HTTP 请求     │  │   文件系统      │
│                 │  │                 │  │                 │
│ • git clone     │  │ • axios.get()   │  │ • 读取配置      │
│ • git pull      │  │ • 下载市场 JSON │  │ • 写入配置      │
│ • sparse-checkout│ │ • 自定义 Headers│  │ • 缓存管理      │
└─────────────────┘  └─────────────────┘  └─────────────────┘
          │                   │                   │
          ▼                   ▼                   ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│  GitHub/自定义   │  │  市场托管服务器  │  │  ~/.claude/     │
│  Git 仓库       │  │                 │  │  plugins/       │
└─────────────────┘  └─────────────────┘  └─────────────────┘
```

### 5.3 配置文件

| 文件 | 路径（相对用户主目录） | 用途 |
|------|------------------------|------|
| known_marketplaces.json | `~/.claude/plugins/known_marketplaces.json` | 存储已知市场的配置 |
| 市场缓存 | `~/.claude/plugins/marketplaces/<name>/` | Git 源的克隆目录 |
| 市场缓存(JSON) | `~/.claude/plugins/marketplaces/<name>.json` | URL 源的缓存文件 |
| 用户设置 | `~/.claude/settings.json` | extraKnownMarketplaces 存储 |

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 风险 1: Git 操作超时

| 项目 | 描述 |
|------|------|
| 风险 | 大型仓库克隆可能超时（默认 120s） |
| 代码 | `marketplaceManager.ts:515-526` |
| 缓解 | 支持 `CLAUDE_CODE_PLUGIN_GIT_TIMEOUT_MS` 环境变量自定义超时 |
| 改进 | 添加进度条显示克隆进度 |

#### 风险 2: 策略冲突

| 项目 | 描述 |
|------|------|
| 风险 | 企业策略（strictKnownMarketplaces/blockedMarketplaces）可能阻止合法添加 |
| 代码 | `marketplaceManager.ts:1796-1831` |
| 缓解 | 清晰的错误消息提示用户联系管理员 |
| 改进 | 添加 `--force` 选项供管理员绕过策略（需认证） |

#### 风险 3: 名称冲突

| 项目 | 描述 |
|------|------|
| 风险 | 同名不同源的市场可能意外覆盖 |
| 代码 | `marketplaceManager.ts:1859-1911` |
| 缓解 | 检查 seed-managed 条目，防止覆盖管理员配置 |
| 改进 | 添加交互式确认对话框 |

#### 风险 4: 路径遍历攻击

| 项目 | 描述 |
|------|------|
| 风险 | 恶意 marketplace.json 可能通过 crafted name 导致路径遍历 |
| 代码 | `marketplaceManager.ts:1714-1719` |
| 缓解 | 验证最终路径是否在缓存目录内 |
| 改进 | 定期安全审计路径处理逻辑 |

### 6.2 边界情况

| 场景 | 行为 |
|------|------|
| 空输入 | 显示错误："Please enter a marketplace source" |
| 无效格式 | 显示错误："Invalid marketplace source format..." |
| 网络超时 | 显示具体超时错误，建议增加超时时间 |
| 重复添加 | 通过 `alreadyMaterialized` 标记，显示友好提示 |
| 策略阻止 | 显示策略限制错误，列出允许的来源 |
| Seed-managed | 阻止修改，提示联系管理员 |

### 6.3 改进建议

#### 建议 1: 增强输入验证

```typescript
// 当前：简单的非空检查
if (!input) {
  setError('Please enter a marketplace source');
  return;
}

// 建议：添加更多预检
if (input.length > 500) {
  setError('Input too long (max 500 characters)');
  return;
}
if (input.includes('\n') || input.includes('\r')) {
  setError('Input cannot contain newlines');
  return;
}
```

#### 建议 2: 添加历史记录

```typescript
// 保存最近添加的市场源，方便快速重新添加
const [recentSources, setRecentSources] = useState<string[]>([]);

// 在 UI 中显示最近使用
<Box>
  <Text dimColor>Recent:</Text>
  {recentSources.map(s => (
    <Button key={s} onPress={() => setInputValue(s)}>{s}</Button>
  ))}
</Box>
```

#### 建议 3: 改进错误恢复

```typescript
// 当前：错误后需要重新输入
// 建议：保留输入，允许用户编辑后重试
const handleAdd = async () => {
  // ... 解析和验证 ...
  
  try {
    // ... 添加逻辑 ...
  } catch (err) {
    // 不清空 inputValue，允许用户修改后重试
    setError(error.message);
    setLoading(false);
  }
};
```

#### 建议 4: 添加测试覆盖

```typescript
// 建议添加的测试用例
describe('AddMarketplace', () => {
  it('should auto-add on mount when inputValue is provided', () => {});
  it('should handle SSH URL format', () => {});
  it('should handle GitHub shorthand format', () => {});
  it('should handle local path format', () => {});
  it('should display progress messages during clone', () => {});
  it('should handle policy block gracefully', () => {});
  it('should handle network timeout', () => {});
  it('should clear caches after successful add', () => {});
  it('should log analytics event on success', () => {});
});
```

### 6.4 性能考虑

| 方面 | 现状 | 建议 |
|------|------|------|
| Git 克隆 | 全量克隆，支持 sparse checkout | 默认启用 sparse checkout 减少下载 |
| 缓存清理 | 全量清理所有缓存 | 细粒度清理，仅清理受影响的缓存 |
| 并发处理 | 单市场顺序处理 | 支持批量添加时的并发处理 |

---

## 附录：相关文件清单

### 核心文件

| 文件 | 行数 | 描述 |
|------|------|------|
| `src/commands/plugin/AddMarketplace.tsx` | 162 | 本组件 |
| `src/commands/plugin/PluginSettings.tsx` | 1000+ | 父组件，管理插件设置整体流程 |
| `src/commands/plugin/ManageMarketplaces.tsx` | 838 | 市场管理组件 |
| `src/commands/plugin/parseArgs.ts` | 103 | 命令行参数解析 |

### 依赖文件

| 文件 | 描述 |
|------|------|
| `src/utils/plugins/marketplaceManager.ts` | 市场管理核心逻辑（2000+ 行） |
| `src/utils/plugins/parseMarketplaceInput.ts` | 输入解析（162 行） |
| `src/utils/plugins/cacheUtils.ts` | 缓存管理（196 行） |
| `src/utils/plugins/schemas.ts` | 类型和验证 Schema（1000+ 行） |
| `src/utils/plugins/marketplaceHelpers.ts` | 市场辅助函数（592 行） |
| `src/services/analytics/index.ts` | 分析服务（173 行） |
| `src/cli/handlers/plugins.ts` | CLI 命令处理（878 行） |

### 类型定义

| 文件 | 描述 |
|------|------|
| `src/commands/plugin/types.js` | 编译后的类型定义（ViewState 等） |
| `src/utils/plugins/schemas.ts` | MarketplaceSource、KnownMarketplace 等 |
| `src/types/plugin.ts` | Plugin 相关类型 |
