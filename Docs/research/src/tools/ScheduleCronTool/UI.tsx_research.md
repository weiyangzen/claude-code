# UI.tsx 深度研究文档

## 1. 场景与职责

### 1.1 定位
`UI.tsx` 是 ScheduleCronTool 模块的视图层，负责三个定时任务工具（CronCreate、CronDelete、CronList）的结果渲染。它使用 React 和 Ink 组件在终端中提供美观、一致的用户界面。

### 1.2 使用场景
- **任务创建反馈**: 显示新创建任务的 ID 和调度时间
- **任务删除确认**: 显示已删除任务的 ID
- **任务列表展示**: 以结构化方式显示所有定时任务
- **工具使用预览**: 在工具执行前显示简要信息

### 1.3 核心职责
1. 为三个工具提供 `renderToolUseMessage`（工具使用预览）
2. 为三个工具提供 `renderToolResultMessage`（结果渲染）
3. 保持与 Claude Code 整体设计系统一致
4. 处理空状态和错误状态的显示

---

## 2. 功能点目的

### 2.1 组件结构

| 函数 | 用途 | 对应工具 |
|------|------|---------|
| `renderCreateToolUseMessage` | 显示创建请求的简要信息 | CronCreateTool |
| `renderCreateResultMessage` | 显示创建成功结果 | CronCreateTool |
| `renderDeleteToolUseMessage` | 显示删除请求的 ID | CronDeleteTool |
| `renderDeleteResultMessage` | 显示删除确认 | CronDeleteTool |
| `renderListToolUseMessage` | 列表请求无预览（返回空） | CronListTool |
| `renderListResultMessage` | 显示任务列表 | CronListTool |

### 2.2 设计原则

1. **简洁性**: 只显示最关键信息（ID、调度时间）
2. **一致性**: 使用 `MessageResponse` 和 `Text` 组件保持与其他工具一致
3. **可读性**: 使用 `bold` 和 `dimColor` 区分主次信息
4. **空状态处理**: 列表为空时显示友好的提示

---

## 3. 具体技术实现

### 3.1 CronCreate 渲染

#### 3.1.1 工具使用预览

```typescript
export function renderCreateToolUseMessage(input: Partial<{
  cron: string;
  prompt: string;
}>): React.ReactNode {
  return `${input.cron ?? ''}${input.prompt ? `: ${truncate(input.prompt, 60, true)}` : ''}`;
}
```

**输出示例**:
```
0 9 * * *: Check PRs and review code...
```

- 显示 cron 表达式
- 使用 `truncate` 截断提示词至 60 字符

#### 3.1.2 结果渲染

```typescript
export function renderCreateResultMessage(output: CreateOutput): React.ReactNode {
  return <MessageResponse>
      <Text>
        Scheduled <Text bold>{output.id}</Text>{' '}
        <Text dimColor>({output.humanSchedule})</Text>
      </Text>
    </MessageResponse>;
}
```

**输出示例**:
```
⎿  Scheduled a1b2c3d4 (Every day at 9:00 AM)
```

- 使用 `MessageResponse` 包裹（提供统一的缩进和样式）
- `bold` 突出显示任务 ID
- `dimColor` 弱化显示调度描述

### 3.2 CronDelete 渲染

#### 3.2.1 工具使用预览

```typescript
export function renderDeleteToolUseMessage(input: Partial<{
  id: string;
}>): React.ReactNode {
  return input.id ?? '';
}
```

**输出示例**:
```
a1b2c3d4
```

#### 3.2.2 结果渲染

```typescript
export function renderDeleteResultMessage(output: DeleteOutput): React.ReactNode {
  return <MessageResponse>
      <Text>
        Cancelled <Text bold>{output.id}</Text>
      </Text>
    </MessageResponse>;
}
```

**输出示例**:
```
⎿  Cancelled a1b2c3d4
```

### 3.3 CronList 渲染

#### 3.3.1 工具使用预览

```typescript
export function renderListToolUseMessage(): React.ReactNode {
  return '';
}
```

列表请求无预览，直接执行。

#### 3.3.2 结果渲染

```typescript
export function renderListResultMessage(output: ListOutput): React.ReactNode {
  if (output.jobs.length === 0) {
    return <MessageResponse>
        <Text dimColor>No scheduled jobs</Text>
      </MessageResponse>;
  }
  return <MessageResponse>
      {output.jobs.map(j => <Text key={j.id}>
          <Text bold>{j.id}</Text> <Text dimColor>{j.humanSchedule}</Text>
        </Text>)}
    </MessageResponse>;
}
```

**输出示例（有任务）**:
```
⎿  a1b2c3d4 Every day at 9:00 AM
   e5f6g7h8 Weekdays at 9:00 AM
```

**输出示例（无任务）**:
```
⎿  No scheduled jobs
```

---

## 4. 关键代码路径与文件引用

### 4.1 文件结构

```
src/tools/ScheduleCronTool/UI.tsx
├── 导入
│   ├── react
│   ├── MessageResponse (组件)
│   ├── Text (Ink 组件)
│   ├── truncate (工具函数)
│   └── 类型定义 (CreateOutput, DeleteOutput, ListOutput)
├── CronCreate 渲染函数
├── CronDelete 渲染函数
├── CronList 渲染函数
└── Shared (预留扩展区域)
```

### 4.2 依赖文件

| 文件路径 | 用途 |
|---------|------|
| `src/components/MessageResponse.tsx` | 消息响应容器组件 |
| `src/ink.js` | Ink 组件（Text, Box 等） |
| `src/utils/format.js` | `truncate` 文本截断函数 |
| `src/tools/ScheduleCronTool/CronCreateTool.js` | `CreateOutput` 类型 |
| `src/tools/ScheduleCronTool/CronDeleteTool.js` | `DeleteOutput` 类型 |
| `src/tools/ScheduleCronTool/CronListTool.js` | `ListOutput` 类型 |

### 4.3 类型定义

```typescript
import type { CreateOutput } from './CronCreateTool.js';
import type { DeleteOutput } from './CronDeleteTool.js';
import type { ListOutput } from './CronListTool.js';
```

这些类型定义了各工具的输出结构，UI 组件据此渲染。

---

## 5. 依赖与外部交互

### 5.1 与 Ink 的交互

Ink 是 React 的终端渲染器，UI.tsx 使用以下 Ink 组件：

| 组件 | 用途 |
|------|------|
| `Text` | 文本渲染，支持 `bold`, `dimColor` 等样式 |
| `MessageResponse` | 统一的消息响应容器（内部使用 `Box`, `NoSelect`） |

### 5.2 与设计系统的交互

```typescript
import { MessageResponse } from '../../components/MessageResponse.js';
```

`MessageResponse` 提供：
- 统一的左侧缩进（`⎿` 符号）
- 适当的间距和布局
- 防止嵌套 `MessageResponse` 的上下文处理

### 5.3 与工具实现的交互

UI.tsx 是纯视图层，不直接调用工具逻辑：
- 工具实现导入 UI 函数并注册到 `buildTool`
- UI 函数接收工具输出作为参数
- 单向数据流：Tool → UI → Terminal

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 截断长度硬编码
```typescript
truncate(input.prompt, 60, true)  // 创建预览
truncate(j.prompt, 80, true)      // 列表输出（在 tool 中）
```
- **风险**: 不同终端宽度下可能显示不佳
- **缓解**: 当前值在典型终端宽度（80-120）下表现良好

#### 6.1.2 无错误状态渲染
- **观察**: UI.tsx 只有成功状态的渲染
- **问题**: 错误信息由 `mapToolResultToToolResultBlockParam` 处理，不在 UI.tsx
- **影响**: 错误显示风格可能与成功状态不一致

### 6.2 边界情况

| 场景 | 行为 |
|------|------|
| 创建时 prompt 为空 | 只显示 cron 表达式 |
| 创建时 cron 为空 | 显示 "undefined" |
| 删除时 id 为空 | 返回空字符串 |
| 列表为空 | 显示 "No scheduled jobs"（dimColor） |
| 列表任务很多 | 全部显示，无滚动或分页 |

### 6.3 改进建议

#### 6.3.1 响应式截断
- **建议**: 根据终端宽度动态计算截断长度
- **实现**: 使用 Ink 的 `useStdout` 获取终端尺寸

```typescript
import { useStdout } from 'ink';

function useTruncatedWidth(): number {
  const { stdout } = useStdout();
  return Math.max(40, stdout.columns - 20);
}
```

#### 6.3.2 错误状态渲染
- **建议**: 添加错误状态的 UI 渲染函数
- **实现**: 
  ```typescript
  export function renderCreateErrorMessage(error: string): React.ReactNode {
    return <MessageResponse>
        <Text color="red">Failed to schedule: {error}</Text>
      </MessageResponse>;
  }
  ```

#### 6.3.3 任务详情展开
- **建议**: 列表支持展开显示完整 prompt
- **实现**: 使用 Ink 的 `useInput` 处理交互，或显示前 N 个任务

#### 6.3.4 图标增强
- **建议**: 使用图标区分任务类型
- **实现**: 
  - 循环任务: 🔄 或 ↻
  - 一次性任务: ⏱ 或 🕐
  - Session-only: 💾 或 ○

#### 6.3.5 颜色编码
- **建议**: 使用颜色区分任务状态
- **实现**:
  - 即将执行（< 1分钟）: 黄色
  - 新创建: 绿色
  - 即将过期: 红色

### 6.4 测试建议

建议添加以下测试场景：
1. **渲染测试**: 各函数的输出快照测试
2. **边界测试**: 空输入、长输入、特殊字符
3. **交互测试**: 如果有交互功能，测试用户输入处理
4. **主题测试**: 在不同主题下验证颜色对比度

---

## 7. 附录

### 7.1 完整输出示例

**创建任务**:
```
┌─────────────────────────────────────┐
│ CronCreate 0 9 * * *: Check PRs...  │
├─────────────────────────────────────┤
│ ⎿  Scheduled a1b2c3d4               │
│       (Every day at 9:00 AM)        │
└─────────────────────────────────────┘
```

**删除任务**:
```
┌─────────────────────────────────────┐
│ CronDelete a1b2c3d4                 │
├─────────────────────────────────────┤
│ ⎿  Cancelled a1b2c3d4               │
└─────────────────────────────────────┘
```

**列出任务**:
```
┌─────────────────────────────────────┐
│ CronList                            │
├─────────────────────────────────────┤
│ ⎿  a1b2c3d4 Every day at 9:00 AM    │
│    e5f6g7h8 Weekdays at 9:00 AM     │
└─────────────────────────────────────┘
```

### 7.2 相关文件

| 文件 | 关系 |
|------|------|
| `CronCreateTool.ts` | 使用 `renderCreateToolUseMessage`, `renderCreateResultMessage` |
| `CronDeleteTool.ts` | 使用 `renderDeleteToolUseMessage`, `renderDeleteResultMessage` |
| `CronListTool.ts` | 使用 `renderListToolUseMessage`, `renderListResultMessage` |

### 7.3 扩展预留

文件末尾的 `// --- Shared -----------------------------------------------------------------` 注释预留了共享组件的扩展位置，可添加：
- 共享的样式常量
- 通用的渲染辅助函数
- 任务状态图标组件
