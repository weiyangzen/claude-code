# updateConfig.ts 深度研究文档

## 场景与职责

`updateConfig.ts` 是 Claude Code CLI 的**核心配置管理技能模块**，负责实现 `/update-config` 命令技能。该技能允许用户通过自然语言交互来修改 Claude Code 的配置文件（`settings.json`），包括权限规则、环境变量、Hooks 配置等。

### 核心职责
1. **配置更新技能注册**：向系统注册 `update-config`  bundled skill
2. **动态 JSON Schema 生成**：从 Zod schema 实时生成 Settings JSON Schema，确保技能提示与类型定义同步
3. **Hooks 配置指导**：提供完整的 hooks 配置文档和验证流程
4. **配置合并策略**：指导如何安全地合并新配置到现有配置文件

### 适用场景
- 用户想要添加权限规则（如"允许 npm 命令"）
- 用户需要配置自动化行为（如"保存后自动格式化"）
- 用户需要设置环境变量
- 用户需要排查 hooks 不工作的问题

---

## 功能点目的

### 1. 动态 Settings Schema 生成
```typescript
function generateSettingsSchema(): string {
  const jsonSchema = toJSONSchema(SettingsSchema(), { io: 'input' })
  return jsonStringify(jsonSchema, null, 2)
}
```
**目的**：确保技能提示中的配置文档始终与实际的 TypeScript 类型定义保持同步，避免文档过时。

### 2. 三层配置文档
- **SETTINGS_EXAMPLES_DOCS**：配置文件位置、权限规则语法、环境变量、模型设置等基础配置
- **HOOKS_DOCS**：Hooks 结构、事件类型、Hook 类型（command/prompt/agent/http）的详细说明
- **HOOK_VERIFICATION_FLOW**：完整的 Hook 构建和验证流程（7 步骤）

### 3. 双模式提示生成
- **完整模式**：提供完整的配置更新指南 + 动态生成的 JSON Schema
- **Hooks-only 模式**：当参数以 `[hooks-only]` 开头时，仅返回 Hooks 相关文档

---

## 具体技术实现

### 关键流程

#### 1. Skill 注册流程
```typescript
export function registerUpdateConfigSkill(): void {
  registerBundledSkill({
    name: 'update-config',
    description: 'Use this skill to configure the Claude Code harness via settings.json...',
    allowedTools: ['Read'],
    userInvocable: true,
    async getPromptForCommand(args) {
      // 根据参数决定返回完整提示或 hooks-only 提示
    },
  })
}
```

#### 2. 提示生成逻辑
```
if args startsWith '[hooks-only]':
  return HOOKS_DOCS + HOOK_VERIFICATION_FLOW + optional task
else:
  generateSettingsSchema() dynamically
  return UPDATE_CONFIG_PROMPT + JSON_SCHEMA + optional user request
```

#### 3. Hook 验证流程（7 步骤）
1. **Dedup check**：检查是否已存在相同 event+matcher 的 hook
2. **Construct command**：构建接收 stdin JSON 的命令
3. **Pipe-test**：使用模拟的 stdin payload 测试命令
4. **Write JSON**：合并到目标配置文件
5. **Validate syntax**：使用 `jq -e` 验证 JSON 语法和 schema
6. **Prove hook fires**：通过实际触发工具验证 hook 是否生效
7. **Handoff**：告知用户 hook 已生效或需要重启

### 数据结构

#### BundledSkillDefinition（来自 bundledSkills.ts）
```typescript
type BundledSkillDefinition = {
  name: string
  description: string
  aliases?: string[]
  allowedTools?: string[]
  userInvocable?: boolean
  files?: Record<string, string>  // 参考文件，首次调用时提取到磁盘
  getPromptForCommand: (args: string, context: ToolUseContext) => Promise<ContentBlockParam[]>
}
```

#### SettingsSchema（来自 utils/settings/types.ts）
包含 100+ 个配置项的 Zod schema，关键字段：
- `permissions`: 权限规则（allow/deny/ask/defaultMode）
- `env`: 环境变量
- `hooks`: Hooks 配置
- `model`: 默认模型
- `enabledPlugins`: 启用的插件
- `mcpServers`: MCP 服务器配置

### 关键代码路径

| 功能 | 文件路径 |
|------|----------|
| Skill 注册入口 | `src/skills/bundled/updateConfig.ts:445-475` |
| Schema 生成 | `src/skills/bundled/updateConfig.ts:10-13` |
| 提示模板定义 | `src/skills/bundled/updateConfig.ts:307-443` |
| Settings 类型定义 | `src/utils/settings/types.ts:255-1148` |
| Hooks Schema 定义 | `src/schemas/hooks.ts:16-222` |
| BundledSkill 注册器 | `src/skills/bundledSkills.ts:53-100` |
| 权限规则验证 | `src/utils/settings/permissionValidation.ts:58-262` |

---

## 依赖与外部交互

### 直接依赖

| 模块 | 路径 | 用途 |
|------|------|------|
| `toJSONSchema` | `zod/v4` | 将 Zod schema 转换为 JSON Schema |
| `SettingsSchema` | `../../utils/settings/types.js` | 设置项的 Zod schema 定义 |
| `jsonStringify` | `../../utils/slowOperations.js` | 带慢操作检测的 JSON 序列化 |
| `registerBundledSkill` | `../bundledSkills.js` | 注册 bundled skill 到系统 |

### 间接依赖

| 模块 | 路径 | 用途 |
|------|------|------|
| `HooksSchema` | `src/schemas/hooks.ts` | Hooks 配置的 Zod schema |
| `PermissionRuleSchema` | `src/utils/settings/permissionValidation.ts` | 权限规则验证 |
| `feature()` | `bun:bundle` | 功能标志检查 |

### 外部交互
- **无直接网络调用**：该模块仅生成提示文本，不直接操作文件系统或网络
- **通过 Read 工具**：技能声明 `allowedTools: ['Read']`，通过 Read 工具读取现有配置文件
- **通过 Edit/Write 工具**：用户通过模型调用 Edit/Write 工具实际修改配置文件

---

## 风险、边界与改进建议

### 风险点

1. **Schema 生成性能**
   - `generateSettingsSchema()` 在每次调用时动态生成 JSON Schema
   - 虽然使用了 `jsonStringify` 的慢操作检测，但大型 schema 的生成仍可能耗时
   - **缓解**：SettingsSchema 使用 `lazySchema` 延迟初始化

2. **提示长度限制**
   - 完整的提示包含 SETTINGS_EXAMPLES_DOCS + HOOKS_DOCS + HOOK_VERIFICATION_FLOW + 动态 JSON Schema
   - 总长度可能超过 15KB，接近某些模型的上下文限制
   - **现状**：代码未做长度检查或截断处理

3. **Hooks 验证依赖外部工具**
   - 文档中提到的 `jq -e` 验证需要用户系统安装 jq
   - 未安装 jq 时验证步骤会失败

### 边界条件

1. **空参数处理**
   ```typescript
   if (args) {
     prompt += `\n\n## User Request\n\n${args}`
   }
   ```
   当 args 为空字符串时，不添加用户请求部分

2. **[hooks-only] 前缀匹配**
   ```typescript
   if (args.startsWith('[hooks-only]')) {
     const req = args.slice('[hooks-only]'.length).trim()
   }
   ```
   严格前缀匹配，大小写敏感

3. **USER_TYPE 限制**
   - `verify.ts` 中的 skill 仅限 `USER_TYPE === 'ant'` 用户
   - `updateConfig.ts` 本身无此限制，对所有用户可用

### 改进建议

1. **缓存生成的 Schema**
   ```typescript
   // 建议添加
   let cachedSchema: string | null = null
   function generateSettingsSchema(): string {
     if (!cachedSchema) {
       cachedSchema = jsonStringify(toJSONSchema(SettingsSchema(), { io: 'input' }), null, 2)
     }
     return cachedSchema
   }
   ```

2. **提示长度监控**
   - 添加提示长度日志，监控接近上下文限制的情况
   - 考虑将大型文档拆分为可按需读取的参考文件

3. **Hooks 验证工具内置**
   - 考虑内置简单的 JSON 验证逻辑，减少对 jq 的外部依赖

4. **类型安全增强**
   - `args` 参数为 `string` 类型，但解析 `[hooks-only]` 前缀时无结构化验证
   - 可考虑使用更严格的参数解析方案

5. **文档同步检查**
   - 添加 CI 检查，确保当 `SettingsSchema` 变更时，手动编写的示例文档（SETTINGS_EXAMPLES_DOCS）也得到更新
