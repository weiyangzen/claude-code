# UI.tsx 研究文档

## 场景与职责

`UI.tsx` 是 SkillTool 的**用户界面渲染层**，负责将技能工具的执行状态、结果和进度以可视化的方式呈现给用户。它基于 React + Ink（终端 UI 库）构建，提供以下核心功能：

1. **工具结果渲染** - 显示技能执行完成后的结果摘要
2. **工具使用消息渲染** - 显示技能名称和来源信息
3. **进度消息渲染** - 实时显示 forked 技能执行过程中的子代理消息
4. **拒绝/错误状态渲染** - 处理用户拒绝或执行失败的情况

该文件与 `SkillTool.ts` 解耦，通过导出纯渲染函数供主工具实现调用，符合工具架构的分离关注点原则。

## 功能点目的

### 1. 结果消息渲染 (`renderToolResultMessage`)

显示技能执行完成后的状态：

- **Forked 技能**: 显示简单的 "Done" 提示
- **内联技能**: 显示加载成功信息，包括：
  - 允许的工具数量（如配置了 `allowedTools`）
  - 模型覆盖信息（如配置了 `model`）

### 2. 工具使用消息渲染 (`renderToolUseMessage`)

在工具被调用时显示技能标识：

- 显示技能名称
- 对来自 `commands_DEPRECATED` 目录的技能添加 `/` 前缀（向后兼容）

### 3. 进度消息渲染 (`renderToolUseProgressMessage`)

Forked 技能执行期间实时显示子代理活动：

- 显示最多 `MAX_PROGRESS_MESSAGES_TO_SHOW`（3 条）最近的消息
- 在 verbose 模式下显示所有消息
- 隐藏多余消息时显示计数提示（"+N more tool uses"）
- 使用 `SubAgentProvider` 支持子代理展开/折叠交互

### 4. 拒绝/错误状态渲染

- **`renderToolUseRejectedMessage`**: 显示进度后接标准拒绝消息
- **`renderToolUseErrorMessage`**: 显示进度后接错误详情

## 具体技术实现

### 关键常量

```typescript
const MAX_PROGRESS_MESSAGES_TO_SHOW = 3;  // 非 verbose 模式最大显示消息数
const INITIALIZING_TEXT = 'Initializing…'; // 初始状态文本
```

### 渲染函数签名

```typescript
// 结果渲染
function renderToolResultMessage(output: Output): React.ReactNode

// 工具使用消息渲染（input 为部分类型，支持流式输入）
function renderToolUseMessage(
  { skill }: Partial<Input>,
  { commands }: { commands?: Command[] }
): React.ReactNode

// 进度消息渲染
function renderToolUseProgressMessage(
  progressMessages: ProgressMessage<Progress>[],
  { tools, verbose }: { tools: Tools; verbose: boolean }
): React.ReactNode

// 拒绝/错误渲染
function renderToolUseRejectedMessage(
  _input: Input,
  { progressMessagesForMessage, tools, verbose }: RenderOptions
): React.ReactNode

function renderToolUseErrorMessage(
  result: ToolResultBlockParam['content'],
  { progressMessagesForMessage, tools, verbose }: RenderOptions
): React.ReactNode
```

### 进度消息处理流程

```
renderToolUseProgressMessage(messages, {tools, verbose})
├── 空消息列表 → 显示 "Initializing…"
├── 截取消息（verbose ? 全部 : 最近 3 条）
├── 计算隐藏数量
├── 构建子代理查找表（buildSubagentLookups）
└── 渲染消息列表
    ├── SubAgentProvider（上下文提供）
    ├── MessageComponent（每条消息）
    │   ├── 高度固定 1 行
    │   ├── 溢出隐藏
    │   ├── condensed 样式
    │   └── 静态模式（无动画）
    └── 隐藏计数提示（如有）
```

### 样式设计

| 元素 | 样式 |
|-----|------|
| 初始化文本 | `dimColor`（暗淡颜色）|
| 结果消息 | `MessageResponse` + `Byline` 组件 |
| 进度消息 | `condensed` 样式，高度限制 1 行 |
| 隐藏计数 | `dimColor` + 前缀 "+" |

## 关键代码路径与文件引用

### 直接依赖

| 文件路径 | 导入内容 | 用途 |
|---------|---------|------|
| `@anthropic-ai/sdk` | `ToolResultBlockParam` | 类型定义 |
| `react` | `React` | JSX 运行时 |
| `src/components/CtrlOToExpand.ts` | `SubAgentProvider` | 子代理展开上下文 |
| `src/components/FallbackToolUseErrorMessage.ts` | `FallbackToolUseErrorMessage` | 错误回退 UI |
| `src/components/FallbackToolUseRejectedMessage.ts` | `FallbackToolUseRejectedMessage` | 拒绝回退 UI |
| `src/commands.ts` | `Command` | 命令类型 |
| `src/components/design-system/Byline.ts` | `Byline` | 署名行组件 |
| `src/components/Message.ts` | `Message` | 消息渲染组件 |
| `src/components/MessageResponse.ts` | `MessageResponse` | 消息响应容器 |
| `src/ink.ts` | `Box`, `Text` | Ink UI 基元 |
| `src/Tool.ts` | `Tools` | 工具集合类型 |
| `src/types/message.ts` | `ProgressMessage` | 进度消息类型 |
| `src/utils/messages.ts` | `buildSubagentLookups`, `EMPTY_LOOKUPS` | 子代理查找构建 |
| `src/utils/stringUtils.ts` | `plural` | 复数化工具 |
| `./SkillTool.ts` | `inputSchema`, `Output`, `Progress` | 类型导入 |

### 被调用方

在 `SkillTool.ts` 中注册为工具定义的一部分：

```typescript
export const SkillTool: Tool<InputSchema, Output, Progress> = buildTool({
  // ...
  renderToolResultMessage,
  renderToolUseMessage,
  renderToolUseProgressMessage,
  renderToolUseRejectedMessage,
  renderToolUseErrorMessage,
})
```

## 依赖与外部交互

### UI 组件依赖

1. **Ink 组件**
   - `Box`: 布局容器（flexDirection: "column"）
   - `Text`: 文本渲染（支持 `dimColor`）

2. **自定义组件**
   - `Message`: 核心消息渲染，支持多种消息类型
   - `MessageResponse`: 响应消息容器，支持高度限制
   - `Byline`: 署名行，用于结果摘要
   - `SubAgentProvider`: 提供 Ctrl+O 展开功能

3. **回退组件**
   - `FallbackToolUseErrorMessage`: 通用错误显示
   - `FallbackToolUseRejectedMessage`: 通用拒绝显示

### 类型系统

```typescript
type Input = z.infer<ReturnType<typeof inputSchema>>
// { skill: string, args?: string }

type Progress = SkillToolProgress
// 定义在 src/types/tools.js（构建时生成）
// 包含 { message, type: 'skill_progress', prompt, agentId }
```

## 风险、边界与改进建议

### 已知风险

1. **进度消息堆积**
   - Forked 技能可能产生大量进度消息
   - 当前仅显示最近 3 条，可能丢失重要上下文
   - 建议：添加展开全部功能或关键消息标记

2. **高度限制导致的截断**
   - 进度消息固定高度 1 行，`overflow: "hidden"`
   - 长消息内容被截断，用户无法查看完整信息
   - 建议：支持点击展开或悬停提示

3. **静态模式限制**
   - `shouldAnimate={false}`, `isStatic={true}`
   - 进度更新无动画，可能影响用户体验
   - 这是设计选择（避免终端过度刷新）

### 边界条件

1. **空技能名称**
   - `renderToolUseMessage` 返回 `null`
   - 工具使用消息不显示

2. **空进度消息**
   - 显示 "Initializing…" 占位
   - 避免空白区域

3. **大量隐藏消息**
   - 显示 "+N more tool uses"
   - 使用 `plural` 处理单复数

### 改进建议

1. **可访问性**
   - 添加键盘导航支持（已部分支持 Ctrl+O 展开）
   - 为进度消息添加时间戳显示选项

2. **性能优化**
   - 考虑虚拟化长消息列表
   - 当前全部渲染后 CSS 隐藏，大数据量时可能影响性能

3. **功能增强**
   - 支持进度消息过滤（按类型、工具等）
   - 添加技能执行时间显示
   - 支持导出/复制完整执行日志

4. **代码组织**
   - 渲染函数参数可提取为命名类型接口
   - 考虑将消息截断逻辑提取为可复用 Hook

5. **测试覆盖**
   - 需要测试不同消息类型的渲染
   - 测试 verbose 模式切换
   - 测试边界条件（空消息、大量消息）

### 与 SkillTool.ts 的协作

```
SkillTool.ts (逻辑层)
    │
    ├── 调用 renderToolUseMessage ───────► 显示技能名称
    │
    ├── 调用 renderToolUseProgressMessage ◄── onProgress 回调
    │   (forked 执行期间)                    实时更新子代理活动
    │
    ├── 调用 renderToolResultMessage ────► 显示执行结果
    │
    └── 调用 renderToolUseErrorMessage ──► 显示错误信息
            或 renderToolUseRejectedMessage ► 显示拒绝信息
```

UI.tsx 作为纯渲染层，不处理任何业务逻辑，所有状态通过 props 传入，便于测试和复用。
