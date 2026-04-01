# UI.tsx 研究文档

## 场景与职责

UI.tsx 是 PowerShellTool 的 React UI 渲染模块，负责在终端界面中呈现 PowerShell 命令执行的各种状态：
- 命令预览（Tool Use Message）
- 执行进度（Progress Message）
- 排队等待状态（Queued Message）
- 执行结果（Result Message）
- 错误信息（Error Message）

该模块与 BashTool 的 UI 架构类似，但针对 PowerShell 的特性进行了适配。

## 功能点目的

### 1. 命令显示截断（renderToolUseMessage）
**目的**：在非 verbose 模式下，防止过长的 PowerShell 命令占用过多终端空间。

**截断规则**：
- 最大显示行数：`MAX_COMMAND_DISPLAY_LINES = 2`
- 最大字符数：`MAX_COMMAND_DISPLAY_CHARS = 160`
- 优先按行截断，再按字符截断
- 截断后添加省略号 "…"

**安全考量**：截断仅影响显示，不影响实际执行的命令。

### 2. 进度消息渲染（renderToolUseProgressMessage）
**目的**：实时显示 PowerShell 命令的执行进度。

**数据流**：
```
PowerShellTool.execute() 
  → 生成 PowerShellProgress 数据
  → 通过 progress 回调传递给 UI
  → ShellProgressMessage 组件渲染
```

**显示内容**：
- 实时输出（最后5行或完整输出）
- 执行时间
- 总行数和总字节数
- 超时倒计时
- 后台任务标识

### 3. 结果消息渲染（renderToolResultMessage）
**目的**：展示 PowerShell 命令的最终执行结果。

**处理逻辑**：
- 图片数据检测：`isImage` 标志显示特殊提示
- stdout/stderr 分离显示
- 空输出处理：显示 "(No output)" 或后台任务提示
- 中断状态显示
- 返回码解释（如适用）

### 4. 错误消息渲染（renderToolUseErrorMessage）
**目的**：统一处理工具执行错误。

**实现**：委托给 `FallbackToolUseErrorMessage` 组件，保持与 BashTool 一致的错误处理风格。

## 具体技术实现

### 关键数据结构

```typescript
// 来自 PowerShellTool.ts
interface PowerShellToolInput {
  command: string;
  timeout?: number;
}

interface Out {
  stdout: string;
  stderr: string;
  interrupted?: boolean;
  returnCodeInterpretation?: string;
  isImage?: boolean;
  backgroundTaskId?: string;
}

// 来自 types/tools.ts
interface PowerShellProgress {
  output: string;
  fullOutput: string;
  elapsedTimeSeconds?: number;
  totalLines?: number;
  totalBytes?: number;
  timeoutMs?: number;
  taskId?: string;
}
```

### 渲染函数签名

```typescript
// 命令预览渲染
function renderToolUseMessage(
  input: Partial<PowerShellToolInput>,
  options: { verbose: boolean; theme: ThemeName }
): React.ReactNode;

// 进度消息渲染
function renderToolUseProgressMessage(
  progressMessages: ProgressMessage<PowerShellProgress>[],
  options: {
    verbose: boolean;
    tools: Tool[];
    terminalSize?: { columns: number; rows: number };
    inProgressToolCallCount?: number;
  }
): React.ReactNode;

// 结果消息渲染
function renderToolResultMessage(
  content: Out,
  progressMessages: ProgressMessage<PowerShellProgress>[],
  options: {
    verbose: boolean;
    theme: ThemeName;
    tools: Tool[];
    style?: 'condensed';
  }
): React.ReactNode;
```

### 依赖组件

| 组件 | 路径 | 用途 |
|------|------|------|
| ShellProgressMessage | ../../components/shell/ShellProgressMessage.js | 进度显示核心组件 |
| OutputLine | ../../components/shell/OutputLine.js | 输出行渲染（支持截断） |
| ShellTimeDisplay | ../../components/shell/ShellTimeDisplay.js | 执行时间/超时显示 |
| MessageResponse | ../../components/MessageResponse.js | 消息容器布局 |
| KeyboardShortcutHint | ../../components/design-system/KeyboardShortcutHint.js | 快捷键提示 |
| FallbackToolUseErrorMessage | ../../components/FallbackToolUseErrorMessage.js | 错误回退显示 |

## 关键代码路径与文件引用

### 调用链

```
1. 命令提交
   src/Tool.ts: Tool.renderToolUseMessage()
   → src/tools/PowerShellTool/UI.tsx: renderToolUseMessage()

2. 进度更新
   src/tools/PowerShellTool/PowerShellTool.ts: execute()
   → onProgress() 回调
   → src/Tool.ts: 聚合 progress messages
   → src/tools/PowerShellTool/UI.tsx: renderToolUseProgressMessage()

3. 结果展示
   src/Tool.ts: Tool.renderToolResultMessage()
   → src/tools/PowerShellTool/UI.tsx: renderToolResultMessage()
```

### 相关文件

- `src/tools/PowerShellTool/PowerShellTool.ts` - 工具主逻辑，定义 input/output 类型
- `src/components/shell/ShellProgressMessage.tsx` - 进度显示组件
- `src/components/shell/OutputLine.tsx` - 输出行处理
- `src/types/tools.ts` - PowerShellProgress 类型定义
- `src/Tool.ts` - 工具基类，定义渲染接口

## 依赖与外部交互

### 外部依赖

```typescript
// Anthropic SDK 类型
import type { ToolResultBlockParam } from '@anthropic-ai/sdk/resources/index.mjs';

// React
import * as React from 'react';

// 内部组件
import { Box, Text } from '../../ink.js';  // Ink 终端渲染库
```

### 与 PowerShellTool 的交互

UI.tsx 是 PowerShellTool 的「视图层」，通过类型导入与主逻辑耦合：

```typescript
import type { Out, PowerShellToolInput } from './PowerShellTool.js';
```

这种设计确保：
1. UI 层不依赖具体实现，只依赖类型
2. PowerShellTool 可以独立测试业务逻辑
3. UI 组件可以被其他工具复用（如 BashTool）

## 风险、边界与改进建议

### 已知风险

1. **截断信息丢失**：长命令截断可能导致用户无法看到完整命令
   - 缓解：verbose 模式可显示完整命令

2. **图片数据处理**：`isImage` 检测依赖后端正确标记
   - 如果后端未正确标记，二进制数据可能污染终端输出

3. **进度消息堆积**：高频 progress 更新可能导致渲染性能问题
   - 缓解：通过 `lastProgress = progressMessages.at(-1)` 只取最后一条

### 边界情况

| 场景 | 处理行为 |
|------|----------|
| command 为空 | 返回 null，不渲染 |
| progressMessages 为空 | 显示 "Running…" |
| stdout 和 stderr 都为空 | 显示状态提示（后台任务/中断/无输出） |
| 图片数据 | 显示 "[Image data detected and sent to Claude]" |
| 后台任务 | 显示 "Running in the background" + 快捷键提示 |

### 改进建议

1. **语法高亮**：PowerShell 命令预览可添加语法高亮，提升可读性
2. **参数折叠**：对于超长参数列表，可考虑折叠显示（如 `-Parameter ...`）
3. **执行时间预估**：基于历史数据提供 ETA 提示
4. **进度条**：对于已知总工作量的操作（如复制大文件），显示进度条

### 测试要点

- 各种长度的命令截断行为
- verbose/non-verbose 模式切换
- 空输出、仅 stderr、仅 stdout 的组合场景
- 图片数据标记的正确处理
- 后台任务状态显示
