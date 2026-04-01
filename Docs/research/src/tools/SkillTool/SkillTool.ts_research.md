# SkillTool.ts 研究文档

## 场景与职责

`SkillTool.ts` 是 Claude Code 中负责**技能调用**的核心工具实现。它实现了 `Tool` 接口，允许 AI 模型通过工具调用的方式执行各种技能（skills），包括：

1. **本地技能执行** - 执行项目目录下的技能文件（如 `/skills/*.md`）
2. **Forked 技能执行** - 在隔离的子代理上下文中运行复杂技能
3. **远程技能执行**（实验性）- 从 AKI/GCS 加载并执行远程规范技能
4. **MCP 技能集成** - 支持通过 MCP 协议提供的技能

该文件是 SkillTool 目录的主入口，导出了完整的工具定义 `SkillTool`，被注册到全局工具列表中（`src/tools.ts` 第 212 行）。

## 功能点目的

### 1. 技能发现与验证 (`validateInput`)
- 验证技能名称格式（支持带/不带前导斜杠）
- 检查技能是否存在（通过 `findCommand` 在命令列表中查找）
- 检查技能是否允许模型调用（`disableModelInvocation` 标志）
- 检查技能类型是否为 prompt-based
- 支持实验性的远程规范技能（`_canonical_<slug>` 格式）

### 2. 权限控制 (`checkPermissions`)
- 支持基于规则的权限检查（allow/deny/ask）
- 支持前缀匹配规则（如 `review:*` 匹配 `review-pr 123`）
- 自动允许仅使用安全属性的技能（`skillHasOnlySafeProperties`）
- 为未知技能提供权限建议（添加精确匹配或前缀规则）

### 3. 技能执行 (`call`)
支持三种执行模式：

#### 3.1 内联执行（Inline）
- 通过 `processPromptSlashCommand` 处理技能提示
- 将技能内容展开为新的用户消息注入对话
- 支持 `allowedTools` 和 `model` 覆盖
- 返回 `contextModifier` 修改后续工具使用上下文

#### 3.2 Forked 执行
- 在独立的子代理中运行技能（`executeForkedSkill`）
- 使用 `runAgent` 启动隔离的查询循环
- 支持进度回调（`onProgress`）
- 适用于复杂、多步骤的技能

#### 3.3 远程技能执行（实验性）
- 从 AKI/GCS 加载远程 SKILL.md（`executeRemoteSkill`）
- 支持本地缓存
- 自动处理 YAML frontmatter 和变量替换（`${CLAUDE_SKILL_DIR}`, `${CLAUDE_SESSION_ID}`）

### 4. 遥测与日志
- 记录技能调用事件（`tengu_skill_tool_invocation`）
- 区分执行上下文（inline/forked/remote）
- 记录调用触发方式（嵌套技能 vs 主动调用）
- 支持插件技能的遥测（市场、仓库信息）

## 具体技术实现

### 关键数据结构

```typescript
// 输入模式（Zod Schema）
inputSchema = z.object({
  skill: z.string(),  // 技能名称
  args: z.string().optional(),  // 可选参数
})

// 输出模式（联合类型）
outputSchema = z.union([
  // 内联执行输出
  z.object({
    success: z.boolean(),
    commandName: z.string(),
    allowedTools: z.array(z.string()).optional(),
    model: z.string().optional(),
    status: z.literal('inline'),
  }),
  // Forked 执行输出
  z.object({
    success: z.boolean(),
    commandName: z.string(),
    status: z.literal('forked'),
    agentId: z.string(),
    result: z.string(),
  }),
])
```

### 关键流程

#### 技能调用流程
```
1. validateInput(skill, context)
   ├── 去除前导斜杠（如需要）
   ├── 检查远程规范技能（_canonical_ 前缀）
   ├── 获取所有命令（包括 MCP 技能）
   ├── 查找命令是否存在
   ├── 检查 disableModelInvocation
   └── 验证类型为 prompt

2. checkPermissions({skill, args}, context)
   ├── 解析命令名称
   ├── 检查 deny 规则（精确匹配或前缀匹配）
   ├── 检查远程规范技能（自动允许）
   ├── 检查 allow 规则
   ├── 检查是否仅使用安全属性
   └── 返回 ask + 建议规则

3. call({skill, args}, context, canUseTool, parentMessage, onProgress)
   ├── 如果是远程规范技能 → executeRemoteSkill
   ├── 查找命令
   ├── 记录技能使用
   ├── 如果是 fork 上下文 → executeForkedSkill
   │   ├── prepareForkedCommandContext
   │   ├── runAgent（子代理循环）
   │   └── 提取结果文本
   └── 否则 → 内联执行
       ├── processPromptSlashCommand
       ├── 记录遥测
       ├── 标记消息 sourceToolUseID
       └── 返回 newMessages + contextModifier
```

### 安全属性白名单

```typescript
const SAFE_SKILL_PROPERTIES = new Set([
  'type', 'progressMessage', 'contentLength', 'argNames',
  'model', 'effort', 'source', 'pluginInfo', 'disableNonInteractive',
  'skillRoot', 'context', 'agent', 'getPromptForCommand',
  'frontmatterKeys', 'name', 'description', 'hasUserSpecifiedDescription',
  'isEnabled', 'isHidden', 'aliases', 'isMcp', 'argumentHint',
  'whenToUse', 'paths', 'version', 'disableModelInvocation',
  'userInvocable', 'loadedFrom', 'immediate', 'userFacingName',
])
```

仅包含这些属性的技能可自动获得执行权限，无需用户确认。

### 远程技能加载（实验性）

```typescript
// 条件加载远程技能模块
const remoteSkillModules = feature('EXPERIMENTAL_SKILL_SEARCH')
  ? {
      ...(require('../../services/skillSearch/remoteSkillState.js')),
      ...(require('../../services/skillSearch/remoteSkillLoader.js')),
      ...(require('../../services/skillSearch/telemetry.js')),
      ...(require('../../services/skillSearch/featureCheck.js')),
    }
  : null
```

## 关键代码路径与文件引用

### 直接依赖

| 文件路径 | 用途 |
|---------|------|
| `src/Tool.ts` | `Tool`, `ToolDef`, `ToolResult`, `ToolUseContext` 类型定义 |
| `src/commands.ts` | `getCommands`, `findCommand`, `PromptCommand` |
| `src/tools/AgentTool/runAgent.ts` | `runAgent` - 子代理执行 |
| `src/utils/forkedAgent.ts` | `prepareForkedCommandContext`, `extractResultText` |
| `src/tools/SkillTool/constants.ts` | `SKILL_TOOL_NAME` |
| `src/tools/SkillTool/prompt.ts` | `getPrompt` - 工具提示生成 |
| `src/tools/SkillTool/UI.tsx` | UI 渲染函数 |
| `src/bootstrap/state.ts` | `addInvokedSkill`, `clearInvokedSkillsForAgent`, `getSessionId` |
| `src/utils/permissions/permissions.ts` | `getRuleByContentsForTool` |
| `src/utils/model/model.ts` | `resolveSkillModelOverride` |

### 远程技能相关（条件加载）

| 文件路径 | 用途 |
|---------|------|
| `src/services/skillSearch/remoteSkillState.ts` | `getDiscoveredRemoteSkill`, `isSkillSearchEnabled` |
| `src/services/skillSearch/remoteSkillLoader.ts` | `loadRemoteSkill`, `stripCanonicalPrefix` |
| `src/services/skillSearch/telemetry.ts` | `logRemoteSkillLoaded` |

### 被调用方

| 文件路径 | 调用方式 |
|---------|---------|
| `src/tools.ts` | 导入并注册到 `getAllBaseTools()` |
| `src/utils/processUserInput/processSlashCommand.ts` | `processPromptSlashCommand`（动态导入）|
| `src/utils/suggestions/skillUsageTracking.ts` | `recordSkillUsage` |

## 依赖与外部交互

### 核心依赖

1. **Zod** - 输入/输出 Schema 验证
2. **Lodash** - `uniqBy`, `memoize`
3. **Bun** - `feature` 标志检查
4. **Anthropic SDK** - `ToolResultBlockParam` 类型

### 外部服务交互

1. **遥测服务** (`src/services/analytics/index.ts`)
   - `logEvent('tengu_skill_tool_invocation', {...})`
   - `logEvent('tengu_skill_tool_slash_prefix')`
   - `logEvent('tengu_skill_descriptions_truncated')`（通过 prompt.ts）

2. **远程技能服务**（条件性）
   - AKI/GCS 文件下载
   - 本地文件缓存

3. **权限系统** (`src/utils/permissions/`)
   - 规则匹配引擎
   - 决策追踪

### 状态管理

- **AppState**: 通过 `context.getAppState()` / `setAppState()` 访问
- **Session State**: `addInvokedSkill` / `clearInvokedSkillsForAgent` 跟踪已调用技能
- **Agent Context**: `getAgentContext()` 获取当前代理信息

## 风险、边界与改进建议

### 已知风险

1. **远程技能加载失败**
   - 网络故障或 URL 失效会导致技能加载失败
   - 已添加错误处理和遥测记录，但用户体验仍需优化

2. **权限绕过风险**
   - `skillHasOnlySafeProperties` 白名单需要持续维护
   - 新增属性默认需要权限，直到被明确审查并加入白名单

3. **Forked 技能资源泄漏**
   - 子代理可能产生长时间运行的任务
   - 已使用 `clearInvokedSkillsForAgent` 在 finally 块中清理

4. **缓存一致性问题**
   - `getPrompt` 使用 memoization，可能导致提示更新不及时
   - `clearPromptCache` 函数提供手动清除能力

### 边界条件

1. **空技能名称** - 返回错误码 1
2. **未知技能** - 返回错误码 2
3. **禁用模型调用的技能** - 返回错误码 4
4. **非 prompt 类型技能** - 返回错误码 5
5. **未发现的远程技能** - 返回错误码 6

### 改进建议

1. **性能优化**
   - 考虑为 `getAllCommands` 添加更细粒度的缓存策略
   - 远程技能加载可添加并行预加载机制

2. **可观测性**
   - 添加技能执行耗时直方图
   - 增加技能缓存命中率监控

3. **安全增强**
   - 为远程技能添加内容签名验证
   - 实现技能沙箱隔离（目前依赖 fork 的进程隔离）

4. **代码组织**
   - 三种执行模式（inline/forked/remote）逻辑可进一步拆分为独立模块
   - `call` 方法较长（200+ 行），可考虑提取策略模式

5. **类型安全**
   - `remoteSkillModules` 使用条件 require，类型推断较弱
   - 考虑使用更严格的类型守卫

### 测试注意事项

- 需要覆盖三种执行模式
- 权限规则匹配需要测试前缀和精确匹配
- 远程技能加载需要 mock AKI/GCS 服务
- Forked 执行需要验证子代理生命周期管理
