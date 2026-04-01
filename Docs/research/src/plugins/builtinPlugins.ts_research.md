# builtinPlugins.ts 深度研究文档

> 研究对象: `src/plugins/builtinPlugins.ts`  
> 研究范围: 代码实现、类型定义、调用依赖、配置交互  
> 执行器: kimi (k2p5)  
> 生成时间: 2026-04-01

---

## 1. 场景与职责

### 1.1 模块定位

`builtinPlugins.ts` 是 Claude Code CLI 的**内置插件注册中心**，负责管理随 CLI 一起分发的内置插件。它与捆绑技能 (`src/skills/bundled/`) 的区别在于：

| 特性 | Built-in Plugins | Bundled Skills |
|------|------------------|----------------|
| UI 可见性 | 在 `/plugin` UI 中显示为 "Built-in" 分区 | 不显示在插件列表中 |
| 用户控制 | 用户可通过 `/plugin` 启用/禁用 | 始终启用，不可禁用 |
| 持久化 | 状态保存到用户设置 (`settings.json`) | 无持久化状态 |
| 组件能力 | 可提供 skills、hooks、MCP servers | 仅提供 skills |
| ID 格式 | `{name}@builtin` | 无 ID 概念 |

### 1.2 核心职责

1. **插件注册**: 提供 `registerBuiltinPlugin()` 供初始化代码注册内置插件
2. **状态管理**: 根据用户设置和插件默认值计算启用/禁用状态
3. **技能暴露**: 将已启用内置插件的技能转换为 `Command` 对象供模型使用
4. **标识识别**: 通过 `isBuiltinPluginId()` 识别内置插件 ID (`@builtin` 后缀)

### 1.3 使用场景

```
CLI 启动流程:
1. initBuiltinPlugins() (src/plugins/bundled/index.ts)
   └── registerBuiltinPlugin({...})  // 注册各个内置插件

2. 用户交互:
   /plugin 命令 → 显示内置插件列表 → 用户启用/禁用
   └── 状态保存到 settings.json 的 enabledPlugins 字段

3. 技能加载 (commands.ts):
   getBuiltinPluginSkillCommands() → 获取已启用插件的技能
   └── 合并到整体技能列表供模型使用
```

---

## 2. 功能点目的

### 2.1 功能清单

| 函数 | 目的 | 调用时机 |
|------|------|----------|
| `registerBuiltinPlugin()` | 注册内置插件定义到内存 Map | CLI 启动时 (`initBuiltinPlugins`) |
| `isBuiltinPluginId()` | 判断插件 ID 是否为内置格式 | 插件操作、启用/禁用逻辑 |
| `getBuiltinPluginDefinition()` | 按名称获取插件定义 | `/plugin` UI 展示详情 |
| `getBuiltinPlugins()` | 获取所有内置插件的启用/禁用状态 | UI 展示、技能加载 |
| `getBuiltinPluginSkillCommands()` | 获取已启用插件的技能命令 | 每次获取技能列表时 |
| `clearBuiltinPlugins()` | 清空注册表 (测试用) | 测试清理 |

### 2.2 启用状态决策逻辑

```
启用状态 = 用户设置 > 插件默认值 > true

具体逻辑 (getBuiltinPlugins 函数):
1. 读取用户设置: settings?.enabledPlugins?.[`${name}@builtin`]
2. 如果用户明确设置: 使用用户设置值
3. 如果用户未设置: 使用 definition.defaultEnabled ?? true
4. 如果插件定义了 isAvailable() 且返回 false: 完全隐藏该插件
```

### 2.3 与 Marketplace 插件的对比

| 维度 | Built-in Plugins | Marketplace Plugins |
|------|------------------|---------------------|
| 来源 | 编译进 CLI 二进制 | 从 marketplace 下载 |
| 安装 | 无需安装，随 CLI 分发 | `claude plugin install` |
| 存储路径 | 无 (内存中) | `~/.claude/plugins/cache/` |
| 版本控制 | 跟随 CLI 版本 | 独立版本管理 |
| 启用配置 | `enabledPlugins["name@builtin"]` | `enabledPlugins["name@marketplace"]` |
| 卸载 | 只能禁用，不能卸载 | 可以完整卸载 |

---

## 3. 具体技术实现

### 3.1 数据结构

#### 3.1.1 内置插件定义 (BuiltinPluginDefinition)

```typescript
// src/types/plugin.ts:18-35
type BuiltinPluginDefinition = {
  name: string                    // 插件名称 (用于构建 {name}@builtin ID)
  description: string             // 在 /plugin UI 中显示的描述
  version?: string               // 可选版本字符串
  skills?: BundledSkillDefinition[]  // 提供的技能
  hooks?: HooksSettings          // 提供的 hooks
  mcpServers?: Record<string, McpServerConfig>  // 提供的 MCP 服务器
  isAvailable?: () => boolean    // 可用性检查 (如系统能力检测)
  defaultEnabled?: boolean       // 默认启用状态 (默认 true)
}
```

#### 3.1.2 内部存储结构

```typescript
// builtinPlugins.ts:21
const BUILTIN_PLUGINS: Map<string, BuiltinPluginDefinition> = new Map()

// Key: plugin name (不含 @builtin 后缀)
// Value: BuiltinPluginDefinition 对象
```

#### 3.1.3 加载后的插件对象 (LoadedPlugin)

```typescript
// src/types/plugin.ts:48-70
type LoadedPlugin = {
  name: string
  manifest: PluginManifest
  path: string              // 内置插件固定为 'builtin' (sentinel)
  source: string            // 完整插件 ID: "name@builtin"
  repository: string        // 同上
  enabled?: boolean
  isBuiltin?: boolean       // true 表示内置插件
  hooksConfig?: HooksSettings
  mcpServers?: Record<string, McpServerConfig>
  // ... 其他 marketplace 插件特有的字段为空
}
```

### 3.2 关键流程

#### 3.2.1 注册流程

```
registerBuiltinPlugin(definition)
  └── BUILTIN_PLUGINS.set(definition.name, definition)
```

**特点**: 
- 简单的内存 Map 写入
- 无持久化操作
- 重复注册会覆盖 (后注册者胜)

#### 3.2.2 获取插件列表流程

```
getBuiltinPlugins()
  ├── 读取用户设置: getSettings_DEPRECATED()
  ├── 遍历 BUILTIN_PLUGINS Map
  │   ├── 检查 isAvailable(): 不可用则跳过
  │   ├── 构建 pluginId: `${name}@builtin`
  │   ├── 查询用户设置: settings?.enabledPlugins?.[pluginId]
  │   ├── 计算 isEnabled: userSetting ?? defaultEnabled ?? true
  │   └── 构建 LoadedPlugin 对象
  │       ├── manifest: { name, description, version }
  │       ├── path: 'builtin' (sentinel)
  │       ├── source/repository: pluginId
  │       ├── isBuiltin: true
  │       ├── hooksConfig: definition.hooks
  │       └── mcpServers: definition.mcpServers
  └── 返回 { enabled: LoadedPlugin[], disabled: LoadedPlugin[] }
```

#### 3.2.3 技能命令转换流程

```
getBuiltinPluginSkillCommands()
  ├── 调用 getBuiltinPlugins() 获取已启用插件
  ├── 遍历 enabled 插件
  │   ├── 获取插件定义: BUILTIN_PLUGINS.get(plugin.name)
  │   ├── 遍历 definition.skills
  │   └── 调用 skillDefinitionToCommand(skill) 转换为 Command
  └── 返回 Command[]

skillDefinitionToCommand(definition)
  └── Command 对象:
      ├── type: 'prompt'
      ├── source: 'bundled' (注释说明: 'builtin' 表示硬编码命令)
      ├── loadedFrom: 'bundled'
      ├── isHidden: !userInvocable
      └── getPromptForCommand: definition.getPromptForCommand
```

### 3.3 协议与约定

#### 3.3.1 插件 ID 格式

```typescript
// 内置插件 ID 格式
const pluginId = `${name}@builtin`

// 示例
"git@builtin"
"github@builtin"
"memory@builtin"
```

#### 3.3.2 路径 Sentinel

内置插件的 `path` 字段固定为 `'builtin'`，用于：
- 标识无文件系统路径 (区别于 marketplace 插件的缓存路径)
- UI 展示时识别为内置插件
- 避免执行文件系统操作

#### 3.3.3 Source 字段约定

在 `Command` 对象中：
- `source: 'builtin'` → 硬编码的 slash 命令 (`/help`, `/clear` 等)
- `source: 'bundled'` → 内置插件提供的技能 (用于 SkillTool 列表、分析日志、截断豁免)
- `loadedFrom: 'bundled'` → 标识来源为捆绑/内置

**设计意图** (来自代码注释):
> `'bundled' not 'builtin' — 'builtin' in Command.source means hardcoded slash commands (/help, /clear). Using 'bundled' keeps these skills in the Skill tool's listing, analytics name logging, and prompt-truncation exemption.

---

## 4. 关键代码路径与文件引用

### 4.1 核心文件依赖图

```
src/plugins/builtinPlugins.ts
├── 导入类型
│   ├── ../commands.ts:Command
│   ├── ../skills/bundledSkills.ts:BundledSkillDefinition
│   ├── ../types/plugin.ts:BuiltinPluginDefinition, LoadedPlugin
│   └── ../utils/settings/settings.ts:getSettings_DEPRECATED
├── 被调用方
│   ├── src/plugins/bundled/index.ts:initBuiltinPlugins()
│   │   └── registerBuiltinPlugin()  // 注册入口
│   ├── src/commands.ts
│   │   ├── getBuiltinPluginSkillCommands()  // 技能加载
│   │   └── getBuiltinPlugins() (间接)
│   ├── src/services/plugins/pluginOperations.ts
│   │   └── isBuiltinPluginId()  // 启用/禁用操作
│   └── src/utils/plugins/pluginLoader.ts
│       └── BUILTIN_MARKETPLACE_NAME  // 插件加载
└── 常量导出
    └── BUILTIN_MARKETPLACE_NAME = 'builtin'
```

### 4.2 关键代码路径详解

#### 路径 1: CLI 启动注册

```typescript
// src/plugins/bundled/index.ts:20
export function initBuiltinPlugins(): void {
  // No built-in plugins registered yet — this is the scaffolding for
  // migrating bundled skills that should be user-toggleable.
}
```

**现状**: 当前无实际注册的内置插件，该文件作为脚手架存在，用于将来将捆绑技能迁移为用户可切换的内置插件。

#### 路径 2: 技能加载集成

```typescript
// src/commands.ts:353-398
async function getSkills(cwd: string): Promise<{
  skillDirCommands: Command[]
  pluginSkills: Command[]
  bundledSkills: Command[]
  builtinPluginSkills: Command[]  // ← 内置插件技能
}> {
  // ...
  const builtinPluginSkills = getBuiltinPluginSkillCommands()
  // ...
}

// src/commands.ts:449-469
const loadAllCommands = memoize(async (cwd: string): Promise<Command[]> => {
  const [
    { skillDirCommands, pluginSkills, bundledSkills, builtinPluginSkills },
    // ...
  ] = await Promise.all([
    getSkills(cwd),
    // ...
  ])

  return [
    ...bundledSkills,
    ...builtinPluginSkills,  // ← 插入位置
    ...skillDirCommands,
    // ...
  ]
})
```

**加载优先级**: bundledSkills → builtinPluginSkills → skillDirCommands → workflowCommands → pluginCommands → COMMANDS

#### 路径 3: 启用/禁用操作

```typescript
// src/services/plugins/pluginOperations.ts:573-604
export async function setPluginEnabledOp(
  plugin: string,
  enabled: boolean,
  scope?: InstallableScope,
): Promise<PluginOperationResult> {
  // Built-in plugins: always use user-scope settings, bypass the normal
  // scope-resolution + installed_plugins lookup (they're not installed).
  if (isBuiltinPluginId(plugin)) {
    const { error } = updateSettingsForSource('userSettings', {
      enabledPlugins: {
        ...getSettingsForSource('userSettings')?.enabledPlugins,
        [plugin]: enabled,
      },
    })
    // ...
  }
  // ...
}
```

**特殊处理**: 内置插件始终使用 `userSettings` 作用域，绕过 marketplace 插件的作用域解析和 `installed_plugins.json` 查找逻辑。

### 4.3 配置文件交互

#### 4.3.1 设置文件位置

```typescript
// 用户设置 (内置插件状态保存位置)
~/.claude/settings.json

// 结构示例
{
  "enabledPlugins": {
    "git@builtin": true,
    "memory@builtin": false
  }
}
```

#### 4.3.2 设置 Schema

```typescript
// src/utils/settings/types.ts:559-567
enabledPlugins: z
  .record(
    z.string(),
    z.union([z.array(z.string()), z.boolean(), z.undefined()]),
  )
  .optional()
  .describe(
    'Enabled plugins using plugin-id@marketplace-id format. Example: { "formatter@anthropic-tools": true }'
  )
```

---

## 5. 依赖与外部交互

### 5.1 直接依赖

| 模块路径 | 导入内容 | 用途 |
|---------|---------|------|
| `../commands.js` | `Command` 类型 | 技能转换的目标类型 |
| `../skills/bundledSkills.js` | `BundledSkillDefinition` | 内置插件技能的定义类型 |
| `../types/plugin.js` | `BuiltinPluginDefinition`, `LoadedPlugin` | 插件类型定义 |
| `../utils/settings/settings.js` | `getSettings_DEPRECATED` | 读取用户设置 |

### 5.2 间接依赖 (调用链)

| 模块路径 | 关系 | 说明 |
|---------|------|------|
| `src/plugins/bundled/index.ts` | 被调用方 | 初始化时注册内置插件 |
| `src/commands.ts` | 被调用方 | 获取技能列表 |
| `src/services/plugins/pluginOperations.ts` | 被调用方 | 启用/禁用操作 |
| `src/utils/plugins/pluginLoader.ts` | 被调用方 | 插件加载时识别内置插件 |

### 5.3 类型依赖详解

#### 5.3.1 BundledSkillDefinition

```typescript
// src/skills/bundledSkills.ts:15-41
type BundledSkillDefinition = {
  name: string
  description: string
  aliases?: string[]
  whenToUse?: string
  argumentHint?: string
  allowedTools?: string[]
  model?: string
  disableModelInvocation?: boolean
  userInvocable?: boolean
  isEnabled?: () => boolean
  hooks?: HooksSettings
  context?: 'inline' | 'fork'
  agent?: string
  files?: Record<string, string>
  getPromptForCommand: (args: string, context: ToolUseContext) => Promise<ContentBlockParam[]>
}
```

#### 5.3.2 HooksSettings

```typescript
// src/schemas/hooks.ts
// 支持的事件类型: PreToolUse, PostToolUse, Notification, UserPromptSubmit, 
//                SessionStart, SessionEnd, Stop, SubagentStop, PreCompact, 
//                PostCompact, TeammateIdle, TaskCreated, TaskCompleted
```

#### 5.3.3 McpServerConfig

```typescript
// src/services/mcp/types.ts:124-135
// 支持类型: stdio, sse, sse-ide, ws-ide, http, ws, sdk, claudeai-proxy
```

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 风险 1: 命名冲突

```typescript
// 内置插件名称与 marketplace 插件可能冲突
const pluginId = `${name}@builtin`  // 通过 @builtin 后缀区分
const marketplaceId = `${name}@some-marketplace`

// 问题: 如果用户有 marketplace 插件也叫 "git"，
// 在 UI 中可能产生混淆
```

**缓解**: 内置插件应使用具有辨识度的名称，避免与热门 marketplace 插件冲突。

#### 风险 2: 状态持久化位置

```typescript
// 内置插件状态保存在用户设置中
// 如果用户删除 settings.json，内置插件状态会重置为默认值

// 这不是严重问题，因为默认通常是 true
// 但如果插件有安全敏感功能，可能需要更严格的默认
```

#### 风险 3: 无版本管理

```typescript
// 内置插件版本跟随 CLI 版本
// 无法独立更新单个内置插件
// 用户无法回退到特定版本的内置插件
```

### 6.2 边界情况

#### 边界 1: isAvailable() 返回 false

```typescript
// builtinPlugins.ts:66-68
if (definition.isAvailable && !definition.isAvailable()) {
  continue  // 完全跳过，不出现在 enabled/disabled 列表中
}
```

**行为**: 插件完全隐藏，用户无法在 `/plugin` UI 中看到或启用它。

#### 边界 2: 空技能列表

```typescript
// getBuiltinPluginSkillCommands()
if (!definition?.skills) continue  // 无技能的插件被跳过
```

**行为**: 不提供技能的插件不会贡献任何 Command。

#### 边界 3: 并发修改

```typescript
// clearBuiltinPlugins() 用于测试
export function clearBuiltinPlugins(): void {
  BUILTIN_PLUGINS.clear()
}
```

**风险**: 生产代码不应调用此函数，可能导致运行时技能列表异常。

### 6.3 改进建议

#### 建议 1: 添加内置插件元数据

```typescript
// 当前 BuiltinPluginDefinition 缺少一些有用的元数据
// 建议添加:
type BuiltinPluginDefinition = {
  // ... 现有字段
  category?: 'core' | 'integration' | 'productivity'  // UI 分组
  tags?: string[]  // 搜索标签
  documentationUrl?: string  // 文档链接
  icon?: string  // UI 图标标识
}
```

#### 建议 2: 支持内置插件配置

```typescript
// 当前内置插件无法接收用户配置
// 建议支持:
type BuiltinPluginDefinition = {
  // ... 现有字段
  configSchema?: z.ZodSchema  // 配置验证 schema
  defaultConfig?: Record<string, unknown>  // 默认配置
}
```

#### 建议 3: 改进版本可见性

```typescript
// 当前内置插件版本信息在 UI 中展示不充分
// 建议:
// 1. 在 LoadedPlugin.manifest 中明确 CLI 版本
// 2. 在 /plugin UI 中显示内置插件对应的 CLI 版本
```

#### 建议 4: 懒加载支持

```typescript
// 当前所有内置插件在启动时注册
// 对于大量内置插件，建议支持:
type BuiltinPluginDefinition = {
  // ... 现有字段
  lazy?: boolean  // 延迟加载直到首次使用
  load?: () => Promise<Partial<BuiltinPluginDefinition>>  // 动态加载
}
```

#### 建议 5: 增强测试覆盖

```typescript
// 当前无直接测试文件
// 建议添加测试:
// - builtinPlugins.test.ts
//   - registerBuiltinPlugin 测试
//   - getBuiltinPlugins 状态计算测试
//   - getBuiltinPluginSkillCommands 转换测试
//   - isBuiltinPluginId 边界测试
```

### 6.4 架构演进方向

根据代码注释和当前实现，内置插件系统的演进方向可能是：

```
当前状态:
  bundled/index.ts (空实现) → 脚手架状态

短期演进:
  将 src/skills/bundled/*.ts 中的技能迁移为内置插件
  使用 registerBuiltinPlugin() 注册
  保持现有功能，增加用户可切换能力

长期演进:
  内置插件可能支持从远程 marketplace 更新
  (类似 Chrome 内置扩展的更新机制)
```

---

## 7. 附录

### 7.1 相关文件清单

| 文件路径 | 作用 |
|---------|------|
| `src/plugins/builtinPlugins.ts` | 本研究对象，内置插件注册中心 |
| `src/plugins/bundled/index.ts` | 内置插件初始化入口 |
| `src/types/plugin.ts` | 插件类型定义 |
| `src/skills/bundledSkills.ts` | 捆绑技能定义和注册 |
| `src/commands.ts` | 命令/技能加载和聚合 |
| `src/services/plugins/pluginOperations.ts` | 插件操作 (启用/禁用) |
| `src/utils/plugins/pluginLoader.ts` | 插件加载器 |
| `src/utils/settings/settings.ts` | 设置读写 |
| `src/utils/settings/types.ts` | 设置 Schema |

### 7.2 关键常量

```typescript
// builtinPlugins.ts:23
export const BUILTIN_MARKETPLACE_NAME = 'builtin'

// 用于构建插件 ID: `${name}@${BUILTIN_MARKETPLACE_NAME}`
```

### 7.3 调试信息

```typescript
// 启用调试日志可观察内置插件加载情况
// 环境变量: DEBUG=1

// 相关日志输出:
// - "getSkills returning: X skill dir commands, Y plugin skills, Z bundled skills, W builtin plugin skills"
// - "Loaded N skills from plugin {name} default directory"
```

---

*文档结束*
