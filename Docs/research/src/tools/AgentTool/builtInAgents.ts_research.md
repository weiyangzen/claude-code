# builtInAgents.ts 深度研究文档

## 场景与职责

`builtInAgents.ts` 是 Claude Code 中负责**内置代理管理**的核心模块。它定义了系统内置的代理类型，并提供动态加载和特性开关控制功能。该模块是代理系统的"代理注册中心"，决定了哪些内置代理对用户可用。

该模块的核心职责：
1. **内置代理注册**：定义和管理系统内置的代理类型
2. **特性开关控制**：通过 Feature Flag 控制代理的启用/禁用
3. **动态代理加载**：支持 Coordinator 模式下的动态代理获取
4. **SDK 兼容性**：处理 SDK 使用场景下的代理禁用逻辑

## 功能点目的

### 1. 内置代理定义
当前系统定义了以下内置代理：
- **GENERAL_PURPOSE_AGENT**：通用目的代理，默认代理类型
- **STATUSLINE_SETUP_AGENT**：状态栏设置代理
- **EXPLORE_AGENT**：代码探索代理（特性控制）
- **PLAN_AGENT**：计划代理（特性控制）
- **CLAUDE_CODE_GUIDE_AGENT**：Claude Code 指南代理（非 SDK 场景）
- **VERIFICATION_AGENT**：验证代理（实验性）

### 2. 特性开关控制
- `areExplorePlanAgentsEnabled()`：控制 Explore/Plan 代理的启用
  - 依赖 `BUILTIN_EXPLORE_PLAN_AGENTS` 特性
  - 使用 GrowthBook A/B 测试 (`tengu_amber_stoat`)
  - 3P 默认启用，A/B 测试可能禁用

### 3. Coordinator 模式支持
- 检测 Coordinator 模式（`COORDINATOR_MODE` 特性 + `CLAUDE_CODE_COORDINATOR_MODE` 环境变量）
- 动态加载 Coordinator 代理（使用 `require` 避免循环依赖）

### 4. SDK 禁用支持
- 支持通过 `CLAUDE_AGENT_SDK_DISABLE_BUILTIN_AGENTS` 环境变量禁用所有内置代理
- 仅在非交互式会话（SDK/API 使用）时生效

## 具体技术实现

### 核心函数实现

**1. Explore/Plan 代理特性检查**

```typescript
export function areExplorePlanAgentsEnabled(): boolean {
  if (feature('BUILTIN_EXPLORE_PLAN_AGENTS')) {
    // 3P default: true — Bedrock/Vertex keep agents enabled
    // A/B test treatment sets false to measure impact of removal
    return getFeatureValue_CACHED_MAY_BE_STALE('tengu_amber_stoat', true)
  }
  return false
}
```

**2. 内置代理获取**

```typescript
export function getBuiltInAgents(): AgentDefinition[] {
  // 1. SDK 禁用检查
  if (
    isEnvTruthy(process.env.CLAUDE_AGENT_SDK_DISABLE_BUILTIN_AGENTS) &&
    getIsNonInteractiveSession()
  ) {
    return []
  }

  // 2. Coordinator 模式检查
  if (feature('COORDINATOR_MODE')) {
    if (isEnvTruthy(process.env.CLAUDE_CODE_COORDINATOR_MODE)) {
      const { getCoordinatorAgents } =
        require('../../coordinator/workerAgent.js') as typeof import('../../coordinator/workerAgent.js')
      return getCoordinatorAgents()
    }
  }

  // 3. 构建基础代理列表
  const agents: AgentDefinition[] = [
    GENERAL_PURPOSE_AGENT,
    STATUSLINE_SETUP_AGENT,
  ]

  // 4. 条件添加 Explore/Plan 代理
  if (areExplorePlanAgentsEnabled()) {
    agents.push(EXPLORE_AGENT, PLAN_AGENT)
  }

  // 5. 非 SDK 入口点添加指南代理
  const isNonSdkEntrypoint =
    process.env.CLAUDE_CODE_ENTRYPOINT !== 'sdk-ts' &&
    process.env.CLAUDE_CODE_ENTRYPOINT !== 'sdk-py' &&
    process.env.CLAUDE_CODE_ENTRYPOINT !== 'sdk-cli'

  if (isNonSdkEntrypoint) {
    agents.push(CLAUDE_CODE_GUIDE_AGENT)
  }

  // 6. 实验性验证代理
  if (
    feature('VERIFICATION_AGENT') &&
    getFeatureValue_CACHED_MAY_BE_STALE('tengu_hive_evidence', false)
  ) {
    agents.push(VERIFICATION_AGENT)
  }

  return agents
}
```

### 代理定义导入

```typescript
import { CLAUDE_CODE_GUIDE_AGENT } from './built-in/claudeCodeGuideAgent.js'
import { EXPLORE_AGENT } from './built-in/exploreAgent.js'
import { GENERAL_PURPOSE_AGENT } from './built-in/generalPurposeAgent.js'
import { PLAN_AGENT } from './built-in/planAgent.js'
import { STATUSLINE_SETUP_AGENT } from './built-in/statuslineSetup.js'
import { VERIFICATION_AGENT } from './built-in/verificationAgent.js'
```

### 特性标志

| 特性名 | 用途 |
|-------|------|
| `BUILTIN_EXPLORE_PLAN_AGENTS` | 控制 Explore/Plan 代理功能开关 |
| `COORDINATOR_MODE` | 控制 Coordinator 模式功能开关 |
| `VERIFICATION_AGENT` | 控制验证代理功能开关 |

### GrowthBook 标志

| 标志名 | 默认值 | 用途 |
|-------|-------|------|
| `tengu_amber_stoat` | `true` | Explore/Plan 代理 A/B 测试 |
| `tengu_hive_evidence` | `false` | 验证代理启用控制 |

## 依赖与外部交互

### 依赖模块

| 模块路径 | 用途 |
|---------|------|
| `bun:bundle` | Feature flag 检查 (`feature`) |
| `../../bootstrap/state.js` | 检查非交互式会话 (`getIsNonInteractiveSession`) |
| `../../services/analytics/growthbook.js` | GrowthBook 特性值获取 |
| `../../utils/envUtils.js` | 环境变量检查 (`isEnvTruthy`) |
| `./built-in/*.js` | 内置代理定义 |
| `./loadAgentsDir.js` | 代理定义类型 (`AgentDefinition`) |

### 被调用方

通过 Grep 搜索，该模块被以下文件引用：
- `src/tools/AgentTool/loadAgentsDir.ts` - 加载代理定义时合并内置代理
- `src/components/agents/AgentDetail.tsx` - 显示代理详情
- `src/components/agents/AgentsList.tsx` - 代理列表显示
- `src/utils/settings/settings.ts` - 设置管理
- `src/utils/systemPrompt.ts` - 系统提示构建

### 环境变量

| 变量名 | 用途 |
|-------|------|
| `CLAUDE_AGENT_SDK_DISABLE_BUILTIN_AGENTS` | SDK 场景下禁用所有内置代理 |
| `CLAUDE_CODE_COORDINATOR_MODE` | 启用 Coordinator 模式 |
| `CLAUDE_CODE_ENTRYPOINT` | 标识入口点类型（sdk-ts/sdk-py/sdk-cli） |

## 风险、边界与改进建议

### 已知风险

1. **循环依赖风险**：使用 `require` 动态加载 Coordinator 代理是为了避免循环依赖，但仍需谨慎
2. **特性标志缓存**：`getFeatureValue_CACHED_MAY_BE_STALE` 可能返回过期的值
3. **环境变量依赖**：过多依赖环境变量可能导致配置复杂化
4. **A/B 测试影响**：Explore/Plan 代理的 A/B 测试可能影响用户体验的一致性

### 边界情况

1. **空代理列表**：如果所有特性都禁用且是 SDK 场景，可能返回空数组
2. **Coordinator 模式冲突**：Coordinator 模式与普通模式代理可能冲突
3. **重复代理**：如果 Coordinator 代理与普通代理重名，可能导致覆盖
4. **动态加载失败**：`require` 失败会导致整个函数抛出异常

### 改进建议

1. **配置化代理列表**：将代理列表改为配置文件驱动，而非硬编码
2. **代理版本控制**：为内置代理添加版本信息，支持平滑升级
3. **代理依赖管理**：明确代理之间的依赖关系，避免循环依赖
4. **懒加载优化**：考虑对所有代理定义使用懒加载，减少启动时间
5. **代理元数据扩展**：添加更多代理元数据（如作者、描述、标签）
6. **代理热更新**：支持运行时动态更新代理定义

### 代码质量建议

1. 添加单元测试覆盖各种特性组合场景
2. 使用依赖注入替代动态 `require`
3. 添加代理加载的日志记录
4. 实现代理加载的性能监控

### 架构建议

1. **插件化架构**：将内置代理改为插件形式，支持第三方扩展
2. **代理注册表**：建立统一的代理注册中心，支持动态发现
3. **代理沙箱**：为代理执行提供沙箱环境，增强安全性
