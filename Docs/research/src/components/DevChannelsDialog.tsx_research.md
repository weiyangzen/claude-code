# DevChannelsDialog.tsx 研究文档

## 场景与职责

`DevChannelsDialog.tsx` 是 Claude Code CLI 中用于**开发频道安全警告**的模态对话框组件。当用户使用 `--dangerously-load-development-channels` 参数加载开发频道时，系统会显示此对话框，警告用户该功能的安全风险并获取确认。

### 核心职责
1. **安全警告**：明确告知用户加载开发频道的风险
2. **教育引导**：指导用户使用更安全的 `--channels` 选项
3. **确认交互**：强制用户明确选择"用于本地开发"或"退出"
4. **频道展示**：列出即将加载的开发频道供用户确认

## 功能点目的

### 1. 安全风险警告
- **警告标题**："WARNING: Loading development channels"
- **警告颜色**：`error`（红色）
- **警告内容**：
  - `--dangerously-load-development-channels` 仅用于本地频道开发
  - 不要使用此选项运行从互联网下载的频道
  - 建议使用 `--channels` 运行已批准的频道列表

### 2. 频道信息展示
- **展示内容**：列出所有待加载的开发频道
- **格式**：
  - Plugin 频道：`plugin:{name}@{marketplace}`
  - Server 频道：`server:{name}`
- **样式**：使用 `dimColor` 显示，作为辅助信息

### 3. 用户确认选项
- **选项 1**："I am using this for local development"（接受）
  - 值：`accept`
  - 行为：调用 `onAccept()` 回调，继续加载
- **选项 2**："Exit"（退出）
  - 值：`exit`
  - 行为：调用 `gracefulShutdownSync(1)`，以错误码退出

### 4. Escape 键处理
- **行为**：按 Escape 键等同于选择退出
- **实现**：通过 `handleEscape` 回调调用 `gracefulShutdownSync(0)`
- **注意**：使用 `useCallback` 缓存回调函数

## 具体技术实现

### 组件接口

```typescript
import type { ChannelEntry } from '../bootstrap/state.js';

type Props = {
  channels: ChannelEntry[];  // 待加载的开发频道列表
  onAccept(): void;          // 用户接受回调
};

export function DevChannelsDialog({ channels, onAccept }: Props): React.ReactNode
```

### ChannelEntry 类型

```typescript
// 来自 bootstrap/state.ts
type ChannelEntry =
  | { kind: 'plugin'; name: string; marketplace: string; dev?: boolean }
  | { kind: 'server'; name: string; dev?: boolean };
```

### 组件结构

```tsx
<Dialog
  title="WARNING: Loading development channels"
  color="error"
  onCancel={handleEscape}
>
  <Box flexDirection="column" gap={1}>
    <Text>
      --dangerously-load-development-channels is for local channel development only...
    </Text>
    <Text>
      Please use --channels to run a list of approved channels.
    </Text>
    <Text dimColor={true}>
      Channels: {channels.map(formatChannel).join(', ')}
    </Text>
  </Box>
  <Select
    options={[
      { label: "I am using this for local development", value: "accept" },
      { label: "Exit", value: "exit" }
    ]}
    onChange={handleChange}
  />
</Dialog>
```

### 频道格式化

```typescript
function formatChannel(c: ChannelEntry): string {
  return c.kind === 'plugin' 
    ? `plugin:${c.name}@${c.marketplace}` 
    : `server:${c.name}`;
}
```

### 选择处理逻辑

```typescript
function handleChange(value: 'accept' | 'exit'): void {
  switch (value) {
    case 'accept':
      onAccept();
      break;
    case 'exit':
      gracefulShutdownSync(1);
      break;
  }
}
```

### React Compiler 优化

- `$[0-1]`：缓存 `handleChange` 回调（依赖 `onAccept`）
- `$[2-3]`：缓存静态警告文本
- `$[4-5]`：缓存频道列表格式化结果（依赖 `channels`）
- `$[6-7]`：缓存包含频道列表的 Box
- `$[8]`：缓存静态 options 数组
- `$[9-10]`：缓存 Select 组件（依赖 `handleChange`）
- `$[11-13]`：缓存 Dialog 组件

## 关键代码路径与文件引用

### 当前文件
- `/home/sansha/Github/claude-code-instructkr/src/components/DevChannelsDialog.tsx`

### 直接依赖
| 导入路径 | 用途 |
|---------|------|
| `react` | React 核心 API（useCallback） |
| `../bootstrap/state.js` | `ChannelEntry` 类型 |
| `../ink.js` | Ink UI 组件（Box, Text） |
| `../utils/gracefulShutdown.js` | `gracefulShutdownSync()` |
| `./CustomSelect/index.js` | Select 组件 |
| `./design-system/Dialog.js` | Dialog 容器组件 |

### 相关依赖文件

#### bootstrap/state.ts (`/home/sansha/Github/claude-code-instructkr/src/bootstrap/state.ts`)

**ChannelEntry 类型定义**：
```typescript
// dev: true on entries that came via --dangerously-load-development-channels.
export type ChannelEntry =
  | { kind: 'plugin'; name: string; marketplace: string; dev?: boolean }
  | { kind: 'server'; name: string; dev?: boolean };
```

**相关 State**：
```typescript
allowedChannels: ChannelEntry[];      // 允许的频道列表
hasDevChannels: boolean;              // 是否有开发频道
```

#### gracefulShutdown.ts (`/home/sansha/Github/claude-code-instructkr/src/utils/gracefulShutdown.ts`)

```typescript
export function gracefulShutdownSync(
  exitCode = 0,
  reason: ExitReason = 'other'
): void;
```

**同步关闭流程**：
1. 设置 `process.exitCode`
2. 调用异步 `gracefulShutdown`
3. 处理错误并强制退出

#### Dialog 组件 (`/home/sansha/Github/claude-code-instructkr/src/components/design-system/Dialog.tsx`)

支持通过 `color` 属性设置主题色：
```typescript
color?: keyof Theme;  // 此处使用 "error"
```

#### CustomSelect 组件 (`/home/sansha/Github/claude-code-instructkr/src/components/CustomSelect/index.ts`)

```typescript
interface Option {
  label: string;
  value: string;
}

interface SelectProps {
  options: Option[];
  onChange: (value: string) => void;
}
```

## 依赖与外部交互

### 调用方

该对话框由 CLI 启动流程调用，当检测到 `--dangerously-load-development-channels` 参数时显示：

```typescript
// 伪代码
if (args['dangerously-load-development-channels']) {
  const channels = parseDevChannels(args);
  return (
    <DevChannelsDialog
      channels={channels}
      onAccept={() => continueStartup(channels)}
    />
  );
}
```

### 与启动流程的交互

1. **阻塞启动**：对话框显示时，CLI 启动流程暂停
2. **分支处理**：
   - 用户接受 → 继续加载开发频道 → 正常启动
   - 用户退出 → 调用 `gracefulShutdownSync(1)` → 进程退出

### 安全考虑

1. **强制确认**：用户必须明确选择，不能忽略
2. **红色警告**：使用 error 颜色强调风险
3. **退出选项**：始终提供安全的退出路径
4. **频道透明**：明确展示将要加载的频道列表

## 风险、边界与改进建议

### 已知风险

1. **用户习惯性点击**
   - 风险：用户可能不看警告内容直接点击接受
   - 缓解：使用红色标题、强制二选一、明确的风险描述
   - 建议：考虑添加倒计时或确认输入

2. **频道列表过长**
   - 风险：如果频道很多，显示可能超出屏幕
   - 现状：使用逗号分隔的纯文本展示
   - 建议：考虑使用可滚动列表或折叠显示

3. **Escape 键行为**
   - 风险：用户可能误按 Escape 期望取消，但会退出程序
   - 现状：Escape 调用 `gracefulShutdownSync(0)`
   - 建议：考虑 Escape 只是关闭对话框而不退出

### 边界情况

1. **空频道列表**
   - 如果 `channels` 为空数组，显示 "Channels: "
   - 建议：添加空列表处理

2. **特殊字符频道名**
   - 频道名包含特殊字符可能影响显示
   - 现状：直接拼接字符串，无转义处理

3. **终端宽度不足**
   - 长频道列表在窄终端中可能换行混乱
   - 建议：添加换行或截断处理

4. **快速连续调用**
   - 如果启动逻辑错误，可能重复显示对话框
   - 建议：添加状态检查防止重复显示

### 改进建议

1. **更详细的频道信息**
   - 显示频道的来源路径或 URL
   - 显示频道的版本或哈希
   - 帮助用户确认频道内容

2. **安全提示增强**
   - 添加具体的攻击场景示例
   - 链接到安全文档
   - 显示上次验证时间（如果有）

3. **频道验证状态**
   - 显示频道是否经过签名验证
   - 标记未验证的频道

4. **批量操作**
   - 允许用户选择性地接受/拒绝某些频道
   - 而不是全有或全无

5. **记住选择**
   - 对于重复启动，提供"记住我的选择"选项
   - 减少重复确认

6. **日志记录**
   - 记录用户的选择用于审计
   - 帮助追踪潜在的安全事件

### 测试建议

1. **单元测试**
   - 测试 `handleChange` 逻辑
   - 测试频道格式化函数
   - 测试 Escape 键处理

2. **集成测试**
   - 测试与启动流程的集成
   - 测试与 gracefulShutdown 的集成

3. **视觉测试**
   - 验证长频道列表的显示
   - 验证不同终端尺寸下的布局

4. **安全测试**
   - 验证对话框无法被绕过
   - 验证退出功能正常工作
