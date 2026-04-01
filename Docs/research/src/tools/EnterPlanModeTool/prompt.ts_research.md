# prompt.ts 研究文档

## 场景与职责

prompt.ts 是 EnterPlanModeTool 的**提示内容定义模块**，负责生成工具的系统提示（system prompt）内容。该模块根据用户类型（内部员工/外部用户）提供差异化的使用指导，并支持计划模式 V2 的 Interview Phase 功能。

### 核心职责
1. **提示内容生成**：提供 EnterPlanMode 工具的完整使用说明
2. **用户类型适配**：为内部员工（ant）和外部用户提供不同的指导策略
3. **功能开关集成**：根据 planModeV2 功能状态调整提示内容
4. **跨工具引用**：引用 AskUserQuestion 和 ExitPlanMode 工具名称

---

## 功能点目的

### 1. 外部用户提示策略（getEnterPlanModeToolPromptExternal）
- **策略**：积极主动使用计划模式
- **核心理念**："Prefer using EnterPlanMode for implementation tasks unless they're simple"
- **适用条件**（7 种场景）：
  1. 新功能实现
  2. 多种有效方案可选
  3. 代码修改影响现有行为
  4. 架构决策
  5. 多文件变更（>2-3 文件）
  6. 需求不明确需要探索
  7. 用户偏好重要

- **不使用场景**：
  - 单行或少行修复
  - 添加单个函数
  - 用户已给出具体指令
  - 纯研究/探索任务

### 2. 内部员工提示策略（getEnterPlanModeToolPromptAnt）
- **策略**：保守使用，仅在真正需要时使用
- **核心理念**："Use this tool when a task has genuine ambiguity"
- **适用条件**（3 种场景）：
  1. 重大架构模糊性
  2. 需求不明确需要探索
  3. 高影响重构

- **不使用场景**：
  - 可以合理推断方案的任务
  - 用户请求具体（如 "can we work on X"）
  - 明显的实现模式
  - Bug 修复

### 3. Interview Phase 适配
- **功能**：当 `isPlanModeInterviewPhaseEnabled()` 返回 true 时
- **行为**：省略 "What Happens in Plan Mode" 部分
- **原因**：详细工作流说明通过 plan_mode 附件提供

---

## 具体技术实现

### 关键数据结构

**1. 共享的 "What Happens" 部分**
```typescript
const WHAT_HAPPENS_SECTION = `## What Happens in Plan Mode

In plan mode, you'll:
1. Thoroughly explore the codebase using Glob, Grep, and Read tools
2. Understand existing patterns and architecture
3. Design an implementation approach
4. Present your plan to the user for approval
5. Use ${ASK_USER_QUESTION_TOOL_NAME} if you need to clarify approaches
6. Exit plan mode with ExitPlanMode when ready to implement

`
```

**2. 外部用户提示函数**
```typescript
function getEnterPlanModeToolPromptExternal(): string {
  const whatHappens = isPlanModeInterviewPhaseEnabled() ? '' : WHAT_HAPPENS_SECTION
  
  return `Use this tool proactively when you're about to start a non-trivial implementation task...

## When to Use This Tool

**Prefer using EnterPlanMode** for implementation tasks unless they're simple...

## When NOT to Use This Tool
...

${whatHappens}## Examples
...
`
}
```

**3. 内部员工提示函数**
```typescript
function getEnterPlanModeToolPromptAnt(): string {
  const whatHappens = isPlanModeInterviewPhaseEnabled() ? '' : WHAT_HAPPENS_SECTION
  
  return `Use this tool when a task has genuine ambiguity about the right approach...

## When to Use This Tool
...

## When NOT to Use This Tool
...

${whatHappens}## Examples
...
`
}
```

**4. 主入口函数**
```typescript
export function getEnterPlanModeToolPrompt(): string {
  return process.env.USER_TYPE === 'ant'
    ? getEnterPlanModeToolPromptAnt()
    : getEnterPlanModeToolPromptExternal()
}
```

### 提示内容结构对比

| 部分 | 外部用户 | 内部员工 |
|------|---------|---------|
| 开场白 | " proactively " | "when genuine ambiguity" |
| 使用条件 | 7 种场景 | 3 种场景 |
| 不使用条件 | 4 种场景 | 5 种场景 |
| 示例数量 | 5 个（3 正 2 负） | 4 个（2 正 2 负） |
| 重要提示 | 强调用户批准 | 强调避免过度规划 |

### 关键代码路径

**1. 外部用户使用场景定义**
```typescript
## When to Use This Tool

**Prefer using EnterPlanMode** for implementation tasks unless they're simple. Use it when ANY of these conditions apply:

1. **New Feature Implementation**: Adding meaningful new functionality
   - Example: "Add a logout button" - where should it go? What should happen on click?
   
2. **Multiple Valid Approaches**: The task can be solved in several different ways
   - Example: "Add caching to the API" - could use Redis, in-memory, file-based, etc.

3. **Code Modifications**: Changes that affect existing behavior or structure
4. **Architectural Decisions**: The task requires choosing between patterns or technologies
5. **Multi-File Changes**: The task will likely touch more than 2-3 files
6. **Unclear Requirements**: You need to explore before understanding the full scope
7. **User Preferences Matter**: If you would use AskUserQuestion to clarify the approach
```

**2. 内部员工使用场景定义**
```typescript
## When to Use This Tool

Plan mode is valuable when the implementation approach is genuinely unclear. Use it when:

1. **Significant Architectural Ambiguity**: Multiple reasonable approaches exist and the choice meaningfully affects the codebase
   - Example: "Add caching to the API" - Redis vs in-memory vs file-based

2. **Unclear Requirements**: You need to explore and clarify before you can make progress
3. **High-Impact Restructuring**: The task will significantly restructure existing code
```

**3. 示例对比**

外部用户 GOOD 示例：
- "Add user authentication to the app"
- "Optimize the database queries"
- "Implement dark mode"
- "Add a delete button to the user profile"
- "Update the error handling in the API"

内部员工 GOOD 示例：
- "Add user authentication to the app"（真正模糊）
- "Redesign the data pipeline"（重大重构）

---

## 依赖与外部交互

### 依赖模块

| 模块 | 导入内容 | 用途 |
|------|---------|------|
| `../../utils/planModeV2.js` | `isPlanModeInterviewPhaseEnabled` | 检查是否启用 Interview Phase |
| `../AskUserQuestionTool/prompt.js` | `ASK_USER_QUESTION_TOOL_NAME` | 引用 AskUserQuestion 工具名称 |

### 环境变量依赖

| 变量 | 用途 |
|------|------|
| `process.env.USER_TYPE` | 区分内部员工（'ant'）和外部用户 |

### 导出内容

| 导出 | 类型 | 用途 |
|------|------|------|
| `getEnterPlanModeToolPrompt` | `() => string` | 主入口函数，返回适当的提示内容 |

---

## 风险、边界与改进建议

### 已知风险

**1. 提示内容漂移**
- 风险：外部和内部提示可能逐渐不一致
- 示例：更新外部提示时忘记同步内部版本
- 缓解：共享 WHAT_HAPPENS_SECTION 是良好的开始

**2. 硬编码工具名称**
- 风险：`ExitPlanMode` 名称硬编码在字符串中
- 位置：WHAT_HAPPENS_SECTION 第 6 点
- 建议：使用常量引用（如 `EXIT_PLAN_MODE_TOOL_NAME`）

**3. 环境变量依赖**
- 风险：`process.env.USER_TYPE` 可能在某些上下文中未定义
- 行为：未定义时按外部用户处理（安全默认值）

### 边界条件

| 场景 | 行为 |
|------|------|
| Interview Phase 启用 | 省略 What Happens 部分 |
| USER_TYPE='ant' | 使用内部员工提示 |
| USER_TYPE=undefined | 使用外部用户提示（默认）|
| 其他 USER_TYPE 值 | 使用外部用户提示 |

### 改进建议

**1. 统一工具名称引用**
```typescript
// 当前（硬编码）
6. Exit plan mode with ExitPlanMode when ready to implement

// 建议（使用常量）
import { EXIT_PLAN_MODE_TOOL_NAME } from '../ExitPlanModeTool/constants.js'

6. Exit plan mode with ${EXIT_PLAN_MODE_TOOL_NAME} when ready to implement
```

**2. 提取共享模板**
```typescript
// 建议：提取通用模板减少重复
const BASE_PROMPT_TEMPLATE = (options: {
  opening: string
  whenToUse: string
  whenNotToUse: string
  examples: string
  whatHappens?: string
}): string => `...`
```

**3. 类型安全增强**
```typescript
// 建议：定义用户类型常量
type UserType = 'ant' | 'external'

const USER_TYPE_ANT: UserType = 'ant'

export function getEnterPlanModeToolPrompt(userType?: UserType): string {
  return userType === USER_TYPE_ANT ? /* ... */ : /* ... */
}
```

**4. 配置化提示**
```typescript
// 建议：支持从配置加载提示内容
export function getEnterPlanModeToolPrompt(): string {
  if (process.env.CLAUDE_CODE_CUSTOM_PLAN_PROMPT) {
    return process.env.CLAUDE_CODE_CUSTOM_PLAN_PROMPT
  }
  return process.env.USER_TYPE === 'ant' ? /* ... */ : /* ... */
}
```

### 测试建议

```typescript
// 需要测试的场景：
1. USER_TYPE='ant' 时返回内部员工提示
2. USER_TYPE=undefined 时返回外部用户提示
3. Interview Phase 启用时省略 What Happens
4. Interview Phase 禁用时包含 What Happens
5. 提示内容包含有效的工具名称引用
6. 提示长度在合理范围内（< 8000 tokens）
```

### 相关文件引用

```
src/tools/EnterPlanModeTool/
├── prompt.ts               # 本文件 - 提示内容定义
├── EnterPlanModeTool.ts    # 工具主逻辑，调用 getEnterPlanModeToolPrompt
├── constants.ts            # 工具名称常量
└── UI.tsx                  # UI 渲染

src/tools/AskUserQuestionTool/
└── prompt.ts               # ASK_USER_QUESTION_TOOL_NAME 定义

src/utils/planModeV2.js     # isPlanModeInterviewPhaseEnabled 实现
```

### 提示内容演进历史（推测）

基于代码结构，可以推断提示内容的演进：

1. **初始版本**：单一提示内容
2. **用户类型分化**：添加内部/外部用户区分
3. **Interview Phase**：添加功能开关支持
4. **当前状态**：两个独立的提示函数，共享 What Happens 部分

### 性能考虑

- 提示内容在每次工具调用时动态生成
- 字符串拼接操作在可接受范围内
- 如需优化，可考虑缓存结果（但环境变量可能变化）
