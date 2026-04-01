# CreateAgentWizard 组件研究文档

## 1. 场景与职责

### 1.1 组件定位

`CreateAgentWizard.tsx` 是 Claude Code CLI 应用中 **Agent 创建向导的主入口组件**，负责编排整个 Agent 创建流程的多步骤交互。它基于通用的 `WizardProvider` 框架，构建了一个完整的 Agent 配置工作流。

### 1.2 核心职责

- **步骤编排**：定义并组装 Agent 创建的完整步骤序列（10-11 个步骤）
- **数据流管理**：通过 WizardProvider 维护跨步骤的共享数据（AgentWizardData）
- **动态步骤控制**：根据条件动态包含/排除步骤（如 MemoryStep 仅在功能开启时显示）
- **回调处理**：处理向导完成和取消事件，通知父组件（AgentsMenu）
- **外部数据注入**：将 `tools` 和 `existingAgents` 等外部数据传递给需要它们的步骤组件

### 1.3 使用场景

该组件在 `AgentsMenu.tsx` 的 `create-agent` 模式下被渲染：

```typescript
// AgentsMenu.tsx 中的调用
<CreateAgentWizard 
  tools={mergedTools} 
  existingAgents={agents} 
  onComplete={handleAgentCreated} 
  onCancel={t13} 
/>
```

用户通过以下路径进入：
1. 主菜单选择 "Agents" → 进入 AgentsMenu
2. 选择 "Create new agent" → 切换到 `create-agent` 模式
3. 渲染 `CreateAgentWizard` 开始创建流程

---

## 2. 功能点目的

### 2.1 步骤流程设计

| 步骤索引 | 组件 | 功能 | 条件/说明 |
|---------|------|------|----------|
| 0 | LocationStep | 选择 Agent 存储位置 | 项目级 (.claude/agents/) 或个人级 (~/.claude/agents/) |
| 1 | MethodStep | 选择创建方式 | "Generate with Claude" 或 "Manual configuration" |
| 2 | GenerateStep | AI 生成 Agent | 仅在 method="generate" 时进入 |
| 3 | TypeStep | 输入 Agent 类型标识符 | 动态组件，接收 existingAgents 用于校验 |
| 4 | PromptStep | 配置系统提示词 | 手动配置时输入系统 Prompt |
| 5 | DescriptionStep | 添加使用描述 | 说明何时使用该 Agent |
| 6 | ToolsStep | 选择可用工具 | 动态组件，接收 tools 列表 |
| 7 | ModelStep | 选择模型 | 可选配置 |
| 8 | ColorStep | 选择颜色主题 | 可选配置 |
| 9 | MemoryStep | 配置记忆范围 | 仅当 `isAutoMemoryEnabled()` 为 true |
| 10 | ConfirmStepWrapper | 确认并保存 | 最终确认页面，执行保存操作 |

### 2.2 分支流程逻辑

**生成路径**（MethodStep 选择 "Generate with Claude"）：
```
LocationStep → MethodStep → GenerateStep → ToolsStep → ModelStep → ColorStep → [MemoryStep] → ConfirmStepWrapper
```
- GenerateStep 调用 AI 生成后，直接跳转到 ToolsStep（索引 6）
- 跳过 TypeStep、PromptStep、DescriptionStep（由 AI 生成）

**手动配置路径**（MethodStep 选择 "Manual configuration"）：
```
LocationStep → MethodStep → TypeStep → PromptStep → DescriptionStep → ToolsStep → ModelStep → ColorStep → [MemoryStep] → ConfirmStepWrapper
```
- MethodStep 调用 `goToStep(3)` 跳转到 TypeStep

### 2.3 动态步骤构建

```typescript
const steps: WizardStepComponent<AgentWizardData>[] = [
  LocationStep,
  MethodStep,
  GenerateStep,
  () => <TypeStep existingAgents={existingAgents} />,  // 动态注入 props
  PromptStep,
  DescriptionStep,
  () => <ToolsStep tools={tools} />,  // 动态注入 props
  ModelStep,
  ColorStep,
  ...(isAutoMemoryEnabled() ? [MemoryStep] : []),  // 条件包含
  () => <ConfirmStepWrapper tools={tools} existingAgents={existingAgents} onComplete={onComplete} />,
]
```

---

## 3. 具体技术实现

### 3.1 AgentWizardData 数据结构

基于代码分析，推断的完整数据类型定义：

```typescript
interface AgentWizardData {
  // 步骤 0: LocationStep
  location?: SettingSource  // 'projectSettings' | 'userSettings'
  
  // 步骤 1: MethodStep
  method?: 'generate' | 'manual'
  wasGenerated?: boolean
  
  // 步骤 2: GenerateStep
  generationPrompt?: string
  isGenerating?: boolean
  generatedAgent?: GeneratedAgent
  
  // 步骤 3: TypeStep (手动) / 步骤 2: GenerateStep (生成)
  agentType?: string
  
  // 步骤 4: PromptStep (手动)
  systemPrompt?: string
  
  // 步骤 5: DescriptionStep (手动)
  whenToUse?: string
  
  // 步骤 6: ToolsStep
  selectedTools?: string[]  // undefined 表示 "All tools"
  
  // 步骤 7: ModelStep
  selectedModel?: string
  
  // 步骤 8: ColorStep
  selectedColor?: AgentColorName
  
  // 步骤 9: MemoryStep
  selectedMemory?: AgentMemoryScope  // 'user' | 'project' | 'local'
  
  // 步骤 8/9/10: ColorStep/MemoryStep 构建的最终 Agent
  finalAgent?: AgentDefinition
}
```

### 3.2 关键流程实现

#### 3.2.1 向导初始化

```typescript
export function CreateAgentWizard({
  tools,
  existingAgents,
  onComplete,
  onCancel,
}: Props): ReactNode {
  // 步骤数组构建（包含动态组件和条件步骤）
  const steps: WizardStepComponent<AgentWizardData>[] = [
    // ... 步骤定义
  ]

  return (
    <WizardProvider<AgentWizardData>
      steps={steps}
      initialData={{}}  // 初始空数据
      onComplete={() => {
        // 完成回调由 ConfirmStepWrapper 内部处理
        // 此处为空是因为 ConfirmStepWrapper 直接调用 onComplete
      }}
      onCancel={onCancel}
      title="Create new agent"
      showStepCounter={false}  // 不显示步骤计数器
    />
  )
}
```

#### 3.2.2 动态组件模式

对于需要外部 props 的步骤，使用箭头函数包装：

```typescript
// 方式 1: 直接组件（无外部依赖）
LocationStep,

// 方式 2: 箭头函数包装（需要外部 props）
() => <TypeStep existingAgents={existingAgents} />,
() => <ToolsStep tools={tools} />,

// 方式 3: 复杂包装（多个 props）
() => (
  <ConfirmStepWrapper
    tools={tools}
    existingAgents={existingAgents}
    onComplete={onComplete}
  />
),
```

#### 3.2.3 条件步骤控制

```typescript
// MemoryStep 仅在功能开启时包含
const conditionalSteps = isAutoMemoryEnabled() ? [MemoryStep] : []
const steps = [...baseSteps, ...conditionalSteps, ConfirmStepWrapper]
```

### 3.3 React Compiler 优化

组件使用 React Compiler 编译，具有以下特征：
- 使用 `_c(n)` 创建 memoization cache（n 为依赖数量）
- 使用 `Symbol.for("react.memo_cache_sentinel")` 作为缓存标记
- 自动比较依赖变化，条件渲染优化

```typescript
// 编译后的代码示例
const $ = _c(17);  // 创建 17 个缓存槽位

// 条件缓存检查
if ($[0] !== existingAgents) {
  t1 = () => <TypeStep existingAgents={existingAgents} />;
  $[0] = existingAgents;
  $[1] = t1;
} else {
  t1 = $[1];  // 复用缓存
}
```

---

## 4. 关键代码路径与文件引用

### 4.1 文件结构

```
src/components/agents/new-agent-creation/
├── CreateAgentWizard.tsx          # 主入口组件（本研究对象）
└── wizard-steps/
    ├── LocationStep.tsx           # 步骤 0: 位置选择
    ├── MethodStep.tsx             # 步骤 1: 创建方式
    ├── GenerateStep.tsx           # 步骤 2: AI 生成
    ├── TypeStep.tsx               # 步骤 3: 类型标识
    ├── PromptStep.tsx             # 步骤 4: 系统提示词
    ├── DescriptionStep.tsx        # 步骤 5: 使用描述
    ├── ToolsStep.tsx              # 步骤 6: 工具选择
    ├── ModelStep.tsx              # 步骤 7: 模型选择
    ├── ColorStep.tsx              # 步骤 8: 颜色选择
    ├── MemoryStep.tsx             # 步骤 9: 记忆配置（可选）
    ├── ConfirmStep.tsx            # 步骤 10: 确认展示
    └── ConfirmStepWrapper.tsx     # 确认步骤包装器（保存逻辑）
```

### 4.2 调用方代码路径

**入口调用** (`src/components/agents/AgentsMenu.tsx`):
```typescript
import { CreateAgentWizard } from './new-agent-creation/CreateAgentWizard.js'

// 在 modeState.mode === "create-agent" 时渲染
case "create-agent":
  return <CreateAgentWizard 
    tools={mergedTools} 
    existingAgents={agents} 
    onComplete={handleAgentCreated} 
    onCancel={handleCancel}
  />
```

### 4.3 依赖导入路径

```typescript
// React 基础
import React, { type ReactNode } from 'react'

// 功能开关
import { isAutoMemoryEnabled } from '../../../memdir/paths.js'

// 类型定义
import type { Tools } from '../../../Tool.js'
import type { AgentDefinition } from '../../../tools/AgentTool/loadAgentsDir.js'

// Wizard 框架
import { WizardProvider } from '../../wizard/index.js'
import type { WizardStepComponent } from '../../wizard/types.js'

// 步骤组件
import { ColorStep } from './wizard-steps/ColorStep.js'
import { ConfirmStepWrapper } from './wizard-steps/ConfirmStepWrapper.js'
import { DescriptionStep } from './wizard-steps/DescriptionStep.js'
import { GenerateStep } from './wizard-steps/GenerateStep.js'
import { LocationStep } from './wizard-steps/LocationStep.js'
import { MemoryStep } from './wizard-steps/MemoryStep.js'
import { MethodStep } from './wizard-steps/MethodStep.js'
import { ModelStep } from './wizard-steps/ModelStep.js'
import { PromptStep } from './wizard-steps/PromptStep.js'
import { ToolsStep } from './wizard-steps/ToolsStep.js'
import { TypeStep } from './wizard-steps/TypeStep.js'
```

---

## 5. 依赖与外部交互

### 5.1 内部依赖

| 依赖模块 | 路径 | 用途 |
|---------|------|------|
| WizardProvider | `src/components/wizard/index.js` | 向导状态管理框架 |
| WizardStepComponent | `src/components/wizard/types.js` | 步骤组件类型定义 |
| isAutoMemoryEnabled | `src/memdir/paths.js` | 记忆功能开关检查 |
| Tools | `src/Tool.js` | 工具类型定义 |
| AgentDefinition | `src/tools/AgentTool/loadAgentsDir.js` | Agent 定义类型 |

### 5.2 步骤组件依赖

| 步骤组件 | 依赖外部数据 | 说明 |
|---------|-------------|------|
| TypeStep | `existingAgents: AgentDefinition[]` | 用于校验类型唯一性 |
| ToolsStep | `tools: Tools` | 工具列表选择 |
| ConfirmStepWrapper | `tools`, `existingAgents`, `onComplete` | 保存时校验和回调 |

### 5.3 与 Wizard 框架的交互

```typescript
// 提供步骤数组和配置
<WizardProvider<AgentWizardData>
  steps={steps}                    // 步骤组件数组
  initialData={{}}                 // 初始空数据
  onComplete={() => {}}            // 完成回调（实际由 ConfirmStepWrapper 处理）
  onCancel={onCancel}              // 取消回调（来自父组件）
  title="Create new agent"         // 向导标题
  showStepCounter={false}          // 隐藏步骤计数
/>
```

### 5.4 与父组件的交互

**Props 接口**：
```typescript
type Props = {
  tools: Tools                      // 可用工具列表
  existingAgents: AgentDefinition[] // 已存在的 Agent（用于校验重复）
  onComplete: (message: string) => void  // 创建成功回调
  onCancel: () => void              // 取消回调
}
```

**回调流程**：
1. 用户在 ConfirmStep 选择 "Save" 或 "Save and Edit"
2. ConfirmStepWrapper 调用 `saveAgentToFile()` 保存文件
3. 成功后调用 `onComplete(message)` 通知 AgentsMenu
4. AgentsMenu 的 `handleAgentCreated` 更新状态，显示成功消息

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 类型定义缺失

**风险**：`AgentWizardData` 类型定义未在代码库中显式声明，仅通过 `import type { AgentWizardData } from './types.js'` 引用，但 `types.ts` 文件不存在。

**影响**：
- 编译时类型检查依赖源映射
- IDE 可能无法提供完整类型提示
- 新开发者难以理解完整数据结构

**缓解**：从代码分析中恢复类型定义（见 3.1 节）

#### 6.1.2 步骤索引硬编码

**风险**：GenerateStep 和 MethodStep 中硬编码了步骤索引：

```typescript
// GenerateStep.tsx
if (method === "generate") {
  goNext()  // 正常前进到 GenerateStep
} else {
  goToStep(3)  // 硬编码跳转到 TypeStep
}

// GenerateStep.tsx 生成完成后
goToStep(6)  // 硬编码跳转到 ToolsStep
```

**影响**：
- 添加/删除步骤时需要同步修改多处硬编码索引
- 容易出错，维护成本高

#### 6.1.3 动态组件的 props 稳定性

**风险**：箭头函数包装导致每次渲染创建新函数：

```typescript
// 每次渲染都创建新函数，可能破坏 React.memo 优化
() => <TypeStep existingAgents={existingAgents} />
```

**影响**：
- React Compiler 缓存可能失效
- 子组件不必要的重渲染

### 6.2 边界情况

| 场景 | 行为 | 说明 |
|------|------|------|
| `isAutoMemoryEnabled()` 运行时变化 | 步骤数组在首次渲染时确定 | 需要重新进入向导才能反映变化 |
| `existingAgents` 在向导期间更新 | 使用初始传入的数组 | 新创建的 Agent 不会立即出现在校验中 |
| 用户快速点击取消 | 调用 `onCancel` | 由父组件处理状态回退 |
| 步骤组件抛出错误 | 无错误边界 | 可能导致整个应用崩溃 |

### 6.3 改进建议

#### 6.3.1 添加类型定义文件

建议创建 `src/components/agents/new-agent-creation/types.ts`：

```typescript
import type { SettingSource } from 'src/utils/settings/constants.js'
import type { AgentDefinition } from '../../../tools/AgentTool/loadAgentsDir.js'
import type { AgentMemoryScope } from '../../../tools/AgentTool/agentMemory.js'
import type { AgentColorName } from '../../../tools/AgentTool/agentColorManager.js'

export interface AgentWizardData {
  location?: SettingSource
  method?: 'generate' | 'manual'
  wasGenerated?: boolean
  generationPrompt?: string
  isGenerating?: boolean
  generatedAgent?: {
    identifier: string
    whenToUse: string
    systemPrompt: string
  }
  agentType?: string
  systemPrompt?: string
  whenToUse?: string
  selectedTools?: string[]
  selectedModel?: string
  selectedColor?: AgentColorName
  selectedMemory?: AgentMemoryScope
  finalAgent?: AgentDefinition
}
```

#### 6.3.2 消除硬编码步骤索引

```typescript
// 建议：使用步骤名称而非索引
const STEP_INDICES = {
  LOCATION: 0,
  METHOD: 1,
  GENERATE: 2,
  TYPE: 3,
  PROMPT: 4,
  DESCRIPTION: 5,
  TOOLS: 6,
  MODEL: 7,
  COLOR: 8,
  MEMORY: 9,
  CONFIRM: 10,
} as const

// 或者使用步骤名称跳转
const stepNames = ['location', 'method', 'generate', ...]
goToStep(stepNames.indexOf('tools'))
```

#### 6.3.3 优化动态组件的缓存

```typescript
// 使用 useMemo 缓存动态组件
const typeStepComponent = useMemo(
  () => () => <TypeStep existingAgents={existingAgents} />,
  [existingAgents]
)

const toolsStepComponent = useMemo(
  () => () => <ToolsStep tools={tools} />,
  [tools]
)
```

#### 6.3.4 添加错误边界

```typescript
// 建议添加错误处理
<WizardProvider>
  <ErrorBoundary fallback={<WizardError />}>
    {/* 步骤内容 */}
  </ErrorBoundary>
</WizardProvider>
```

#### 6.3.5 支持步骤前置条件

```typescript
// 建议：支持动态跳过步骤
const steps = [
  { component: LocationStep },
  { component: MethodStep },
  { 
    component: GenerateStep,
    shouldSkip: (data) => data.method === 'manual'
  },
  // ...
]
```

### 6.4 架构建议

1. **步骤配置化**
   - 当前：步骤数组硬编码在组件中
   - 建议：提取为配置对象，支持插件化扩展

2. **状态持久化**
   - 当前：向导数据仅存在于内存
   - 建议：支持 localStorage 自动保存，崩溃后恢复

3. **步骤验证链**
   - 当前：各步骤自行验证
   - 建议：统一的步骤验证接口，支持跨步骤依赖校验

---

## 附录：完整步骤数据流

```
[LocationStep]          → 设置 location
  ↓
[MethodStep]            → 设置 method, wasGenerated
  ↓ (method=generate)
[GenerateStep]          → 设置 generationPrompt, agentType, systemPrompt, 
  ↓                       whenToUse, generatedAgent, wasGenerated
[ToolsStep]             → 设置 selectedTools
  ↓
[ModelStep]             → 设置 selectedModel
  ↓
[ColorStep]             → 设置 selectedColor, finalAgent
  ↓
[MemoryStep] (可选)      → 设置 selectedMemory, 更新 finalAgent.memory
  ↓
[ConfirmStepWrapper]    → 保存 finalAgent 到文件，调用 onComplete
```

---

*文档生成时间：2026-04-01*
*研究范围：src/components/agents/new-agent-creation/CreateAgentWizard.tsx 及其依赖*
*执行器：kimi (k2p5)*
