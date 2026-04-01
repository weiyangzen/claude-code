# Research Document: src/utils/settings/types.ts

## 场景与职责

`src/utils/settings/types.ts` 是 Claude Code 设置系统的**核心类型与 Schema 定义源文件**，承担以下职责：

1. **统一定义用户可配置的 settings.json 结构**（`SettingsSchema`），涵盖个人设置、项目设置、本地设置、策略设置（managed/enterprise）以及 CLI flag 设置。
2. **提供强类型推导**：通过 Zod schema 反向推导 TypeScript 类型（`SettingsJson`、`AllowedMcpServerEntry`、`DeniedMcpServerEntry` 等），供整个代码库消费。
3. **维护向后兼容性**：文件顶部有显式的 BACKWARD COMPATIBILITY NOTICE，规定允许/禁止的 schema 变更，并指引开发者运行 `test/utils/settings/backward-compatibility.test.ts`。
4. **支持企业级策略字段**：如 `strictPluginOnlyCustomization`、`allowedMcpServers`、`deniedMcpServers`、`strictKnownMarketplaces`、`blockedMarketplaces`、`channelsEnabled` 等。
5. **集成特性开关（feature flags）**：大量字段通过 `feature('...')` 或 `process.env.USER_TYPE === 'ant'` 条件控制是否出现在 schema 中，防止外部构建泄露未发布功能。

## 功能点目的

### 1. SettingsSchema — 统一设置 Schema
- 使用 `lazySchema` 延迟构造，避免模块初始化时立即构建庞大的 Zod 对象，减少启动开销。
- 通过 `.passthrough()` 保留未知字段，确保：
  - 旧客户端遇到新字段不会直接报错；
  - 更新设置时不会误删用户手动写入但当前版本不认识的键。
- 大量字段使用 `.optional()` 和 `.describe()`，既兼容缺失值，又为 JSON Schema 生成提供文档。

### 2. 权限相关子 Schema
- `PermissionsSchema`：定义 `allow`/`deny`/`ask` 规则数组、`defaultMode` 枚举、`additionalDirectories` 等。
- `PermissionRuleSchema` 实际定义在 `permissionValidation.ts`，通过 `lazySchema` + `superRefine` 对每条规则做自定义语法校验（如括号匹配、工具名大写、MCP 规则格式等）。

### 3. MCP 企业管理字段
- `AllowedMcpServerEntrySchema` / `DeniedMcpServerEntrySchema`：
  - 支持按 `serverName`、`serverCommand`、`serverUrl` 三种互斥方式匹配；
  - 通过 `.refine()` 强制要求三者必须且只能出现一个；
  - 使用 `count(..., Boolean)` 工具函数计算已定义字段数。

### 4. Marketplace 与插件配置
- `ExtraKnownMarketplaceSchema`：定义额外的 marketplace 源（`source`、`installLocation`、`autoUpdate`）。
- `extraKnownMarketplaces`：record 类型，key 为 marketplace 名称；通过 `.check()` 自定义校验确保 `source.name` 与 record key 一致，防止调和器（reconciler）无限循环。
- `strictKnownMarketplaces` / `blockedMarketplaces`：企业策略字段，控制 marketplace 白名单/黑名单。
- `pluginConfigs`：按插件 ID 存储的 MCP server 用户配置与非敏感选项。

### 5. 特性门控字段
- `xaaIdp`：仅在 `CLAUDE_CODE_ENABLE_XAA` 环境变量为真时出现（SEP-990 XAA 身份提供商配置）。
- `classifierPermissionsEnabled`、`minSleepDurationMs`、`maxSleepDurationMs`：仅对 `USER_TYPE === 'ant'` 或特定 feature flag 开启。
- `voiceEnabled`、`assistant`、`assistantName`、`defaultView`：分别受 `VOICE_MODE`、`KAIROS`、`KAIROS_BRIEF` 等 feature flag 控制。

### 6. `strictPluginOnlyCustomization` 的前向兼容预处理
- 使用 `.preprocess()` 过滤掉不在 `CUSTOMIZATION_SURFACES`（`skills`、`agents`、`hooks`、`mcp`）中的未知值；
- 配合 `.catch(undefined)`，当传入完全非法值时降级为 `undefined`（即不锁定任何 surface），避免整个 managed-settings 文件被置空。

### 7. 内部类型补充
- `PluginHookMatcher`、`SkillHookMatcher`：供插件/技能系统内部使用，附加 `pluginRoot`/`skillRoot` 等上下文。
- `UserConfigValues`、`PluginConfig`：插件配置值的类型别名。
- 三个 MCP entry type guards：`isMcpServerNameEntry`、`isMcpServerCommandEntry`、`isMcpServerUrlEntry`。

## 具体技术实现（关键流程/数据结构/协议/命令）

### lazySchema 延迟构造
```ts
export function lazySchema<T>(factory: () => T): () => T {
  let cached: T | undefined
  return () => (cached ??= factory())
}
```
所有 schema 均以工厂函数形式导出（如 `SettingsSchema` 是 `() => z.object(...)`），首次调用时缓存结果。这在大型 schema 依赖链中显著降低模块加载时 CPU 消耗。

### feature flag 条件展开
```ts
...(feature('TRANSCRIPT_CLASSIFIER')
  ? {
      disableAutoMode: z.enum(['disable']).optional().describe('Disable auto mode'),
    }
  : {}),
```
利用对象展开语法，在 schema 构造时动态决定字段是否存在。注意：这会影响 TypeScript 推导类型；外部构建中未启用的字段不会出现在 `SettingsJson` 类型中。

### `extraKnownMarketplaces` 自定义 check
```ts
.check(ctx => {
  for (const [key, entry] of Object.entries(ctx.value)) {
    if (entry.source.source === 'settings' && entry.source.name !== key) {
      ctx.issues.push({
        code: 'custom',
        input: entry.source.name,
        path: [key, 'source', 'name'],
        message: `Settings-sourced marketplace name must match its extraKnownMarketplaces key ...`,
      })
    }
  }
})
```
该 check 直接阻止因 key/name 不一致导致的 reconciler 抖动：每次会话都会误判为 missing → 触发无意义缓存清除。

### `strictPluginOnlyCustomization` 的降级设计
```ts
.preprocess(
  v => Array.isArray(v)
    ? v.filter(x => (CUSTOMIZATION_SURFACES as readonly string[]).includes(x))
    : v,
  z.union([z.boolean(), z.array(z.enum(CUSTOMIZATION_SURFACES))]),
)
.catch(undefined)
```
- **preprocess**：旧客户端遇到未来新增 surface（如 `"commands"`）时，安全地过滤掉不认识项，保留已知项。
- **catch(undefined)**：如果用户传入字符串 `"skills"` 或对象，preprocess 无法转换，union 校验失败；`.catch(undefined)` 确保不会 null 掉整个 managed-settings 文件。

### Hooks 的解耦与再导出
文件通过 `export { ... } from '../../schemas/hooks.js'` 和 `import { ... } from '../../schemas/hooks.js'` 双向引用，既保持 `types.ts` 作为对外暴露 hooks 类型的统一入口，又将实际 schema 定义下沉到 `src/schemas/hooks.ts`，打破与 `plugins/schemas.ts` 的循环依赖。

## 关键代码路径与文件引用

### 直接依赖（被导入）
| 文件 | 用途 |
|------|------|
| `bun:bundle` (`feature`) | 特性开关判断 |
| `zod/v4` | Schema 定义核心库 |
| `src/entrypoints/sandboxTypes.js` | `SandboxSettingsSchema` |
| `src/utils/envUtils.js` (`isEnvTruthy`) | 环境变量判断 |
| `src/utils/lazySchema.js` | 延迟 schema 构造 |
| `src/utils/permissions/PermissionMode.js` | `PERMISSION_MODES` / `EXTERNAL_PERMISSION_MODES` |
| `src/utils/plugins/schemas.js` | `MarketplaceSourceSchema` |
| `src/utils/settings/constants.js` | `CLAUDE_CODE_SETTINGS_SCHEMA_URL` |
| `src/utils/settings/permissionValidation.js` | `PermissionRuleSchema` |
| `src/schemas/hooks.js` | Hooks schema 与类型 |
| `src/utils/array.js` (`count`) | 数组计数工具 |

### 被调用方（导入本文件）
| 文件 | 用途 |
|------|------|
| `src/utils/settings/settings.ts` | 核心设置加载、合并、缓存逻辑 |
| `src/utils/settings/validation.ts` | `SettingsSchema` 用于文件内容校验 |
| `src/utils/settings/schemaOutput.ts` | 生成 JSON Schema 文档 |
| `src/utils/settings/mdm/settings.ts` | MDM/企业策略设置解析 |
| `src/utils/settings/pluginOnlyPolicy.ts` | 读取 `CUSTOMIZATION_SURFACES` 与 `strictPluginOnlyCustomization` |
| `src/bootstrap/state.ts` | 可能消费 `SettingsJson` 类型 |
| `src/services/mcp/config.ts` | MCP 配置读取 |
| `src/components/TrustDialog/utils.ts` | 信任对话框使用设置类型 |
| `src/skills/loadSkillsDir.ts` | 技能加载可能读取设置 |
| `src/tools/AgentTool/loadAgentsDir.ts` | Agent 目录加载 |
| `src/hooks/useSettingsChange.ts` | React hook 监听设置变更 |
| `src/state/AppStateStore.ts` | 全局状态存储 |

## 依赖与外部交互

### Zod v4
- 使用 `z.object()`、`z.array()`、`z.enum()`、`z.record()`、`z.union()`、`z.discriminatedUnion()`、`z.preprocess()`、`z.coerce.string()`、`z.check()`、`z.refine()`、`z.passthrough()`、`z.strict()`、`z.catch()`、`z.partialRecord()`、`z.superRefine()` 等高级 API。
- 通过 `z.infer<ReturnType<typeof SettingsSchema>>` 推导 `SettingsJson`。

### bun:bundle feature 系统
- `feature('TRANSCRIPT_CLASSIFIER')`、`feature('LODESTONE')`、`feature('PROACTIVE')`、`feature('KAIROS')`、`feature('VOICE_MODE')`、`feature('KAIROS_BRIEF')` 等。
- 这些在构建时由 bundler 做死代码消除，未启用 feature 的字段不会进入外部产物。

### 环境变量
- `process.env.CLAUDE_CODE_ENABLE_XAA`：控制 `xaaIdp` 字段。
- `process.env.USER_TYPE === 'ant'`：控制 ant 内部功能（如 `classifierPermissionsEnabled`、`effortLevel` 的 `max` 选项、`autoMode.deny` 别名）。

### 设置文件格式
- 最终用户写入的是标准 JSON（`.claude/settings.json`、`.claude/settings.local.json`、`~/.claude/settings.json`、managed-settings.json 等）。
- `SettingsSchema` 本身不直接读写文件，只负责内存对象的校验与类型推导。

## 风险、边界与改进建议

### 风险

1. **向后兼容性破坏**
   - 虽然文件顶部有显式注释和测试指引，但 `SettingsSchema` 非常庞大（约 80+ 字段），新增字段若忘记 `.optional()` 或修改已有字段类型，可能导致大量用户设置文件失效。
   - `test/utils/settings/backward-compatibility.test.ts` 被引用为守护测试，但本次调研未在仓库中找到该测试文件，需确认是否已迁移或遗漏。

2. **feature flag 导致的类型碎片化**
   - 同一字段在不同构建产物中可能存在/不存在，导致 SDK 或文档生成器（如 `schemaOutput.ts` 的 `toJSONSchema`）在 ant 构建与外部构建之间输出不一致。
   - `generateSettingsJSONSchema` 运行在 ant 环境时可能包含外部用户看不到的字段，反之亦然。

3. **`extraKnownMarketplaces` 的 `.check()` 与 `.passthrough()` 交互**
   - `.check()` 在 Zod v4 中运行于解析阶段之后，但 `.passthrough()` 会保留未知字段；如果未来 `MarketplaceSourceSchema` 也使用 `.passthrough()`，check 逻辑可能需要同步更新。

4. **`strictPluginOnlyCustomization` 的 catch(undefined) 静默降级**
   - 用户若误写 `"skills"`（字符串）而非 `["skills"]`，字段会被静默丢弃为 `undefined`，用户可能困惑为何锁定未生效。当前由 "Doctor" 工具标记原始值，但 Doctor 是否覆盖所有场景存疑。

5. **MCP entry refine 的互斥校验**
   - `AllowedMcpServerEntrySchema` 和 `DeniedMcpServerEntrySchema` 的 refine 逻辑几乎完全一致，存在代码重复；未来若增加第四种匹配维度，需要同时修改两处。

### 边界

- **不处理文件 I/O**：`types.ts` 只定义 schema 和类型，文件读取、合并、缓存完全下沉到 `settings.ts` 和 `mdm/settings.ts`。
- **不处理 UI 渲染**：错误提示文本、文档链接由 `validationTips.ts` 和 `validation.ts` 负责。
- **不处理权限决策**：`PermissionRuleSchema` 的语法校验在 `permissionValidation.ts`，而权限匹配引擎在 `src/utils/permissions/filesystem.ts`。

### 改进建议

1. **提取共享的 MCP entry refine 逻辑**
   - 将 "exactly one of serverName/serverCommand/serverUrl" 的 refine 提取为可复用工厂函数，减少重复代码。

2. **为 feature flag 字段增加运行时文档注释**
   - 在生成 JSON Schema 时，可考虑标注哪些字段是 "ant-only" 或 "feature-gated"，帮助外部贡献者理解字段可见性。

3. **增强 `strictPluginOnlyCustomization` 的无效值反馈**
   - 与其完全 `.catch(undefined)`，可考虑增加一个 `.superRefine()` 在值为字符串时给出更明确的错误信息（"Expected array or boolean, received string"），同时仍不 null 整个文件。

4. **Schema 拆分**
   - `SettingsSchema` 已超过 800 行，可考虑按功能域拆分为子 schema 文件（如 `modelSettingsSchema.ts`、`mcpSettingsSchema.ts`、`sandboxSettingsSchema.ts`），由 `types.ts` 组合导入，降低单文件维护成本。

5. **类型测试补齐**
   - 若 backward-compatibility 测试确实缺失，建议尽快补齐：收集历史版本的典型 settings.json 样本，确保它们在新 schema 下仍能 `safeParse` 成功。
