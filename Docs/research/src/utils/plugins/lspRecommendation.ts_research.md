# lspRecommendation.ts 深度研究文档

## 文件元数据
- **路径**: `src/utils/plugins/lspRecommendation.ts`
- **大小**: 10,695 bytes
- **核心职责**: 基于文件扩展名和已安装 LSP 二进制文件，向用户推荐合适的 LSP 插件

---

## 一、场景与职责

### 1.1 功能定位
本模块是 Claude Code 的 **LSP 插件推荐引擎**，负责：
1. 扫描所有已安装的市场源（marketplaces）中的 LSP 插件
2. 根据文件扩展名匹配支持该文件类型的 LSP 插件
3. 验证对应的 LSP 二进制文件是否已安装在系统上
4. 过滤掉已安装或在用户"不再推荐"列表中的插件
5. 提供推荐管理功能（忽略、禁用、重置等）

### 1.2 业务场景
- **文件打开时**: 用户打开特定类型文件时检测并推荐 LSP 插件
- **首次使用**: 新用户首次使用 Claude Code 时引导安装常用 LSP
- **渐进式发现**: 随着用户使用不同语言，逐步推荐相应 LSP

---

## 二、功能点目的

### 2.1 推荐条件
一个 LSP 插件被推荐需满足以下所有条件：
1. **扩展名匹配**: 插件支持当前文件的扩展名
2. **二进制存在**: 对应的 LSP 服务器命令在系统 PATH 中可找到
3. **未安装**: 插件尚未安装
4. **未忽略**: 不在用户的"不再推荐"列表中
5. **未禁用**: 用户未全局禁用 LSP 推荐功能

### 2.2 推荐排序
- **官方市场优先**: 来自官方市场（Anthropic）的插件排在前面
- **非官方在后**: 第三方市场插件排在后面

### 2.3 防骚扰机制
- **忽略计数**: 用户连续忽略推荐 5 次后自动禁用推荐功能
- **永久忽略**: 用户可将特定插件加入"不再推荐"列表
- **手动重置**: 提供 `resetIgnoredCount()` 重置忽略计数

---

## 三、具体技术实现

### 3.1 核心数据类型

#### 3.1.1 推荐结果
```typescript
export type LspPluginRecommendation = {
  pluginId: string        // "plugin-name@marketplace-name"
  pluginName: string      // 人类可读的插件名称
  marketplaceName: string // 市场名称
  description?: string    // 插件描述
  isOfficial: boolean    // 是否来自官方市场
  extensions: string[]   // 支持的文件扩展名
  command: string        // LSP 服务器命令（如 "typescript-language-server"）
}
```

#### 3.1.2 内部 LSP 信息
```typescript
type LspInfo = {
  extensions: Set<string>  // 支持的扩展名集合
  command: string          // LSP 命令
}

type LspPluginInfo = {
  entry: PluginMarketplaceEntry
  marketplaceName: string
  extensions: Set<string>
  command: string
  isOfficial: boolean
}
```

### 3.2 关键流程

#### 3.2.1 获取匹配插件（`getMatchingLspPlugins`）
```
getMatchingLspPlugins(filePath)
  └── isLspRecommendationsDisabled() → 检查全局禁用
  └── 提取文件扩展名（extname，转小写）
  └── getLspPluginsFromMarketplaces() → 获取所有 LSP 插件
      ├── loadKnownMarketplacesConfig() → 加载市场配置
      ├── 遍历每个市场
      │   └── getMarketplace(marketplaceName) → 获取市场数据
      │   └── 遍历市场中的插件
      │       └── extractLspInfoFromManifest(entry.lspServers)
      │           ├── 跳过字符串路径（无法从市场读取）
      │           └── 从内联配置提取 extensions 和 command
      └── 返回 Map<pluginId, LspPluginInfo>
  └── 过滤匹配插件
      ├── 扩展名匹配检查
      ├── 检查 neverPlugins 列表
      └── 检查是否已安装（isPluginInstalled）
  └── 过滤二进制存在
      └── isBinaryInstalled(info.command) → 异步检查
  └── 排序（官方优先）
  └── 转换为 LspPluginRecommendation 数组
```

#### 3.2.2 LSP 信息提取（`extractLspInfoFromManifest`）
```typescript
function extractLspInfoFromManifest(
  lspServers: PluginMarketplaceEntry['lspServers']
): LspInfo | null
```
- **限制**: 仅支持内联配置，不支持外部 `.lsp.json` 文件路径
- **原因**: 市场条目中的字符串路径指向的文件在安装前不可用
- **处理逻辑**:
  - 字符串路径 → 返回 null
  - 数组 → 遍历查找第一个有效的内联配置
  - 内联对象 → 直接提取

#### 3.2.3 内联配置提取（`extractFromServerConfigRecord`）
```typescript
function extractFromServerConfigRecord(
  serverConfigs: Record<string, unknown>
): LspInfo | null
```
- 遍历每个服务器配置
- 收集所有 `extensionToLanguage` 映射中的扩展名
- 使用第一个有效的 `command` 字段
- 返回条件: `command` 存在且 `extensions` 非空

### 3.3 配置管理

#### 3.3.1 全局配置字段
```typescript
// src/utils/config.ts 中的相关配置
type GlobalConfig = {
  lspRecommendationDisabled?: boolean        // 全局禁用
  lspRecommendationIgnoredCount?: number     // 忽略计数
  lspRecommendationNeverPlugins?: string[]   // 不再推荐列表
}
```

#### 3.3.2 配置操作函数
| 函数 | 功能 |
|------|------|
| `addToNeverSuggest(pluginId)` | 将插件加入不再推荐列表 |
| `incrementIgnoredCount()` | 增加忽略计数 |
| `resetIgnoredCount()` | 重置忽略计数为 0 |
| `isLspRecommendationsDisabled()` | 检查推荐是否被禁用 |

### 3.4 关键常量
```typescript
const MAX_IGNORED_COUNT = 5  // 最大忽略次数，超过则自动禁用
```

---

## 四、关键代码路径与文件引用

### 4.1 入口点
| 函数 | 导出类型 | 调用方 |
|------|----------|--------|
| `getMatchingLspPlugins` | async | `src/hooks/useLspPluginRecommendation.tsx`, `src/utils/plugins/hintRecommendation.ts` |
| `addToNeverSuggest` | sync | `src/hooks/useLspPluginRecommendation.tsx` |
| `incrementIgnoredCount` | sync | `src/hooks/useLspPluginRecommendation.tsx` |
| `resetIgnoredCount` | sync | `src/commands/plugin/PluginSettings.tsx` |
| `isLspRecommendationsDisabled` | sync | `src/hooks/useLspPluginRecommendation.tsx`, `src/screens/REPL.tsx` |

### 4.2 关键依赖
```typescript
// 核心依赖
import { isBinaryInstalled } from '../binaryCheck.js'
import { getGlobalConfig, saveGlobalConfig } from '../config.js'
import { isPluginInstalled } from './installedPluginsManager.js'
import { getMarketplace, loadKnownMarketplacesConfig } from './marketplaceManager.js'
import { ALLOWED_OFFICIAL_MARKETPLACE_NAMES } from './schemas.js'

// 类型定义
import type { PluginMarketplaceEntry } from './schemas.js'
```

### 4.3 文件引用关系
```
lspRecommendation.ts
  ├── ../binaryCheck.js              # isBinaryInstalled
  ├── ../config.js                   # getGlobalConfig, saveGlobalConfig
  ├── ./installedPluginsManager.js   # isPluginInstalled
  ├── ./marketplaceManager.js        # getMarketplace, loadKnownMarketplacesConfig
  ├── ./schemas.js                   # ALLOWED_OFFICIAL_MARKETPLACE_NAMES, PluginMarketplaceEntry
  └── ../debug.js                    # logForDebugging
```

---

## 五、依赖与外部交互

### 5.1 上游依赖（被调用）
| 模块 | 用途 |
|------|------|
| `binaryCheck.ts` | 检查 LSP 二进制文件是否在系统 PATH 中 |
| `config.ts` | 读取和保存全局配置 |
| `installedPluginsManager.ts` | 检查插件是否已安装 |
| `marketplaceManager.ts` | 获取市场配置和插件列表 |
| `schemas.ts` | 官方市场名称白名单、类型定义 |

### 5.2 下游消费者（调用方）
| 模块 | 用途 |
|------|------|
| `src/hooks/useLspPluginRecommendation.tsx` | React Hook 封装，UI 集成 |
| `src/screens/REPL.tsx` | REPL 界面集成 |
| `src/utils/plugins/hintRecommendation.ts` | 提示推荐集成 |
| `src/commands/plugin/PluginSettings.tsx` | 插件设置界面 |

### 5.3 配置交互
```typescript
// 配置存储位置: ~/.claude/settings.json
{
  "lspRecommendationDisabled": false,
  "lspRecommendationIgnoredCount": 2,
  "lspRecommendationNeverPlugins": [
    "python-lsp@claude-code-marketplace"
  ]
}
```

---

## 六、风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 检测局限性
- **问题**: 无法检测使用外部 `.lsp.json` 文件的 LSP 插件
- **原因**: 市场条目中的字符串路径指向的文件在安装前不可用
- **影响**: 部分 LSP 插件可能无法被推荐

#### 6.1.2 二进制检查性能
- **问题**: `isBinaryInstalled` 可能涉及文件系统遍历，频繁调用影响性能
- **缓解**: 调用方应缓存结果，避免重复检查

#### 6.1.3 扩展名冲突
- **问题**: 多个 LSP 插件可能支持相同的扩展名
- **缓解**: 官方市场插件优先排序，但最终选择由用户决定

### 6.2 边界情况

| 场景 | 处理方式 |
|------|----------|
| 文件无扩展名 | 提前返回空数组 |
| 市场加载失败 | 记录调试日志，继续处理其他市场 |
| 二进制检查失败 | 跳过该插件，不中断流程 |
| 插件已在 never 列表 | 记录调试日志，跳过 |
| 插件已安装 | 记录调试日志，跳过 |
| 忽略计数达到上限 | `isLspRecommendationsDisabled` 返回 true |

### 6.3 改进建议

#### 6.3.1 功能增强
1. **项目级推荐**: 基于项目依赖（package.json、Cargo.toml 等）推荐 LSP
2. **智能排序**: 基于用户使用频率、社区评分排序推荐
3. **批量安装**: 支持一键安装多个推荐的 LSP 插件
4. **推荐解释**: 向用户说明为什么推荐某个插件

#### 6.3.2 准确性提升
1. **版本匹配**: 检查 LSP 二进制版本与插件要求是否匹配
2. **多二进制支持**: 支持一个插件对应多个 LSP 命令（如前端插件对应 tsserver 和 eslint）
3. **动态检测**: 安装后动态检测新可用的 LSP 插件

#### 6.3.3 用户体验
1. **推荐时机优化**: 避免在用户专注工作时弹出推荐
2. **推荐频率控制**: 限制同一插件的推荐次数
3. **一键禁用**: 提供快速禁用所有推荐的入口

#### 6.3.4 性能优化
1. **缓存市场数据**: 避免每次调用都重新加载市场
2. **并行二进制检查**: 多个候选插件的二进制检查可并行化
3. **增量更新**: 仅检查新增或变更的插件

#### 6.3.5 可观测性
1. **推荐统计**: 记录推荐展示、点击、忽略、安装转化率
2. **用户反馈**: 允许用户对推荐进行反馈（有用/无用）
3. **A/B 测试**: 支持不同推荐策略的效果对比

### 6.4 技术债务
1. **硬编码限制**: `MAX_IGNORED_COUNT = 5` 应为可配置
2. **日志标签**: `[lspRecommendation]` 硬编码，应使用统一日志工具
3. **类型守卫**: `isRecord` 函数可提取为通用工具
4. **错误处理**: 市场加载错误仅记录调试日志，用户无感知

---

## 七、附录

### 7.1 推荐流程时序图
```
用户打开文件
    │
    ▼
REPL.tsx / useLspPluginRecommendation
    │
    ▼
getMatchingLspPlugins(filePath)
    │
    ├── 检查全局禁用
    ├── 提取扩展名
    ├── 获取所有 LSP 插件
    │   └── 遍历市场
    │       └── 提取内联 LSP 配置
    ├── 过滤匹配扩展名
    ├── 过滤已安装
    ├── 过滤 never 列表
    ├── 检查二进制存在
    └── 排序（官方优先）
    │
    ▼
返回推荐列表
    │
    ▼
UI 展示推荐
```

### 7.2 配置示例
```typescript
// 用户配置示例
{
  "lspRecommendationDisabled": false,
  "lspRecommendationIgnoredCount": 3,
  "lspRecommendationNeverPlugins": [
    "typescript-language-server@claude-code-marketplace",
    "rust-analyzer@claude-code-marketplace"
  ]
}
```

### 7.3 市场条目 LSP 配置示例
```json
{
  "name": "typescript",
  "description": "TypeScript language support",
  "lspServers": {
    "typescript": {
      "command": "typescript-language-server",
      "args": ["--stdio"],
      "extensionToLanguage": {
        ".ts": "typescript",
        ".tsx": "typescriptreact",
        ".js": "javascript",
        ".jsx": "javascriptreact"
      }
    }
  }
}
```

### 7.4 官方市场白名单
```typescript
ALLOWED_OFFICIAL_MARKETPLACE_NAMES = new Set([
  'claude-code-marketplace',
  'claude-code-plugins',
  'claude-plugins-official',
  'anthropic-marketplace',
  'anthropic-plugins',
  'agent-skills',
  'life-sciences',
  'knowledge-work-plugins',
])
```
