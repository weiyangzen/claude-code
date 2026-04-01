# UI.tsx 研究文档

## 场景与职责

`UI.tsx` 是 `TeamCreateTool` 的 UI 渲染子模块，专门负责在工具被调用时向终端/REPL 界面输出一条简洁的"工具使用中"提示消息。它属于 Claude Code 工具架构中的 `renderToolUseMessage` 职责层：当模型发起 `TeamCreate` 工具调用但参数尚未完全流式到达时，UI 需要尽快给用户一个可视反馈。

## 功能点目的

- **即时反馈**：在 `team_name` 参数一旦可用时，立即在界面上显示 `create team: {team_name}`，让用户知道系统正在执行创建团队的操作。
- **保持简洁**：作为非结果性消息，它不需要展示复杂的状态或进度，只需一句话概括当前动作即可。

## 具体技术实现

### 代码结构

```tsx
import React from 'react';
import type { Input } from './TeamCreateTool.js';

export function renderToolUseMessage(input: Partial<Input>): React.ReactNode {
  return `create team: ${input.team_name}`;
}
```

### 关键细节

- **类型签名**：`input: Partial<Input>`
  - 使用 `Partial` 是因为 `renderToolUseMessage` 在参数流式传输过程中就可能被调用，此时 `team_name` 可能还未完整到达，但代码中直接模板字符串拼接，若 `team_name` 为 `undefined` 则会显示为 `create team: undefined`。
- **返回值**：直接返回一个普通字符串，React 在渲染时会将其作为文本节点处理。没有使用 JSX 标签或 `Text` 组件，保持了最小开销。
- **source map**：文件末尾包含一个内联的 base64 source map（`//# sourceMappingURL=data:application/json;...`），这是构建产物特征，不影响运行时行为。

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/tools/TeamCreateTool/UI.tsx` | 本文件，提供 `renderToolUseMessage` |
| `src/tools/TeamCreateTool/TeamCreateTool.ts` | 在 `buildTool` 配置中引用 `renderToolUseMessage` |
| `src/Tool.ts` | `Tool.renderToolUseMessage` 接口定义 |

## 依赖与外部交互

### 调用方

- **`src/tools/TeamCreateTool/TeamCreateTool.ts`**：通过 `renderToolUseMessage` 字段将其注册到工具定义中。当 REPL 或 SDK 流式渲染工具调用时，框架会调用此函数。

### 被调用方/依赖模块

- **React**：仅导入 `React` 命名空间以满足 `React.ReactNode` 类型引用；运行时实际上不需要 JSX 转换。
- **`./TeamCreateTool.js`**：导入 `Input` 类型用于类型标注。

## 风险、边界与改进建议

### 风险与边界

1. **空值显示问题**
   - 当 `input.team_name` 为 `undefined` 时，模板字符串会原样输出 `create team: undefined`。虽然这通常只在极短的流式窗口期出现，但在慢网络或模型生成极慢时可能让用户看到不专业的占位文本。

2. **无主题/颜色适配**
   - 返回值是纯字符串，没有使用主题颜色或 `figures` 图标。与某些工具（如 `BashTool`、`FileEditTool`）相比，视觉区分度较弱。

3. **无错误/拒绝态渲染**
   - 本文件只实现了 `renderToolUseMessage`，没有实现 `renderToolUseRejectedMessage` 或 `renderToolUseErrorMessage`。当创建团队失败或被用户拒绝时，框架会回退到默认的 Fallback 组件。

### 改进建议

1. **防御性空值处理**
   ```tsx
   export function renderToolUseMessage(input: Partial<Input>): React.ReactNode {
     return `create team: ${input.team_name ?? '...'}`;
   }
   ```
   或者当 `team_name` 缺失时返回 `null`，让框架跳过渲染直到参数就绪。

2. **增加视觉标识**
   - 可引入 `figures` 中的团队/用户图标（如 `👥` 或 `🐝`）前缀，提升可识别性。
   - 若产品需要，可返回 `<Text color="cyan">create team: {input.team_name}</Text>` 以使用主题色，但需评估是否值得增加复杂度。

3. **统一结果渲染（可选）**
   - 当前 `TeamCreateTool.ts` 中未定义 `renderToolResultMessage`，因此工具结果在界面上不会显示额外内容（仅显示 JSON 结果或文件持久化提示）。若产品希望创建成功后显示一行确认文本（如 `Team "my-team" created at ~/.claude/teams/my-team/config.json`），可在此模块中补充 `renderToolResultMessage` 实现。
