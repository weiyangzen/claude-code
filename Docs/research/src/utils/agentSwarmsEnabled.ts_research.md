# agentSwarmsEnabled.ts 深度研究文档

## 场景与职责

`agentSwarmsEnabled.ts` 是 Claude Code CLI 中**代理团队（Agent Swarms/Teammates）功能的集中式功能开关**。它提供了一个统一的运行时检查机制，用于控制所有与 swarm/teammate 相关的功能是否可用。

### 核心场景

1. **功能发布控制**：作为功能发布（feature rollout）的单一控制点
2. **内外部构建差异**：内部（Ant）构建与外部用户构建的功能差异管理
3. **安全开关（Killswitch）**：通过远程配置（GrowthBook）在紧急情况下快速关闭功能
4. **用户选择加入**：外部用户需要通过环境变量或 CLI 标志显式启用实验性功能

### 设计原则

模块注释强调了以下设计原则：
- **单一入口**：这是应该被所有引用 teammate 的地方检查的唯一入口
- **分层控制**：Ant 构建总是启用，外部构建需要满足多个条件
- **安全第一**：即使外部用户选择加入，也可以通过 killswitch 远程关闭

---

## 功能点目的

### 1. 功能启用检查

`isAgentSwarmsEnabled()` 是核心函数，实现了以下启用逻辑：

| 用户类型 | 启用条件 |
|----------|----------|
| Ant (内部) | 总是启用 |
| 外部用户 | 需要同时满足：(1) 选择加入 + (2) Killswitch 开启 |

### 2. 选择加入机制

外部用户可以通过以下方式选择加入：
- 环境变量：`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`
- CLI 标志：`--agent-teams`

### 3. Killswitch 机制

通过 GrowthBook 功能标志 `tengu_amber_flint` 控制：
- 标志为 `true`：功能可用
- 标志为 `false`：功能被禁用（紧急情况下远程关闭）

---

## 具体技术实现

### 启用逻辑流程

```
isAgentSwarmsEnabled()
├── Ant 用户?
│   └── YES → 返回 true
│
└── 外部用户
    ├── 选择加入检查
    │   ├── CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS 设置?
    │   │   └── YES → 继续
    │   └── --agent-teams 标志存在?
    │       └── NO → 返回 false
    │
    └── Killswitch 检查
        ├── GrowthBook: tengu_amber_flint = true?
        │   └── NO → 返回 false
        └── YES → 返回 true
```

### 核心代码实现

```typescript
export function isAgentSwarmsEnabled(): boolean {
  // Ant: always on
  if (process.env.USER_TYPE === 'ant') {
    return true
  }

  // External: require opt-in via env var or --agent-teams flag
  if (
    !isEnvTruthy(process.env.CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS) &&
    !isAgentTeamsFlagSet()
  ) {
    return false
  }

  // Killswitch — always respected for external users
  if (!getFeatureValue_CACHED_MAY_BE_STALE('tengu_amber_flint', true)) {
    return false
  }

  return true
}
```

### 辅助函数

#### 1. CLI 标志检查
```typescript
function isAgentTeamsFlagSet(): boolean {
  return process.argv.includes('--agent-teams')
}
```
- 直接检查 `process.argv` 避免与 bootstrap/state 的导入循环
- 注释说明：即使该标志只在 Ant 用户的帮助中显示，外部用户传递它也会生效

#### 2. 环境变量检查
```typescript
// 来自 envUtils.ts
export function isEnvTruthy(envVar: string | boolean | undefined): boolean {
  if (!envVar) return false
  if (typeof envVar === 'boolean') return envVar
  const normalizedValue = envVar.toLowerCase().trim()
  return ['1', 'true', 'yes', 'on'].includes(normalizedValue)
}
```
- 支持多种真值表示：`1`, `true`, `yes`, `on`
- 大小写不敏感

---

## 关键代码路径与文件引用

### 调用方（Consumers）

该函数被广泛使用于以下场景：

| 类别 | 文件 | 用途 |
|------|------|------|
| **工具** | `src/tools.ts` | 工具注册时检查 |
| | `src/tools/AgentTool/AgentTool.tsx` | Agent 工具启用检查 |
| | `src/tools/TaskCreateTool/prompt.ts` | 任务创建提示 |
| | `src/tools/TaskUpdateTool/TaskUpdateTool.ts` | 任务更新工具 |
| | `src/tools/TeamCreateTool/TeamCreateTool.ts` | 团队创建工具 |
| | `src/tools/TeamDeleteTool/TeamDeleteTool.ts` | 团队删除工具 |
| | `src/tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts` | 退出计划模式 |
| | `src/tools/SendMessageTool/SendMessageTool.ts` | 发送消息工具 |
| **UI 组件** | `src/components/TaskListV2.tsx` | 任务列表 UI |
| | `src/components/messages/UserTextMessage.tsx` | 用户消息显示 |
| | `src/components/messages/AttachmentMessage.tsx` | 附件消息 |
| | `src/components/Settings/Config.tsx` | 设置配置 |
| | `src/components/PromptInput/PromptInput.tsx` | 提示输入 |
| | `src/components/PromptInput/PromptInputModeIndicator.tsx` | 模式指示器 |
| | `src/components/PromptInput/PromptInputFooterLeftSide.tsx` | 输入页脚 |
| | `src/components/permissions/ExitPlanModePermissionRequest/ExitPlanModePermissionRequest.tsx` | 权限请求 |
| **Hooks** | `src/hooks/useSwarmInitialization.ts` | Swarm 初始化 |
| | `src/hooks/toolPermission/handlers/swarmWorkerHandler.ts` | 权限处理 |
| | `src/hooks/useLogMessages.ts` | 日志消息 |
| | `src/hooks/useTypeahead.tsx` | 类型提示 |
| **其他** | `src/setup.ts` | 应用设置 |
| | `src/main.tsx` | 主入口 |
| | `src/screens/REPL.tsx` | REPL 界面 |
| | `src/utils/agentContext.ts` | 代理上下文类型守卫 |
| | `src/utils/api.ts` | API 工具 |
| | `src/utils/messages.ts` | 消息处理 |
| | `src/utils/attachments.ts` | 附件处理 |
| | `src/services/PromptSuggestion/promptSuggestion.ts` | 提示建议 |

### 依赖方

| 模块 | 关系 |
|------|------|
| `growthbook.ts` | 运行时依赖 - 功能标志获取 |
| `envUtils.ts` | 运行时依赖 - 环境变量检查 |

---

## 依赖与外部交互

### 直接依赖

```typescript
import { getFeatureValue_CACHED_MAY_BE_STALE } from '../services/analytics/growthbook.js'
import { isEnvTruthy } from './envUtils.js'
```

### GrowthBook 集成

- **功能标志**：`tengu_amber_flint`
- **默认值**：`true`（功能默认开启，除非明确关闭）
- **缓存注意**：函数名中的 `CACHED_MAY_BE_STALE` 表示值可能来自缓存，不一定是最新的

### 环境变量

| 变量名 | 用途 | 示例值 |
|--------|------|--------|
| `USER_TYPE` | 区分 Ant 和外部用户 | `'ant'` / `'external'` |
| `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` | 外部用户选择加入 | `'1'`, `'true'`, `'yes'`, `'on'` |

### CLI 参数

| 标志 | 用途 |
|------|------|
| `--agent-teams` | 外部用户选择加入 swarm 功能 |

---

## 风险、边界与改进建议

### 已知风险

1. **缓存延迟风险**
   - 使用 `getFeatureValue_CACHED_MAY_BE_STALE` 表示值可能来自缓存
   - 如果 killswitch 被远程切换，可能需要一段时间才能生效
   - 风险：在紧急情况下，功能可能不会立即被禁用

2. **导入循环风险**
   - 注释特别提到检查 `process.argv` 直接以避免与 `bootstrap/state` 的导入循环
   - 如果未来重构不小心引入循环依赖，可能导致模块加载失败

3. **功能标志命名泄露**
   - `tengu_amber_flint` 是内部代码名，可能暴露内部实现细节
   - 虽然外部构建可能通过代码分析发现，但功能本身被保护

4. **测试复杂性**
   - Ant 构建总是启用，外部构建有条件启用
   - 测试需要覆盖两种场景，增加了测试复杂度

### 边界情况

1. **环境变量值处理**
   ```typescript
   // 以下值都被视为 true
   isEnvTruthy('1')      // true
   isEnvTruthy('true')   // true
   isEnvTruthy('TRUE')   // true
   isEnvTruthy('yes')    // true
   isEnvTruthy('on')     // true
   
   // 以下值被视为 false
   isEnvTruthy('0')      // false
   isEnvTruthy('false')  // false
   isEnvTruthy('')       // false
   isEnvTruthy(undefined)// false
   ```

2. **USER_TYPE 未设置**
   - 如果 `USER_TYPE` 未设置，被视为外部用户
   - 需要满足外部用户的所有条件

3. **GrowthBook 不可用**
   - 如果 GrowthBook 服务不可用，使用默认值 `true`
   - 这意味着功能保持开启，除非明确关闭

### 改进建议

1. **添加强制刷新机制**
   ```typescript
   export function isAgentSwarmsEnabled(forceRefresh = false): boolean {
     // Ant: always on
     if (process.env.USER_TYPE === 'ant') {
       return true
     }
     
     // ... opt-in check ...
     
     // Killswitch with optional force refresh
     const killswitchValue = forceRefresh 
       ? getFeatureValue_FRESH('tengu_amber_flint', true)
       : getFeatureValue_CACHED_MAY_BE_STALE('tengu_amber_flint', true)
     
     return killswitchValue
   }
   ```

2. **添加遥测记录**
   ```typescript
   export function isAgentSwarmsEnabled(): boolean {
     const enabled = checkEnabled()
     
     // 记录功能检查事件（仅一次）
     logEventOnce('tengu_swarm_feature_check', {
       enabled,
       userType: process.env.USER_TYPE,
       optInMethod: detectOptInMethod(),
     })
     
     return enabled
   }
   ```

3. **改进错误处理**
   ```typescript
   export function isAgentSwarmsEnabled(): boolean {
     try {
       // ... existing logic ...
     } catch (error) {
       // 出错时保守返回 false
       logError(error)
       return false
     }
   }
   ```

4. **添加配置验证**
   ```typescript
   export function validateSwarmConfig(): { valid: boolean; errors: string[] } {
     const errors: string[] = []
     
     if (process.env.USER_TYPE !== 'ant') {
       if (isEnvTruthy(process.env.CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS) && 
           isAgentTeamsFlagSet()) {
         errors.push('Both env var and CLI flag are set, using env var')
       }
     }
     
     return { valid: errors.length === 0, errors }
   }
   ```

5. **文档完善**
   - 添加更多使用示例
   - 明确说明功能标志的更新频率
   - 提供故障排除指南

6. **考虑 A/B 测试支持**
   ```typescript
   export function getSwarmFeatureVariant(): 'control' | 'treatment' | 'disabled' {
     // 支持更细粒度的功能发布控制
     if (process.env.USER_TYPE === 'ant') {
       return 'treatment'
     }
     // ... 检查实验分组
   }
   ```
