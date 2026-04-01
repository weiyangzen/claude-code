# 研究文档: src/utils/plugins/schemas.ts

## 1. 场景与职责

`schemas.ts` 是 Claude Code 插件系统的核心 Schema 定义模块，负责使用 Zod 库定义和验证所有与插件相关的数据结构。它是整个插件生态系统的类型安全基石，确保从 marketplace 配置到插件清单、从安装记录到 MCP/LSP 服务器配置的所有数据都符合预期格式。

### 核心职责
- **类型安全**: 为插件系统提供完整的 TypeScript 类型定义和运行时验证
- **数据验证**: 验证 plugin.json、marketplace.json、installed_plugins.json 等配置文件的合法性
- **安全防护**: 防止路径遍历攻击、官方市场名称仿冒、非 ASCII 字符的同形异义攻击
- **配置标准化**: 统一 MCP、LSP、Hooks、Channels 等子系统的配置格式

## 2. 功能点目的

### 2.1 官方市场名称保护

**目的**: 防止第三方插件仿冒 Anthropic 官方市场。

**实现机制**:
- `ALLOWED_OFFICIAL_MARKETPLACE_NAMES`: 定义允许保留的官方市场名称集合
- `NO_AUTO_UPDATE_OFFICIAL_MARKETPLACES`: 定义不参与自动更新的官方市场
- `BLOCKED_OFFICIAL_NAME_PATTERN`: 正则表达式检测仿冒名称（如 "official-claude"、"anthropic-marketplace" 等变体）
- `NON_ASCII_PATTERN`: 检测非 ASCII 字符，防止同形异义攻击（如用西里尔字母 'а' 冒充拉丁字母 'a'）
- `isBlockedOfficialName()`: 综合判断名称是否被禁止
- `validateOfficialNameSource()`: 验证保留名称是否来自官方 GitHub 组织 (anthropics)

### 2.2 插件清单 Schema (PluginManifestSchema)

**目的**: 定义 plugin.json 的完整结构，支持插件的元数据、命令、代理、技能、钩子、MCP/LSP 服务器等配置。

**关键字段**:
- `name`: 插件唯一标识符（kebab-case，不能含空格）
- `version`: 语义化版本
- `description`, `author`, `homepage`, `license`: 元数据
- `dependencies`: 依赖的其他插件
- `commands`/`agents`/`skills`/`outputStyles`: 附加组件路径（支持单路径、路径数组或对象映射）
- `hooks`: 钩子配置（内联或外部文件路径）
- `mcpServers`: MCP 服务器配置
- `lspServers`: LSP 服务器配置
- `channels`: 消息通道声明（用于 Telegram、Slack 等集成）
- `userConfig`: 用户可配置选项（安装时提示）
- `settings`: 合并到设置级联的配置

### 2.3 市场配置 Schema (PluginMarketplaceSchema)

**目的**: 定义 marketplace.json 的结构，支持插件市场的发现、分类和安装。

**关键字段**:
- `name`: 市场名称（受保护名称限制）
- `owner`: 市场维护者信息
- `plugins`: 插件条目数组
- `forceRemoveDeletedPlugins`: 自动卸载已从市场移除的插件
- `allowCrossMarketplaceDependenciesOn`: 允许跨市场依赖的白名单

### 2.4 插件源 Schema (PluginSourceSchema)

**目的**: 支持多种插件来源格式，实现灵活的插件分发机制。

**支持的源类型**:
- 本地相对路径: `"./my-plugin"`
- NPM 包: `{ source: "npm", package: "@org/pkg", version?: string }`
- Python 包: `{ source: "pip", package: "pkg-name" }`
- Git URL: `{ source: "url", url: "https://...", ref?: string, sha?: string }`
- GitHub: `{ source: "github", repo: "owner/repo", ref?: string }`
- Git 子目录: `{ source: "git-subdir", url: "...", path: "subdir" }`（支持稀疏检出）

### 2.5 市场源 Schema (MarketplaceSourceSchema)

**目的**: 定义市场本身的来源，支持从多种渠道加载市场配置。

**支持的源类型**:
- `url`: 直接 URL 到 marketplace.json
- `github`: GitHub 仓库
- `git`: 任意 Git 仓库 URL
- `npm`: NPM 包
- `file`/`directory`: 本地文件系统路径
- `hostPattern`: 主机名正则匹配（用于策略限制）
- `pathPattern`: 路径正则匹配
- `settings`: 内联在 settings.json 中的市场定义

### 2.6 已安装插件 Schema

**目的**: 跟踪插件的安装状态，支持多作用域安装（V2 格式）。

**V1 格式** (`InstalledPluginsFileSchemaV1`):
```json
{
  "version": 1,
  "plugins": {
    "plugin@marketplace": { "version": "1.0.0", "installedAt": "...", "installPath": "..." }
  }
}
```

**V2 格式** (`InstalledPluginsFileSchemaV2`):
- 支持同一插件在不同作用域（managed/user/project/local）的多版本安装
- 每个插件 ID 映射到安装条目数组

### 2.7 辅助 Schema

- **CommandMetadataSchema**: 命令元数据（source/content、description、model、allowedTools 等）
- **PluginHooksSchema**: 钩子配置结构
- **PluginAuthorSchema**: 作者信息
- **PluginIdSchema**: 插件 ID 格式验证（`name@marketplace`）
- **DependencyRefSchema**: 依赖引用格式（支持字符串和对象形式，带版本前向兼容）
- **LspServerConfigSchema**: LSP 服务器详细配置

## 3. 具体技术实现

### 3.1 延迟 Schema 构造 (lazySchema)

```typescript
export function lazySchema<T>(factory: () => T): () => T {
  let cached: T | undefined
  return () => (cached ??= factory())
}
```

**设计目的**: 将 Zod Schema 的构造从模块加载时延迟到首次访问时，避免循环依赖和初始化性能问题。

### 3.2 路径验证模式

```typescript
const RelativePath = lazySchema(() => z.string().startsWith('./'))
const RelativeJSONPath = lazySchema(() => RelativePath().endsWith('.json'))
const RelativeMarkdownPath = lazySchema(() => RelativePath().endsWith('.md'))
```

**安全机制**: 所有相对路径必须以 `./` 开头，防止路径遍历攻击。

### 3.3 市场名称验证

```typescript
const MarketplaceNameSchema = lazySchema(() =>
  z
    .string()
    .min(1)
    .refine(name => !name.includes(' '), { message: 'Use kebab-case' })
    .refine(name => !name.includes('/') && !name.includes('\\') && !name.includes('..'), { ... })
    .refine(name => !isBlockedOfficialName(name), { message: 'Impersonates official marketplace' })
    .refine(name => name.toLowerCase() !== 'inline', { message: 'Reserved for session plugins' })
    .refine(name => name.toLowerCase() !== 'builtin', { message: 'Reserved for built-in plugins' })
)
```

**验证层级**:
1. 基本格式（非空、无空格）
2. 路径安全（无分隔符、无 `..`）
3. 官方名称保护（非保留名称或来自官方源）
4. 保留关键字（`inline`、`builtin`）

### 3.4 依赖引用规范化

```typescript
export const DependencyRefSchema = lazySchema(() =>
  z.union([
    z.string()
      .regex(DEP_REF_REGEX, 'Dependency must be a plugin name, optionally qualified with @marketplace')
      .transform(s => s.replace(/@\^[^@]*$/, '')), // 去除版本后缀
    z.object({ name: ..., marketplace: ... })
      .transform(o => o.marketplace ? `${o.name}@${o.marketplace}` : o.name),
  ]),
)
```

**前向兼容设计**: 允许版本约束语法（如 `@^1.2`），但当前版本仅提取名称部分，为未来版本约束功能预留空间。

### 3.5 自动更新判断

```typescript
export function isMarketplaceAutoUpdate(
  marketplaceName: string,
  entry: { autoUpdate?: boolean },
): boolean {
  return (
    entry.autoUpdate ??
    (ALLOWED_OFFICIAL_MARKETPLACE_NAMES.has(normalizedName) &&
      !NO_AUTO_UPDATE_OFFICIAL_MARKETPLACES.has(normalizedName))
  )
}
```

**逻辑**: 用户显式设置 > 官方市场默认启用（除明确排除的如 `knowledge-work-plugins`）> 默认禁用。

## 4. 关键代码路径与文件引用

### 4.1 核心依赖

| 依赖文件 | 用途 |
|---------|------|
| `zod/v4` | Schema 定义和验证库 |
| `src/schemas/hooks.ts` | HooksSchema 定义（打破循环依赖） |
| `src/services/mcp/types.ts` | McpServerConfigSchema 定义 |
| `src/utils/lazySchema.ts` | 延迟 Schema 构造工具 |

### 4.2 被调用方（使用者）

| 调用方 | 用途 |
|-------|------|
| `src/utils/plugins/validatePlugin.ts` | 验证 plugin.json 和 marketplace.json |
| `src/utils/plugins/pluginLoader.ts` | 加载插件时使用 Schema 验证 |
| `src/utils/plugins/loadPluginCommands.ts` | 使用 CommandMetadata 类型 |
| `src/utils/plugins/loadPluginAgents.ts` | 加载代理组件 |
| `src/utils/plugins/marketplaceManager.ts` | 管理市场配置 |
| `src/utils/plugins/dependencyResolver.ts` | 解析依赖关系 |
| `src/utils/plugins/installedPluginsManager.ts` | 管理已安装插件记录 |
| `src/utils/plugins/pluginVersioning.ts` | 版本计算 |
| `src/utils/plugins/lspPluginIntegration.ts` | LSP 服务器集成 |
| `src/services/plugins/pluginCliCommands.ts` | CLI 插件命令 |
| `src/cli/handlers/plugins.ts` | 插件 CLI 处理 |
| `src/utils/settings/types.ts` | 设置类型集成 |

### 4.3 类型导出

文件末尾导出所有推断类型供外部使用：
- `CommandMetadata`, `MarketplaceSource`, `PluginAuthor`
- `PluginSource`, `PluginManifest`, `PluginManifestChannel`
- `PluginMarketplace`, `PluginMarketplaceEntry`
- `PluginId`, `InstalledPlugin`, `PluginScope`
- `KnownMarketplace`, `KnownMarketplacesFile`

## 5. 依赖与外部交互

### 5.1 输入依赖

1. **Zod v4**: 核心验证库
2. **HooksSchema** (`src/schemas/hooks.ts`): 钩子配置结构
3. **McpServerConfigSchema** (`src/services/mcp/types.ts`): MCP 服务器配置
4. **lazySchema** (`src/utils/lazySchema.ts`): 延迟初始化工具

### 5.2 输出消费

1. **验证层** (`validatePlugin.ts`): 运行时验证
2. **加载层** (`pluginLoader.ts`, `loadPluginCommands.ts`, `loadPluginAgents.ts`): 数据加载
3. **管理层** (`marketplaceManager.ts`, `installedPluginsManager.ts`): 状态管理
4. **CLI 层** (`pluginCliCommands.ts`, `cli/handlers/plugins.ts`): 用户交互
5. **类型系统**: 为整个插件系统提供 TypeScript 类型

## 6. 风险、边界与改进建议

### 6.1 安全风险

| 风险 | 缓解措施 | 建议 |
|-----|---------|------|
| 路径遍历攻击 | 所有路径必须以 `./` 开头；禁止 `..` | 考虑增加对符号链接的额外检查 |
| 官方名称仿冒 | 正则表达式检测 + GitHub 组织验证 | 考虑增加 Levenshtein 距离检测近似名称 |
| 同形异义攻击 | 非 ASCII 字符检测 | 考虑增加 IDN 同形异义字符库比对 |
| 版本混淆 | marketplace.json 版本与 plugin.json 版本分离 | 已在 validatePlugin.ts 中增加版本不匹配警告 |

### 6.2 边界情况

1. **Schema 严格性**:
   - 顶层字段使用宽松模式（未知字段被静默剥离）
   - 嵌套对象（userConfig、channels、lspServers）使用严格模式
   - 权衡：运行时弹性 vs 开发者反馈

2. **版本兼容性**:
   - `DependencyRefSchema` 接受版本语法但忽略它（前向兼容）
   - `InstalledPluginsFileSchema` 支持 V1/V2 联合验证

3. **Git URL 格式**:
   - 故意不强制 `.git` 后缀，支持 Azure DevOps 和 AWS CodeCommit

### 6.3 改进建议

1. **性能优化**:
   - 考虑为高频验证路径添加编译后的 Zod Schema 缓存
   - 评估是否需要将部分验证逻辑移至构建时

2. **安全增强**:
   - 增加对市场源 URL 的协议限制（仅允许 https/ssh）
   - 考虑增加插件内容哈希验证

3. **开发者体验**:
   - 为 `claude plugin validate` 提供更详细的错误上下文
   - 考虑增加 Schema 版本字段以支持未来迁移

4. **代码组织**:
   - 文件已接近 1700 行，考虑按功能拆分为多个子模块：
     - `schemas/manifest.ts`: 插件清单 Schema
     - `schemas/marketplace.ts`: 市场配置 Schema
     - `schemas/source.ts`: 源配置 Schema
     - `schemas/installed.ts`: 已安装插件 Schema
     - `schemas/index.ts`: 统一导出
