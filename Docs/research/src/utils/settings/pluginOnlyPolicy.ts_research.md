# pluginOnlyPolicy.ts 研究文档

## 场景与职责

`pluginOnlyPolicy.ts` 实现了 `strictPluginOnlyCustomization` 策略的验证逻辑。该策略允许企业管理员限制特定自定义表面（customization surfaces）只能从插件加载，阻止用户级和项目级的自定义配置。

这是企业安全策略的一部分，与 `strictKnownMarketplaces` 配合实现端到端的自定义内容管控。

## 功能点目的

### 1. 插件独占检查 (`isRestrictedToPluginOnly`)
- **策略来源**: 从 `policySettings` 读取 `strictPluginOnlyCustomization`
- **锁定模式**:
  - `true`: 锁定所有四个表面（skills, agents, hooks, mcp）
  - `string[]`: 只锁定指定的表面
  - `undefined/false`: 不锁定任何表面
- **锁定含义**: 跳过用户级（`~/.claude/*`）和项目级（`.claude/*`）的源
- **例外**: 托管（policySettings）和插件提供的源始终加载

### 2. 管理员信任源检查 (`isSourceAdminTrusted`)
- **用途**: 检查自定义内容的源是否在严格策略下被信任
- **信任源集合**:
  - `plugin` - 插件提供的（通过 `strictKnownMarketplaces` 单独管控）
  - `policySettings` - 托管设置（管理员控制）
  - `built-in` / `builtin` / `bundled` - 内置/捆绑的（随 CLI 分发）
- **使用模式**: `!isRestrictedToPluginOnly(surface) || isSourceAdminTrusted(item.source)`

## 具体技术实现

### 关键类型和常量

```typescript
// 可锁定的自定义表面
import type { CUSTOMIZATION_SURFACES } from './types.js'
export type CustomizationSurface = (typeof CUSTOMIZATION_SURFACES)[number]
// CUSTOMIZATION_SURFACES = ['skills', 'agents', 'hooks', 'mcp']

// 管理员信任源集合
const ADMIN_TRUSTED_SOURCES: ReadonlySet<string> = new Set([
  'plugin',
  'policySettings',
  'built-in',
  'builtin',
  'bundled',
])
```

### 关键函数

```typescript
/**
 * 检查表面是否被锁定为插件独占
 */
export function isRestrictedToPluginOnly(
  surface: CustomizationSurface,
): boolean {
  const policy =
    getSettingsForSource('policySettings')?.strictPluginOnlyCustomization
  if (policy === true) return true
  if (Array.isArray(policy)) return policy.includes(surface)
  return false
}

/**
 * 检查源是否被管理员信任
 */
export function isSourceAdminTrusted(source: string | undefined): boolean {
  return source !== undefined && ADMIN_TRUSTED_SOURCES.has(source)
}
```

### 使用模式

```typescript
// 在 frontmatter-hook 注册等场景中的典型使用模式
const allowed = !isRestrictedToPluginOnly(surface) || isSourceAdminTrusted(item.source)
if (item.hooks && allowed) {
  register(...)
}
```

### 关键代码路径

| 函数 | 行号 | 说明 |
|------|------|------|
| `isRestrictedToPluginOnly` | 19-27 | 检查表面锁定状态 |
| `isSourceAdminTrusted` | 58-60 | 检查源信任状态 |
| `ADMIN_TRUSTED_SOURCES` | 40-46 | 信任源集合定义 |

## 依赖与外部交互

### 导入依赖

| 模块 | 路径 | 用途 |
|------|------|------|
| `getSettingsForSource` | `./settings.js` | 获取策略设置 |
| `CUSTOMIZATION_SURFACES` | `./types.js` | 可锁定表面常量 |

### 被调用方

| 模块 | 路径 | 用途 |
|------|------|------|
| `loadSkillsDir.ts` | `../../skills/loadSkillsDir.js` | 技能加载管控 |
| `mcp/config.ts` | `../../services/mcp/config.js` | MCP 服务器管控 |
| `markdownConfigLoader.ts` | `../markdownConfigLoader.js` | Markdown 配置加载 |
| `processSlashCommand.tsx` | `../processUserInput/processSlashCommand.js` | 斜杠命令处理 |
| `hooksConfigSnapshot.ts` | `../hooks/hooksConfigSnapshot.js` | Hooks 配置管控 |
| `runAgent.ts` | `../../tools/AgentTool/runAgent.js` | Agent 执行管控 |

### 导出 API

```typescript
export type CustomizationSurface = 'skills' | 'agents' | 'hooks' | 'mcp'

export function isRestrictedToPluginOnly(surface: CustomizationSurface): boolean
export function isSourceAdminTrusted(source: string | undefined): boolean
```

## 风险、边界与改进建议

### 风险点

1. **策略读取时机**: `isRestrictedToPluginOnly` 每次调用都读取设置，虽然设置有缓存，但频繁调用仍有开销。

2. **源名称不一致**: `built-in`（带连字符）和 `builtin`（无连字符）都作为信任源，这是因为不同的子系统使用了不同的命名约定。

3. **表面名称前向兼容**: `types.ts` 中的 `CUSTOMIZATION_SURFACES` 是常量数组，添加新表面需要修改多个地方。

### 边界情况

| 场景 | 行为 |
|------|------|
| `strictPluginOnlyCustomization: true` | 所有表面锁定 |
| `strictPluginOnlyCustomization: ['skills', 'hooks']` | 只锁定 skills 和 hooks |
| `strictPluginOnlyCustomization: false` | 不锁定 |
| `strictPluginOnlyCustomization: undefined` | 不锁定 |
| 未知的 surface 名称 | TypeScript 类型错误（编译时） |
| source 为 undefined | `isSourceAdminTrusted` 返回 false |

### 改进建议

1. **性能优化**: 考虑缓存策略值，或使用 React Context 在 UI 层传递
2. **命名统一**: 考虑统一 `built-in` 和 `builtin` 的使用
3. **扩展性**: 考虑使用配置驱动的方式定义可锁定表面
4. **审计日志**: 考虑在策略阻止加载时记录审计日志
5. **用户反馈**: 在 UI 中显示哪些表面被锁定，以及为什么

## 文件引用

- **本文件**: `src/utils/settings/pluginOnlyPolicy.ts`
- **相关文件**:
  - `src/utils/settings/types.ts` - CUSTOMIZATION_SURFACES 定义
  - `src/utils/settings/settings.ts` - 设置获取
  - `src/utils/settings/constants.ts` - 设置源常量

## 企业安全策略说明

`strictPluginOnlyCustomization` 是企业安全策略的一部分，与 `strictKnownMarketplaces` 配合工作：

```
端到端管控流程:
1. strictKnownMarketplaces: 控制哪些插件市场可以被添加
   - 阻止未授权市场的插件下载
   
2. strictPluginOnlyCustomization: 控制自定义内容的来源
   - 只允许插件、托管设置和内置内容
   - 阻止用户和项目的自定义配置

示例配置 (managed-settings.json):
{
  "strictKnownMarketplaces": [
    { "source": "github", "org": "mycompany", "repo": "claude-plugins" }
  ],
  "strictPluginOnlyCustomization": ["skills", "agents", "hooks"]
}
```

这种组合确保：
- 插件只能从公司批准的仓库安装
- 自定义技能、agents 和 hooks 只能来自这些受控插件
- 用户无法通过项目设置绕过管控
