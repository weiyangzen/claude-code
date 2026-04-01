# prompt.ts 深度研究文档

## 文件元数据
- **路径**: `src/tools/ExitPlanModeTool/prompt.ts`
- **大小**: 2,139 bytes
- **类型**: TypeScript 提示词定义
- **所属模块**: ExitPlanModeTool

---

## 1. 场景与职责

### 1.1 核心定位
`prompt.ts` 是 `ExitPlanModeV2Tool` 的**提示词定义文件**，负责：

1. **指导模型使用**: 告诉 Claude 何时以及如何使用 ExitPlanMode 工具
2. **明确使用边界**: 区分计划模式退出与其他工具（如 AskUserQuestion）的使用场景
3. **提供示例**: 通过具体示例帮助模型理解使用时机

### 1.2 使用场景

| 场景 | 描述 |
|------|------|
| **系统提示词组装** | 在 `ExitPlanModeV2Tool.prompt()` 方法中被调用 |
| **模型决策指导** | 帮助 Claude 判断是否应该调用此工具 |
| **避免误用** | 明确区分与 AskUserQuestion 的使用边界 |

---

## 2. 功能点目的

### 2.1 提示词结构

提示词分为以下几个部分：

#### 2.1.1 使用时机说明
```
Use this tool when you are in plan mode and have finished writing your 
plan to the plan file and are ready for user approval.
```
- **目的**: 明确工具的核心用途
- **关键条件**: 必须在 plan 模式下 + 已完成计划编写

#### 2.1.2 工具工作原理
- 计划已写入文件（工具不接收计划内容作为参数）
- 工具仅作为信号通知用户审批
- 用户会看到计划文件内容

#### 2.1.3 使用边界（重要）
```
IMPORTANT: Only use this tool when the task requires planning the 
implementation steps of a task that requires writing code. For research 
tasks where you're gathering information, searching files, reading files 
or in general trying to understand the codebase - do NOT use this tool.
```
- **目的**: 防止在研究任务中误用
- **区分标准**: 是否需要编写代码 vs 仅收集信息

#### 2.1.4 前置条件
- 计划必须完整且无歧义
- 如有未解决的问题，先使用 `AskUserQuestion`
- 明确禁止用 `AskUserQuestion` 询问"计划是否可行"

#### 2.1.5 使用示例
通过三个对比示例明确使用边界：
1. **不使用的例子**: 纯研究任务（搜索 vim mode 实现）
2. **使用的例子**: 需要编码的任务（实现 vim yank mode）
3. **先澄清后使用的例子**: 需要澄清方案的任务（认证方法选择）

---

## 3. 具体技术实现

### 3.1 代码实现

```typescript
// External stub for ExitPlanModeTool prompt - excludes Ant-only allowedPrompts section

// Hardcoded to avoid relative import issues in stub
const ASK_USER_QUESTION_TOOL_NAME = 'AskUserQuestion'

export const EXIT_PLAN_MODE_V2_TOOL_PROMPT = `Use this tool when you are in plan mode and have finished writing your plan to the plan file and are ready for user approval.

## How This Tool Works
- You should have already written your plan to the plan file specified in the plan mode system message
- This tool does NOT take the plan content as a parameter - it will read the plan from the file you wrote
- This tool simply signals that you're done planning and ready for the user to review and approve
- The user will see the contents of your plan file when they review it

## When to Use This Tool
IMPORTANT: Only use this tool when the task requires planning the implementation steps of a task that requires writing code. For research tasks where you're gathering information, searching files, reading files or in general trying to understand the codebase - do NOT use this tool.

## Before Using This Tool
Ensure your plan is complete and unambiguous:
- If you have unresolved questions about requirements or approach, use ${ASK_USER_QUESTION_TOOL_NAME} first (in earlier phases)
- Once your plan is finalized, use THIS tool to request approval

**Important:** Do NOT use ${ASK_USER_QUESTION_TOOL_NAME} to ask "Is this plan okay?" or "Should I proceed?" - that's exactly what THIS tool does. ExitPlanMode inherently requests user approval of your plan.

## Examples

1. Initial task: "Search for and understand the implementation of vim mode in the codebase" - Do not use the exit plan mode tool because you are not planning the implementation steps of a task.
2. Initial task: "Help me implement yank mode for vim" - Use the exit plan mode tool after you have finished planning the implementation steps of the task.
3. Initial task: "Add a new feature to handle user authentication" - If unsure about auth method (OAuth, JWT, etc.), use ${ASK_USER_QUESTION_TOOL_NAME} first, then use exit plan mode tool after clarifying the approach.
`
```

### 3.2 设计特点

#### 3.2.1 硬编码工具名称
```typescript
const ASK_USER_QUESTION_TOOL_NAME = 'AskUserQuestion'
```
- **原因**: 注释说明 "Hardcoded to avoid relative import issues in stub"
- **权衡**: 牺牲一点可维护性，避免复杂的导入问题

#### 3.2.2 模板字符串插值
```typescript
`...use ${ASK_USER_QUESTION_TOOL_NAME} first...`
```
- **目的**: 动态插入工具名称
- **优势**: 如果工具名称变更，只需修改常量

### 3.3 提示词设计原则

| 原则 | 体现 |
|------|------|
| **清晰性** | 每个部分都有明确的标题和说明 |
| **具体性** | 提供具体的使用示例 |
| **边界明确** | 明确区分与其他工具的使用场景 |
| **否定说明** | 明确说明"不要做什么" |

---

## 4. 关键代码路径与文件引用

### 4.1 导入位置

| 文件路径 | 导入的常量 | 用途 |
|---------|-----------|------|
| `src/tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts` | `EXIT_PLAN_MODE_V2_TOOL_PROMPT` | 在 `prompt()` 方法中返回 |

### 4.2 使用代码片段

```typescript
// ExitPlanModeV2Tool.ts
import { EXIT_PLAN_MODE_V2_TOOL_PROMPT } from './prompt.js'

export const ExitPlanModeV2Tool = buildTool({
  // ...
  async prompt() {
    return EXIT_PLAN_MODE_V2_TOOL_PROMPT
  },
  // ...
})
```

---

## 5. 依赖与外部交互

### 5.1 模块关系

```
prompt.ts
├── 导出: EXIT_PLAN_MODE_V2_TOOL_PROMPT
│   └── 被导入: ExitPlanModeV2Tool.ts
│       └── 用途: tool.prompt() 方法返回
│
└── 硬编码引用: 'AskUserQuestion'
    └── 用途: 提示词中对比说明
```

### 5.2 与其他提示词的关系

```
src/tools/
├── ExitPlanModeTool/
│   └── prompt.ts      # 退出计划模式提示词
├── EnterPlanModeTool/
│   └── prompt.ts      # 进入计划模式提示词
├── AskUserQuestionTool/
│   └── prompt.ts      # 询问用户问题提示词
└── ...
```

### 5.3 提示词协作

| 工具 | 提示词职责 | 与 ExitPlanMode 的关系 |
|------|-----------|----------------------|
| `EnterPlanMode` | 指导何时进入计划模式 | 前置条件 |
| `ExitPlanMode` | 指导何时退出计划模式 | 本文件 |
| `AskUserQuestion` | 指导何时询问用户 | 明确区分使用场景 |

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 硬编码工具名称风险
- **风险**: `ASK_USER_QUESTION_TOOL_NAME` 硬编码为 `'AskUserQuestion'`，如果实际工具名称变更会不一致
- **缓解**: 工具名称变更频率极低，且属于破坏性变更
- **改进**: 可以考虑从统一常量文件导入

#### 6.1.2 提示词长度风险
- **风险**: 提示词较长，可能占用 token 预算
- **现状**: 约 1500 字符，属于中等长度
- **缓解**: 内容必要，无法精简

### 6.2 边界情况

| 情况 | 处理 |
|------|------|
| 提示词国际化 | 当前仅支持英文，多语言支持需要重构 |
| 动态提示词 | 当前为静态字符串，不支持运行时动态生成 |
| A/B 测试 | 需要修改代码才能测试不同提示词版本 |

### 6.3 改进建议

#### 6.3.1 代码组织
```typescript
// 建议: 从统一常量导入工具名称
import { TOOL_NAMES } from '../../constants/tools.js'

const ASK_USER_QUESTION_TOOL_NAME = TOOL_NAMES.ASK_USER_QUESTION
```

#### 6.3.2 提示词模块化
```typescript
// 建议: 将提示词拆分为可组合的部分
const HOW_IT_WORKS = `...`
const WHEN_TO_USE = `...`
const EXAMPLES = `...`

export const EXIT_PLAN_MODE_V2_TOOL_PROMPT = [
  HOW_IT_WORKS,
  WHEN_TO_USE,
  EXAMPLES,
].join('\n\n')
```

#### 6.3.3 类型安全
```typescript
// 建议: 添加类型注解
export const EXIT_PLAN_MODE_V2_TOOL_PROMPT: string = `...`
```

#### 6.3.4 注释说明
```typescript
/**
 * Prompt for ExitPlanModeV2Tool.
 * 
 * Guides the model on:
 * - When to use this tool (only in plan mode, after writing plan)
 * - How the tool works (signals completion, reads from file)
 * - Boundaries vs AskUserQuestion (don't ask "is this plan ok?")
 * 
 * Note: AskUserQuestion tool name is hardcoded to avoid import issues.
 */
export const EXIT_PLAN_MODE_V2_TOOL_PROMPT = `...`
```

### 6.4 与 EnterPlanModeTool 提示词的对比

| 方面 | ExitPlanMode | EnterPlanMode |
|------|-------------|---------------|
| **触发时机** | 完成计划后 | 需要计划时 |
| **核心动作** | 请求审批 | 开始探索 |
| **与 AskUserQuestion 关系** | 明确区分 | 明确区分 |
| **示例数量** | 3 个 | 类似 |

---

## 7. 相关文件索引

### 7.1 同目录文件
- `ExitPlanModeV2Tool.ts` - 工具主逻辑，使用此提示词
- `constants.ts` - 工具名称常量
- `UI.tsx` - UI 渲染组件

### 7.2 相关提示词文件
- `src/tools/EnterPlanModeTool/prompt.ts` - 进入计划模式提示词
- `src/tools/AskUserQuestionTool/prompt.ts` - 询问用户提示词

### 7.3 工具定义
- `src/Tool.ts` - Tool 类型定义，包含 `prompt()` 方法签名
