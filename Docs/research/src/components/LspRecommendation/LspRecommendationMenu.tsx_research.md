# LspRecommendationMenu.tsx 深度研究文档

## 1. 场景与职责

### 1.1 功能定位
`LspRecommendationMenu.tsx` 是 Claude Code CLI 中用于**LSP (Language Server Protocol) 插件推荐**的交互式 UI 组件。当用户编辑特定类型的文件时，系统检测到对应的 LSP 插件可用且未安装时，会弹出此菜单询问用户是否安装该插件。

### 1.2 核心职责
- **展示推荐信息**: 显示插件名称、描述、触发该推荐的文件扩展名
- **用户交互**: 提供四种响应选项（是/否/永不/禁用所有）
- **自动关闭**: 30秒无操作自动关闭并视为"否"
- **教育提示**: 向用户解释 LSP 提供的功能（代码智能提示、错误检查等）

### 1.3 触发场景
1. 用户编辑文件时，文件历史追踪系统记录被修改的文件
2. `useLspPluginRecommendation` hook 检测到新文件
3. 匹配文件扩展名与 Marketplace 中声明了 `lspServers` 的插件
4. 检查 LSP 二进制文件是否已安装在系统中
5. 检查插件是否已安装、是否在"永不推荐"列表中
6. 满足所有条件后，通过 REPL 渲染此菜单组件

---

## 2. 功能点目的

### 2.1 用户选项设计

| 选项 | Value | 行为 |
|------|-------|------|
| Yes, install {pluginName} | `yes` | 安装插件并启用，发送通知提示重启生效 |
| No, not now | `no` | 关闭推荐，超时（>28秒）会递增忽略计数 |
| Never for {pluginName} | `never` | 将该插件加入"永不推荐"列表 |
| Disable all LSP recommendations | `disable` | 全局禁用所有 LSP 推荐功能 |

### 2.2 防打扰机制
- **每会话仅一次**: 通过 `lspRecommendationShownThisSession` 状态控制
- **忽略计数上限**: 累计忽略5次后自动禁用推荐（`MAX_IGNORED_COUNT = 5`）
- **自动关闭**: 30秒无操作自动关闭，避免阻塞用户工作流

### 2.3 安全与隐私
- 仅在 LSP 二进制已存在系统时才推荐（不自动安装二进制）
- 官方 Marketplace 插件优先排序
- 支持用户永久禁用或针对特定插件禁用

---

## 3. 具体技术实现

### 3.1 组件接口定义

```typescript
type Props = {
  pluginName: string;           // 插件名称（显示用）
  pluginDescription?: string;   // 插件描述（可选）
  fileExtension: string;        // 触发推荐的文件扩展名
  onResponse: (response: 'yes' | 'no' | 'never' | 'disable') => void;
};
```

### 3.2 核心实现逻辑

#### 3.2.1 自动关闭定时器
```typescript
const AUTO_DISMISS_MS = 30_000;

// 使用 ref 避免 onResponse 变化时重置定时器
const onResponseRef = React.useRef(onResponse);
onResponseRef.current = onResponse;

React.useEffect(() => {
  const timeoutId = setTimeout(ref => ref.current('no'), AUTO_DISMISS_MS, onResponseRef);
  return () => clearTimeout(timeoutId);
}, []);
```

#### 3.2.2 选项配置
```typescript
const options = [
  {
    label: <Text>Yes, install <Text bold>{pluginName}</Text></Text>,
    value: 'yes'
  },
  {
    label: 'No, not now',
    value: 'no'
  },
  {
    label: <Text>Never for <Text bold>{pluginName}</Text></Text>,
    value: 'never'
  },
  {
    label: 'Disable all LSP recommendations',
    value: 'disable'
  }
];
```

#### 3.2.3 渲染结构
```tsx
<PermissionDialog title="LSP Plugin Recommendation">
  <Box flexDirection="column" paddingX={2} paddingY={1}>
    {/* 教育文本 */}
    <Box marginBottom={1}>
      <Text dimColor>LSP provides code intelligence...</Text>
    </Box>
    {/* 插件信息 */}
    <Box><Text dimColor>Plugin:</Text><Text> {pluginName}</Text></Box>
    {pluginDescription && <Box><Text dimColor>{pluginDescription}</Text></Box>}
    <Box><Text dimColor>Triggered by:</Text><Text> {fileExtension} files</Text></Box>
    {/* 询问文本 */}
    <Box marginTop={1}><Text>Would you like to install this LSP plugin?</Text></Box>
    {/* 选择器 */}
    <Box>
      <Select options={options} onChange={onSelect} onCancel={() => onResponse('no')} />
    </Box>
  </Box>
</PermissionDialog>
```

### 3.3 数据结构

#### 3.3.1 LSP 插件推荐数据结构
```typescript
// src/utils/plugins/lspRecommendation.ts
export type LspPluginRecommendation = {
  pluginId: string;           // "plugin-name@marketplace-name"
  pluginName: string;         // 人类可读的插件名
  marketplaceName: string;    // Marketplace 名称
  description?: string;       // 插件描述
  isOfficial: boolean;        // 是否来自官方 Marketplace
  extensions: string[];       // 支持的文件扩展名
  command: string;            // LSP 服务器命令（如 "typescript-language-server"）
};
```

#### 3.3.2 推荐状态
```typescript
// src/hooks/useLspPluginRecommendation.tsx
export type LspRecommendationState = {
  pluginId: string;
  pluginName: string;
  pluginDescription?: string;
  fileExtension: string;
  shownAt: number;  // 时间戳，用于超时检测
} | null;
```

### 3.4 配置持久化

全局配置中存储的 LSP 推荐相关字段（`src/utils/config.ts`）：

```typescript
export type GlobalConfig = {
  // ... 其他字段
  lspRecommendationDisabled?: boolean;       // 禁用所有推荐
  lspRecommendationNeverPlugins?: string[];  // 永不推荐的插件ID列表
  lspRecommendationIgnoredCount?: number;    // 忽略次数计数
};
```

---

## 4. 关键代码路径与文件引用

### 4.1 调用链

```
REPL.tsx
├── useLspPluginRecommendation()          // src/hooks/useLspPluginRecommendation.tsx
│   ├── usePluginRecommendationBase()     // src/hooks/usePluginRecommendationBase.tsx
│   ├── getMatchingLspPlugins()           // src/utils/plugins/lspRecommendation.ts
│   │   ├── getLspPluginsFromMarketplaces()
│   │   ├── isBinaryInstalled()           // src/utils/binaryCheck.ts
│   │   └── isPluginInstalled()           // src/utils/plugins/installedPluginsManager.ts
│   └── handleResponse()
│       ├── installPluginAndNotify()      // 安装插件
│       ├── addToNeverSuggest()           // 加入永不推荐列表
│       └── incrementIgnoredCount()       // 递增忽略计数
└── LspRecommendationMenu                 // src/components/LspRecommendation/LspRecommendationMenu.tsx
```

### 4.2 核心文件清单

| 文件路径 | 职责 |
|---------|------|
| `src/components/LspRecommendation/LspRecommendationMenu.tsx` | 本组件：UI 渲染与交互处理 |
| `src/hooks/useLspPluginRecommendation.tsx` | 推荐逻辑 hook：检测、状态管理、响应处理 |
| `src/hooks/usePluginRecommendationBase.tsx` | 基础推荐 hook：通用状态机与安装辅助 |
| `src/utils/plugins/lspRecommendation.ts` | LSP 推荐核心逻辑：匹配、过滤、配置管理 |
| `src/utils/plugins/lspPluginIntegration.ts` | 插件 LSP 服务器加载与配置解析 |
| `src/utils/config.ts` | 全局配置：持久化推荐偏好设置 |
| `src/bootstrap/state.ts` | 会话状态：`lspRecommendationShownThisSession` |
| `src/screens/REPL.tsx` | 主界面：整合 hook 与条件渲染菜单 |

### 4.3 依赖组件

| 组件 | 路径 | 用途 |
|------|------|------|
| `PermissionDialog` | `src/components/permissions/PermissionDialog.tsx` | 权限对话框容器 |
| `Select` | `src/components/CustomSelect/select.tsx` | 选项选择器 |
| `Box`, `Text` | `src/ink.ts` | Ink 渲染组件 |

---

## 5. 依赖与外部交互

### 5.1 运行时依赖

```typescript
// React 核心
import * as React from 'react';

// Ink 组件库（终端 UI）
import { Box, Text } from '../../ink.js';
import { Select } from '../CustomSelect/select.js';
import { PermissionDialog } from '../permissions/PermissionDialog.js';
```

### 5.2 外部系统交互

#### 5.2.1 Marketplace 系统
- **读取**: `getMarketplace()` / `getPluginById()`
- **来源**: GitHub 仓库、本地文件、URL
- **缓存**: `~/.claude/plugins/marketplaces/`

#### 5.2.2 插件安装系统
- **安装**: `cacheAndRegisterPlugin()` → 下载到 `~/.claude/plugins/`
- **启用**: 更新 `~/.claude/settings.json` 的 `enabledPlugins`
- **元数据**: `installed_plugins.json` 记录安装信息

#### 5.2.3 LSP 服务器管理
- **配置加载**: `getAllLspServers()` 合并内置 + 插件 LSP 配置
- **服务器启动**: `LSPServerManager` 按需启动 LSP 进程
- **文件同步**: `openFile/changeFile/saveFile/closeFile` 通知 LSP

#### 5.2.4 文件历史追踪
```typescript
// 通过 AppState 订阅 trackedFiles
const trackedFiles = useAppState(s => s.fileHistory.trackedFiles);
```

### 5.3 配置存储位置

| 配置项 | 存储文件 | 格式 |
|--------|---------|------|
| 全局推荐设置 | `~/.claude/config.json` | `lspRecommendationDisabled`, `lspRecommendationNeverPlugins`, `lspRecommendationIgnoredCount` |
| 会话状态 | 内存（bootstrap/state.ts） | `lspRecommendationShownThisSession: boolean` |
| 插件启用状态 | `~/.claude/settings.json` | `enabledPlugins: Record<string, boolean>` |

---

## 6. 风险、边界与改进建议

### 6.1 已知限制

#### 6.1.1 检测限制
- **仅检测内联配置**: `extractLspInfoFromManifest()` 只能读取 `manifest.lspServers` 内联配置，无法读取外部 `.lsp.json` 文件路径
- **字符串路径跳过**: 如果 `lspServers` 是字符串（如 `"./.lsp.json"`），则无法检测
- **二进制预检查**: 仅当 LSP 二进制已存在时才推荐，不会自动安装二进制

#### 6.1.2 会话限制
- **每会话一次**: `hasShownLspRecommendationThisSession()` 限制每会话只显示一次推荐，即使用户编辑多种类型的文件
- **超时检测**: 28秒阈值（`TIMEOUT_THRESHOLD_MS`）用于区分显式关闭和超时

### 6.2 边界情况

| 场景 | 行为 |
|------|------|
| 远程模式 (`--remote`) | 完全禁用推荐（`getIsRemoteMode()` 检查）|
| 推荐已显示中 | `usePluginRecommendationBase` 阻止并发检查 |
| 插件已安装 | `isPluginInstalled()` 过滤，不显示推荐 |
| 在"永不推荐"列表 | `neverPlugins.includes(pluginId)` 过滤 |
| 忽略5次后 | `isLspRecommendationsDisabled()` 返回 true，停止所有推荐 |
| 无文件扩展名 | `extname(filePath)` 为空时直接返回 |

### 6.3 潜在风险

#### 6.3.1 竞态条件
```typescript
// useLspPluginRecommendation.tsx
if (hasShownLspRecommendationThisSession()) {
  return null;  // 早期返回
}
// ... 异步检查 ...
setLspRecommendationShownThisSession(true);  // 可能多个检查同时通过
```
**缓解**: `usePluginRecommendationBase` 的 `isCheckingRef` 提供额外保护

#### 6.3.2 配置漂移
- `lspRecommendationIgnoredCount` 在进程间不共享，多个并发会话可能各自递增
- 全局配置写入使用 `saveGlobalConfig`，有锁机制但跨进程仍可能竞争

#### 6.3.3 二进制检查性能
```typescript
// lspRecommendation.ts
for (const { info, pluginId } of matchingPlugins) {
  const binaryExists = await isBinaryInstalled(info.command);  // 串行检查
}
```
**风险**: 多个插件匹配时串行检查可能较慢，但通常匹配数量很少

### 6.4 改进建议

#### 6.4.1 功能增强
1. **支持外部 .lsp.json 检测**: 在 Marketplace 缓存阶段预读取 `.lsp.json` 文件内容
2. **批量二进制检查**: 使用 `Promise.all()` 并行检查多个二进制
3. **文件类型聚合**: 一次推荐多个匹配的 LSP 插件，而非每会话一次
4. **安装后自动重启**: 当前需要手动重启，可探索热重载 LSP 服务器

#### 6.4.2 代码质量
1. **超时阈值配置化**: 将 `TIMEOUT_THRESHOLD_MS` 和 `AUTO_DISMISS_MS` 统一配置
2. **测试覆盖**: 添加单元测试覆盖四种响应选项的处理逻辑
3. **遥测**: 添加推荐展示/接受/拒绝的埋点，用于优化推荐算法

#### 6.4.3 用户体验
1. **预览功能**: 安装前展示插件将提供的具体功能
2. **延迟推荐**: 新用户首次使用时延迟推荐，避免信息过载
3. **快捷操作**: 添加键盘快捷键（如 `Y/N/Never/D`）快速响应

### 6.5 相关测试文件

建议检查以下测试文件（如果存在）：
- `src/utils/plugins/__tests__/lspRecommendation.test.ts`
- `src/hooks/__tests__/useLspPluginRecommendation.test.tsx`

当前代码中未发现直接测试文件，建议补充测试覆盖：
- `getMatchingLspPlugins()` 的过滤逻辑
- `handleResponse()` 的四种响应处理
- 自动关闭与忽略计数递增的关联
