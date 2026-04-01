# pluginIdentifier.ts 研究文档

## 场景与职责

`src/utils/plugins/pluginIdentifier.ts` 是 Claude Code 插件系统的**标识符解析与作用域映射中心**。它提供了一组纯工具函数，用于：

1. **解析与构建插件标识符**：在 `name@marketplace` 格式与结构化对象之间转换。
2. **官方市场判定**：判断一个 marketplace 名称是否属于 Anthropic 控制的官方市场，用于 telemetry 中的 PII  redaction 决策。
3. **插件作用域（scope）与设置来源（setting source）的双向映射**：将 `userSettings`/`projectSettings`/`localSettings` 等配置层映射到 `user`/`project`/`local` 安装作用域，反之亦然。

该模块被插件系统的几乎所有子模块和 UI 层广泛引用，是插件生态的“基础设施代码”。

---

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| `parsePluginIdentifier(plugin)` | 将 `"name@marketplace"` 解析为 `{name, marketplace?}`。仅使用第一个 `@` 作为分隔符，后续 `@` 被忽略。 |
| `buildPluginId(name, marketplace?)` | 从 name 和可选 marketplace 构建插件 ID 字符串。 |
| `isOfficialMarketplaceName(marketplace)` | 判断 marketplace 是否在 `ALLOWED_OFFICIAL_MARKETPLACE_NAMES` 集合中，用于 telemetry 数据分级。 |
| `scopeToSettingSource(scope)` | 将插件安装作用域转换为可编辑设置来源（`user`→`userSettings` 等）。`managed` 作用域不允许安装，会抛出 Error。 |
| `settingSourceToScope(source)` | 将可编辑设置来源转换为插件安装作用域（`userSettings`→`user` 等）。 |
| `SETTING_SOURCE_TO_SCOPE` | 常量映射表，被上述两个函数及外部模块共享。 |

---

## 具体技术实现

### 关键流程

#### 1. 标识符解析

```typescript
export function parsePluginIdentifier(plugin: string): ParsedPluginIdentifier {
  if (plugin.includes('@')) {
    const parts = plugin.split('@')
    return { name: parts[0] || '', marketplace: parts[1] }
  }
  return { name: plugin }
}
```

- ** intentionally 只认第一个 `@`**：注释说明 marketplace 名称不应包含 `@`，因此 `plugin@market@place` 会被解析为 `{name: 'plugin', marketplace: 'market'}`，第二个 `@place` 被丢弃。
- 空 name 场景：`parts[0]` 可能为空字符串（如输入为 `@marketplace`），函数不会抛异常，而是返回空 name。

#### 2. 官方市场判定

```typescript
export function isOfficialMarketplaceName(marketplace: string | undefined): boolean {
  return (
    marketplace !== undefined &&
    ALLOWED_OFFICIAL_MARKETPLACE_NAMES.has(marketplace.toLowerCase())
  )
}
```

- 大小写不敏感比较。
- 允许的官方市场名称定义在 `schemas.ts` 的 `ALLOWED_OFFICIAL_MARKETPLACE_NAMES` 集合中，包括：
  - `claude-code-marketplace`
  - `claude-code-plugins`
  - `claude-plugins-official`
  - `anthropic-marketplace`
  - `anthropic-plugins`
  - `agent-skills`
  - `life-sciences`
  - `knowledge-work-plugins`

#### 3. Scope 与 Setting Source 映射

```typescript
export const SETTING_SOURCE_TO_SCOPE = {
  policySettings: 'managed',
  userSettings: 'user',
  projectSettings: 'project',
  localSettings: 'local',
  flagSettings: 'flag',
} as const satisfies Record<SettingSource, ExtendedPluginScope>
```

- `ExtendedPluginScope = PluginScope | 'flag'`：`'flag'` 是会话级作用域（来自 `--plugin-dir` 或 CLI flag），**不持久化**到 `installed_plugins.json`。
- `PersistablePluginScope = Exclude<ExtendedPluginScope, 'flag'>`：可用于持久化的作用域。
- `scopeToSettingSource` 对 `managed` 抛出 Error，因为 `managed` 来自企业策略，用户不能通过安装操作直接写入。

---

## 关键代码路径与文件引用

### 本文件导出

| 导出 | 用途 |
|------|------|
| `parsePluginIdentifier(plugin)` | 解析插件 ID |
| `buildPluginId(name, marketplace?)` | 构建插件 ID |
| `isOfficialMarketplaceName(marketplace)` | 官方市场判定 |
| `scopeToSettingSource(scope)` | scope → setting source |
| `settingSourceToScope(source)` | setting source → scope |
| `SETTING_SOURCE_TO_SCOPE` | 映射常量 |
| `ExtendedPluginScope` / `PersistablePluginScope` / `ParsedPluginIdentifier` | 类型定义 |

### 直接依赖文件

- `src/utils/settings/constants.ts`：`EditableSettingSource`, `SettingSource`
- `src/utils/plugins/schemas.ts`：`ALLOWED_OFFICIAL_MARKETPLACE_NAMES`, `PluginScope`

### 调用方文件（广泛）

该模块是插件系统中被引用最广泛的工具模块之一，主要调用方包括：

- `src/utils/plugins/pluginAutoupdate.ts`：`parsePluginIdentifier`
- `src/utils/plugins/pluginBlocklist.ts`：间接通过 `marketplaceManager.ts`
- `src/utils/plugins/installedPluginsManager.ts`：`parsePluginIdentifier`, `settingSourceToScope`
- `src/utils/plugins/marketplaceManager.ts`：`parsePluginIdentifier`
- `src/utils/plugins/pluginLoader.ts`：`parsePluginIdentifier`
- `src/utils/plugins/dependencyResolver.ts`：`parsePluginIdentifier`
- `src/utils/plugins/pluginInstallationHelpers.ts`：`parsePluginIdentifier`, `isOfficialMarketplaceName`, `scopeToSettingSource`
- `src/services/plugins/pluginOperations.ts`：`parsePluginIdentifier`, `scopeToSettingSource`
- `src/services/plugins/pluginCliCommands.ts`：`parsePluginIdentifier`
- `src/cli/handlers/plugins.ts`：`parsePluginIdentifier`, `scopeToSettingSource`, `settingSourceToScope`
- `src/cli/print.ts`：`parsePluginIdentifier`
- `src/commands/plugin/ManagePlugins.tsx`：`parsePluginIdentifier`, `PersistablePluginScope`
- `src/utils/processUserInput/processSlashCommand.tsx`：`parsePluginIdentifier`, `isOfficialMarketplaceName`
- `src/services/mcp/channelAllowlist.ts` / `channelNotification.ts`：`parsePluginIdentifier`
- `src/utils/telemetry/pluginTelemetry.ts`：`isOfficialMarketplaceName`
- `src/tools/SkillTool/SkillTool.ts`：`parsePluginIdentifier`, `buildPluginId`
- `src/utils/plugins/hintRecommendation.ts`：`parsePluginIdentifier`
- `src/utils/plugins/pluginStartupCheck.ts`：`parsePluginIdentifier`, `scopeToSettingSource`

---

## 依赖与外部交互

### 外部系统/配置

- **`ALLOWED_OFFICIAL_MARKETPLACE_NAMES`**（定义于 `schemas.ts`）：硬编码的 Anthropic 官方市场白名单。该集合的变更需要同步更新本模块及 telemetry redaction 逻辑。
- **Settings 常量层**：`SettingSource` 和 `EditableSettingSource` 来自 `src/utils/settings/constants.ts`，确保插件作用域与 Claude Code 的配置层级模型一致。

### 与 Telemetry 的交互

- `isOfficialMarketplaceName` 的判定结果直接影响 analytics 事件中的字段处理：
  - 官方市场的 `plugin_id` 可直接写入 `additional_metadata`。
  - 非官方市场的 `plugin_id` 会被 redacted 为 `'third-party'`，仅保留 `_PROTO_plugin_name` 和 `_PROTO_marketplace_name` 等 PII-tagged 字段到受控列。

---

## 风险、边界与改进建议

### 风险与边界

1. **`parsePluginIdentifier` 对空 name 不抛异常**
   - 输入 `"@marketplace"` 会返回 `{name: '', marketplace: 'marketplace'}`。调用方若未检查空 name，可能构造出非法的插件 ID 或导致后续逻辑异常。

2. **`scopeToSettingSource('managed')` 会抛出 Error**
   - 这是一个运行时断言。虽然设计上 `managed` scope 不能通过用户安装产生，但如果调用方未提前过滤，可能在某些边缘流程中触发未捕获异常。

3. **官方市场名单硬编码**
   - `ALLOWED_OFFICIAL_MARKETPLACE_NAMES` 是编译期常量。新增官方市场需要发版更新，无法通过远程配置热更新。若紧急需要新增官方市场，必须等待客户端 release。

4. **映射表的维护负担**
   - `SETTING_SOURCE_TO_SCOPE` 与 `SCOPE_TO_EDITABLE_SOURCE` 是手动维护的双向映射。未来若增加新的 setting source 或 plugin scope，必须同时更新两个映射及类型定义，容易遗漏。

5. **无输入校验**
   - `buildPluginId` 对 `name` 和 `marketplace` 没有任何格式校验（如是否包含空格、是否为空字符串）。非法输入会原样拼接，导致后续 schema 验证失败。

### 改进建议

1. **增强 `parsePluginIdentifier` 的健壮性**
   - 可考虑增加对空 name 的 warn log 或返回 `null` 的严格版本（`parsePluginIdentifierStrict`），供关键路径使用，防止非法 ID 向下传播。

2. **将官方市场名单配置化**
   - 若业务允许，可将 `ALLOWED_OFFICIAL_MARKETPLACE_NAMES` 改为从远程配置（如 GrowthBook 或 bootstrap API）获取的缓存列表，减少新增官方市场对客户端发版的依赖。

3. **用类型系统强制双向映射一致性**
   - 当前 `SCOPE_TO_EDITABLE_SOURCE` 是手动定义的 `Record`。可改用 TypeScript 的映射推导，从 `SETTING_SOURCE_TO_SCOPE` 自动生成反向映射的类型和运行时对象，避免 drift。

4. **`buildPluginId` 增加前置校验**
   - 在拼接前检查 `name` 是否为空、是否包含非法字符（如空格、`@`），提前抛出带有明确错误信息的异常，而不是让非法 ID 流向后方的 schema / 文件系统操作。

5. **考虑引入 PluginId  branded type**
   - 将插件 ID 从普通 `string` 提升为 branded type（如 `type PluginId = string & { __brand: 'PluginId' }`），由 `parsePluginIdentifier` + `buildPluginId` 负责生成，强制调用方在类型层面区分“任意字符串”和“已验证的插件 ID”。
