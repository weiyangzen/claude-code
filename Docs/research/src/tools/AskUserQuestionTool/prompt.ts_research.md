# prompt.ts 研究文档

## 场景与职责

`prompt.ts` 是 `AskUserQuestionTool` 的**纯常量配置模块**，职责单一且明确：集中管理该工具在模型提示（system prompt / tool description）中所需的所有文本常量。它不包含任何运行时逻辑或 UI 代码，仅通过导出字符串和字典对象，为 `AskUserQuestionTool.tsx` 及其他引用方提供权威的文案来源。

该文件处于工具目录的"文案层"，与"实现层"（`AskUserQuestionTool.tsx`）和"渲染层"（`AskUserQuestionPermissionRequest/` 下的各组件）形成清晰的分层边界。

---

## 功能点目的

| 导出项 | 功能目的 |
|--------|----------|
| `ASK_USER_QUESTION_TOOL_NAME` | 定义工具在 API/内部注册表中的唯一标识符（`'AskUserQuestion'`），作为跨文件引用的权威名称常量 |
| `ASK_USER_QUESTION_TOOL_CHIP_WIDTH` | 定义问题 `header` 字段的最大显示宽度（12 字符），供 schema 描述和 UI 布局共享同一约束源 |
| `DESCRIPTION` | 提供给 `buildTool.description()` 的单行能力摘要，用于工具列表和 deferred loading 描述 |
| `PREVIEW_FEATURE_PROMPT` | 根据 `markdown` / `html` 两种格式，分别向模型说明 `preview` 字段的用途、内容规范和限制 |
| `ASK_USER_QUESTION_TOOL_PROMPT` | 工具的核心 system prompt 文本，包含使用场景、用法说明、与 `ExitPlanModeTool` 的边界约定 |

---

## 具体技术实现

### 1. 模块结构与导出清单

文件共导出 5 个顶层符号，全部为 `const` 常量：

```ts
// 工具身份标识
ASK_USER_QUESTION_TOOL_NAME    // string = 'AskUserQuestion'
ASK_USER_QUESTION_TOOL_CHIP_WIDTH // number = 12

// 简短描述（用于工具元数据）
DESCRIPTION                    // string

// Preview 功能补充说明（按格式分键）
PREVIEW_FEATURE_PROMPT         // { markdown: string, html: string }

// 主提示文本
ASK_USER_QUESTION_TOOL_PROMPT  // string (template literal)
```

### 2. 工具名称常量

```ts
export const ASK_USER_QUESTION_TOOL_NAME = 'AskUserQuestion'
```

该名称被以下场景引用：
- `AskUserQuestionTool.tsx` 中作为 `buildTool({ name: ... })` 的注册名
- `src/tools.ts` 中无显式引用（直接导入对象），但 `src/constants/tools.ts`、`src/utils/permissions/classifierDecision.ts`、`src/utils/messages.ts` 等文件通过导入该常量进行字符串匹配
- `src/tools/EnterPlanModeTool/prompt.ts` 在 EnterPlanMode 的 prompt 中嵌入该名称，指导模型在计划模式中使用哪个工具澄清问题
- `src/skills/bundled/batch.ts`、`src/skills/bundled/scheduleRemoteAgents.ts` 等 skill prompt 中引用

### 3. CHIP 宽度常量

```ts
export const ASK_USER_QUESTION_TOOL_CHIP_WIDTH = 12
```

该数值的引用链路：
- `AskUserQuestionTool.tsx` 行 21：在 `questionSchema.header` 的 `.describe()` 中动态嵌入，作为模型生成输入时的约束说明
- `QuestionNavigationBar.tsx` 行 39：UI 渲染时以 `header` 或 `Q${index + 1}` 作为 tab 标签，实际截断逻辑由终端宽度决定，但 12 字符是 schema 层面的软性约束来源

### 4. DESCRIPTION

```ts
export const DESCRIPTION =
  'Asks the user multiple choice questions to gather information, clarify ambiguity, understand preferences, make decisions or offer them choices.'
```

使用位置：
- `AskUserQuestionTool.tsx` 行 114-116：`async description() { return DESCRIPTION; }`
- 该返回值用于 `ToolSearch` 的关键词匹配、工具列表展示以及非交互式会话中的能力描述

### 5. PREVIEW_FEATURE_PROMPT

```ts
export const PREVIEW_FEATURE_PROMPT = {
  markdown: `...`,
  html: `...`,
} as const
```

这是一个按预览格式分发的字典对象，在 `AskUserQuestionTool.tsx` 的 `prompt()` 方法中被动态拼接：

```ts
async prompt() {
  const format = getQuestionPreviewFormat()
  if (format === undefined) {
    return ASK_USER_QUESTION_TOOL_PROMPT
  }
  return ASK_USER_QUESTION_TOOL_PROMPT + PREVIEW_FEATURE_PROMPT[format]
}
```

#### markdown 变体要点
- 适用场景：ASCII mockups、代码片段、图表变体、配置示例
- 渲染环境：CLI 中的等宽字体框（`PreviewBox.tsx`）
- 约束：支持多行文本和换行；当任意选项包含 preview 时，UI 自动切换为左右分栏布局
- 限制：**preview 仅支持单选（不支持 `multiSelect`）**

#### html 变体要点
- 适用场景：HTML mockups、格式化代码片段、视觉对比
- 约束：必须是自包含的 HTML fragment，禁止 `<html>` / `<body>` 包装
- 安全限制：禁止 `<script>` / `<style>` 标签，要求使用 inline `style` 属性
- 同样限制：**仅支持单选**

### 6. ASK_USER_QUESTION_TOOL_PROMPT

这是该工具最核心的 prompt 文本，采用模板字符串（template literal）编写，结构如下：

1. **使用时机**：明确告诉模型在需要收集偏好、澄清歧义、获取决策或提供选择时使用该工具
2. **用法说明（Usage notes）**：
   - 用户始终可以选择 "Other" 提供自定义文本输入
   - 使用 `multiSelect: true` 允许多选
   - 若推荐某个选项，应将其列为第一个并追加 `"(Recommended)"`
3. **Plan mode note**：
   - 在计划模式下，使用该工具在最终确定计划**之前**澄清需求或选择方案
   - **禁止**用该工具询问 "Is my plan ready?" 或 "Should I proceed?"
   - 此类计划审批应使用 `ExitPlanModeTool`（通过导入 `EXIT_PLAN_MODE_TOOL_NAME` 动态嵌入名称）
   - **禁止**在问题中提及 "the plan"，因为用户在 UI 中看不到计划内容，直到 `ExitPlanModeTool` 被调用

#### 动态名称注入
```ts
import { EXIT_PLAN_MODE_TOOL_NAME } from '../ExitPlanModeTool/constants.js'
// ...
`use ${EXIT_PLAN_MODE_TOOL_NAME} for plan approval`
```

这种跨工具引用确保了：如果 `ExitPlanModeTool` 被重命名，AskUserQuestion 的 prompt 会自动同步，避免文案与代码脱节。

---

## 关键代码路径与文件引用

### 本文件内部
- **行 1**：从 `ExitPlanModeTool` 导入 `EXIT_PLAN_MODE_TOOL_NAME`，实现跨工具名称解耦
- **行 3-5**：基础身份/尺寸常量
- **行 7-8**：`DESCRIPTION`
- **行 10-30**：`PREVIEW_FEATURE_PROMPT`（markdown / html 双版本）
- **行 32-44**：`ASK_USER_QUESTION_TOOL_PROMPT`（主提示文本）

### 直接调用方

| 文件 | 引用符号 | 用途 |
|------|----------|------|
| `src/tools/AskUserQuestionTool/AskUserQuestionTool.tsx` | `ASK_USER_QUESTION_TOOL_NAME` | `buildTool.name` |
| | `ASK_USER_QUESTION_TOOL_CHIP_WIDTH` | schema 描述中的字符上限 |
| | `DESCRIPTION` | `description()` 返回值 |
| | `PREVIEW_FEATURE_PROMPT` | `prompt()` 动态拼接 |
| | `ASK_USER_QUESTION_TOOL_PROMPT` | `prompt()` 基础文本 |
| `src/constants/tools.ts` | `ASK_USER_QUESTION_TOOL_NAME` | `ALL_AGENT_DISALLOWED_TOOLS` |
| `src/utils/permissions/classifierDecision.ts` | `ASK_USER_QUESTION_TOOL_NAME` | `SAFE_YOLO_ALLOWLISTED_TOOLS` |
| `src/utils/messages.ts` | `ASK_USER_QUESTion_TOOL_NAME` | 消息处理逻辑中的工具名匹配 |
| `src/constants/prompts.ts` | `ASK_USER_QUESTION_TOOL_NAME` | 检测工具启用状态 |
| `src/components/tasks/RemoteSessionDetailDialog.tsx` | `ASK_USER_QUESTION_TOOL_NAME` | 远程会话详情中的工具名判断 |
| `src/skills/bundled/batch.ts` | `ASK_USER_QUESTION_TOOL_NAME` | skill prompt 模板嵌入 |
| `src/skills/bundled/scheduleRemoteAgents.ts` | `ASK_USER_QUESTION_TOOL_NAME` | skill prompt 模板嵌入 |
| `src/tools/EnterPlanModeTool/prompt.ts` | `ASK_USER_QUESTION_TOOL_NAME` | EnterPlanMode prompt 中引用 |

### 无反向依赖

该文件**不导入**任何项目内部的运行时模块（除 `EXIT_PLAN_MODE_TOOL_NAME` 外），因此：
- 编译/打包时无循环依赖风险
- 可被安全地用于 skill prompt、常量表、测试 fixture 等场景
- 修改文案不会影响类型签名或运行时逻辑（除非修改了 `ASK_USER_QUESTION_TOOL_NAME` 导致字符串匹配失效）

---

## 依赖与外部交互

### 编译时依赖
- **TypeScript / Bun**：作为 `.ts` 文件，依赖项目的 Bun 构建管道
- **无框架依赖**：不依赖 React、Ink、Zod 等运行时库

### 唯一的外部导入
```ts
import { EXIT_PLAN_MODE_TOOL_NAME } from '../ExitPlanModeTool/constants.js'
```

这是一个"横向同级目录"导入，遵循项目中"每个工具的常量放在各自目录的 `prompt.ts` 或 `constants.ts` 中"的约定。

### 与 `getQuestionPreviewFormat()` 的间接交互

`PREVIEW_FEATURE_PROMPT` 的内容虽然定义在本文件，但**是否被注入到模型上下文**由 `AskUserQuestionTool.tsx` 的 `prompt()` 方法根据 `getQuestionPreviewFormat()` 的返回值决定：
- `undefined` → 不注入 preview 说明（SDK 消费者未启用该功能）
- `'markdown'` → 注入 markdown 变体
- `'html'` → 注入 html 变体

这意味着本文件是"静态文案仓库"，而 `AskUserQuestionTool.tsx` 是"动态分发器"。

---

## 风险、边界与改进建议

### 风险与边界

1. **名称变更的级联影响**
   - `ASK_USER_QUESTION_TOOL_NAME` 被近 10 个文件直接引用。若重命名该常量但未同步更新所有引用点（尤其是 `src/constants/tools.ts`、`src/utils/permissions/classifierDecision.ts` 等集合中的字符串），会导致工具权限判断、代理过滤、自动模式分类等核心逻辑失效。
   - 当前项目没有针对常量重命名的自动化重构测试（如 snapshot 测试），依赖开发者手动全局替换。

2. **Prompt 文本的硬编码与可维护性**
   - `ASK_USER_QUESTION_TOOL_PROMPT` 和 `PREVIEW_FEATURE_PROMPT` 均为硬编码英文长文本，直接写在 TS 文件中。随着功能迭代，这些文本经历了多次追加（如 plan mode note、preview feature），导致单文件行数不多但信息密度极高。
   - 当前没有版本化的 prompt 管理或 A/B 测试框架，任何文案调整都需要发版全量生效。

3. **HTML 安全说明的"免责声明"模式**
   - `PREVIEW_FEATURE_PROMPT.html` 中明确告诉模型："no `<script>` or `<style>` tags — use inline style attributes instead"。然而这只是一条 prompt 层面的软性约束，真正的校验在 `AskUserQuestionTool.tsx` 的 `validateHtmlPreview()` 中通过正则完成。模型可能因 prompt 理解偏差或故意绕过而生成违规 HTML。
   - 更严重的是，prompt 中未提及 inline event handlers 的风险，而 `validateHtmlPreview` 也不检查它们。

4. **CHIP_WIDTH 的语义漂移风险**
   - `ASK_USER_QUESTION_TOOL_CHIP_WIDTH = 12` 被同时用于：
     a) Zod schema 的 `.describe()`（软性约束，模型可能不严格遵守）
     b) `QuestionNavigationBar.tsx` 的 tab 截断逻辑（UI 硬性约束）
   - 如果未来 UI 设计放宽了 header 宽度，但忘记同步更新 schema 描述，会导致模型生成的 header 被终端截断，用户看到的标签不完整。

5. **跨工具引用的脆弱性**
   - 文件导入 `EXIT_PLAN_MODE_TOOL_NAME` 来避免硬编码 `"ExitPlanMode"`，但如果 `ExitPlanModeTool` 的目录结构或导出文件名发生变化（如从 `constants.js` 改为 `prompt.ts`），本文件的编译会直接失败。
   - 这种跨工具耦合虽然比硬编码字符串好，但仍属于编译时强依赖。

### 改进建议

1. **引入 Prompt 版本化或配置化机制**
   - 将 `ASK_USER_QUESTION_TOOL_PROMPT` 和 `PREVIEW_FEATURE_PROMPT` 提取到 YAML/JSON 配置文件中，支持按环境（ant / external）或按实验组加载不同版本，便于非工程师角色（PM、UX writer）迭代文案。

2. **增强 HTML Preview 的 prompt 安全说明**
   - 在 `PREVIEW_FEATURE_PROMPT.html` 中补充对内联事件处理器的禁止说明，例如：
     > "Do not use inline event handlers such as `onclick` or `onload`."
   - 这可以与 `validateHtmlPreview` 的校验逻辑形成上下呼应，降低模型违规概率。

3. **统一 CHIP_WIDTH 的契约校验**
   - 在 `QuestionNavigationBar.tsx` 或相关 UI 组件中显式导入 `ASK_USER_QUESTION_TOOL_CHIP_WIDTH`，而不是使用魔法数字，确保 schema 约束与 UI 约束始终同源。
   - 可考虑在 `questionSchema` 的 `.refine()` 中加入对 `header` 长度的硬性校验（通过 `stringWidth` 或简单 `length`），而不仅依赖 `.describe()` 的软性提示。

4. **补充类型安全网**
   - 为 `PREVIEW_FEATURE_PROMPT` 显式标注类型：
     ```ts
     export const PREVIEW_FEATURE_PROMPT: Record<'markdown' | 'html', string> = { ... }
     ```
     目前使用 `as const` 已经提供了较好的字面量类型推断，但显式类型可以进一步防止键名拼写错误。

5. **考虑将常量与 prompt 分离为两个文件**
   - 当前 `prompt.ts` 混合了"机器标识常量"（`NAME`、`CHIP_WIDTH`）和"人类可读文案"（`DESCRIPTION`、`PROMPT`）。
   - 若拆分为 `constants.ts`（NAME、CHIP_WIDTH）和 `prompt.ts`（DESCRIPTION、PROMPT、PREVIEW），可减少引用方的认知负担：例如 `src/constants/tools.ts` 只需导入 `NAME`，不必感知大段 prompt 文本的存在。

6. **增加文案变更的回归测试**
   - 为 `ASK_USER_QUESTION_TOOL_PROMPT` 添加 snapshot 测试，确保关键段落（如 plan mode note、preview 限制）不会被意外删除或篡改。
   - 特别保护 `${EXIT_PLAN_MODE_TOOL_NAME}` 模板插值的存在性，防止未来重构时不小心将其静态化。
