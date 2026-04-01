# SessionPreview.tsx 研究文档

## 场景与职责

`SessionPreview` 是一个会话预览组件，用于在会话选择器 (`LogSelector`) 中显示选定会话的详细内容预览。它支持加载精简日志的完整内容，并渲染消息列表供用户查看。

**使用场景：**
- 会话选择器 (`LogSelector`) 中的会话预览面板
- 用户浏览历史会话时查看会话详情
- 确认恢复特定会话前的预览

## 功能点目的

### 1. 会话内容预览
- 显示会话的消息历史
- 渲染完整的对话内容

### 2. 懒加载支持
- 检测精简日志 (`LiteLog`) 并自动加载完整内容
- 显示加载状态

### 3. 会话元信息展示
- 显示相对时间（如 "2 hours ago"）
- 显示消息数量
- 显示 Git 分支信息（如果有）

### 4. 交互支持
- 支持确认/取消快捷键
- 可选择恢复该会话

## 具体技术实现

### 关键数据结构

```typescript
type Props = {
  log: LogOption;              // 日志选项（可能是精简或完整）
  onExit: () => void;          // 退出回调
  onSelect: (log: LogOption) => void;  // 选择会话回调
};
```

### 懒加载逻辑

```typescript
const [fullLog, setFullLog] = React.useState<LogOption | null>(null);

React.useEffect(() => {
  setFullLog(null);  // 切换日志时重置
  if (isLiteLog(log)) {
    loadFullLog(log).then(setFullLog);  // 异步加载完整日志
  }
}, [log]);

const isLoading = isLiteLog(log) && fullLog === null;
const displayLog = fullLog ?? log;  // 优先使用完整日志
```

### 快捷键绑定

```typescript
// 取消快捷键 (Esc)
useKeybinding("confirm:no", onExit, { context: "Confirmation" });

// 确认快捷键 (Enter) - 选择会话
const handleSelect = () => onSelect(fullLog ?? log);
useKeybinding("confirm:yes", handleSelect, { context: "Confirmation" });
```

### 元信息格式化

```typescript
// 相对时间格式化
const formattedTime = formatRelativeTimeAgo(displayLog.modified);

// Git 分支后缀
const gitBranchSuffix = displayLog.gitBranch ? ` · ${displayLog.gitBranch}` : "";

// 元信息行
<Text>
  {formattedTime} · {displayLog.messageCount} messages{gitBranchSuffix}
</Text>
```

### 消息渲染

```typescript
<Messages
  messages={displayLog.messages}
  tools={tools}
  commands={[]}
  verbose={true}
  toolJSX={null}
  toolUseConfirmQueue={[]}
  inProgressToolUseIDs={new Set()}
  isMessageSelectorVisible={false}
  conversationId={conversationId}
  screen="transcript"
  streamingToolUses={[]}
  showAllInTranscript={true}
  isLoading={false}
/>
```

## 关键代码路径与文件引用

### 本文件
- `/home/sansha/Github/claude-code-instructkr/src/components/SessionPreview.tsx` - 组件实现

### 调用方
- `/home/sansha/Github/claude-code-instructkr/src/components/LogSelector.tsx` - 会话选择器（主要调用方）

### 依赖文件
- `/home/sansha/Github/claude-code-instructkr/src/utils/sessionStorage.ts` - 会话存储工具
  - `isLiteLog()` - 检测精简日志
  - `loadFullLog()` - 加载完整日志
  - `getSessionIdFromLog()` - 获取会话 ID
- `/home/sansha/Github/claude-code-instructkr/src/utils/format.ts` - 格式化工具
  - `formatRelativeTimeAgo()` - 相对时间格式化
- `/home/sansha/Github/claude-code-instructkr/src/keybindings/useKeybinding.ts` - 快捷键绑定
- `/home/sansha/Github/claude-code-instructkr/src/tools.ts` - 工具注册
  - `getAllBaseTools()` - 获取所有基础工具

### 依赖组件
- `../ink.js` - Box, Text
- `./ConfigurableShortcutHint.js` - 可配置快捷键提示
- `./design-system/Byline.js` - 元信息行组件
- `./design-system/KeyboardShortcutHint.js` - 快捷键提示
- `./design-system/LoadingState.js` - 加载状态
- `./Messages.js` - 消息列表渲染

### 类型定义
- `/home/sansha/Github/claude-code-instructkr/src/types/logs.ts` - `LogOption` 类型

## 依赖与外部交互

### 会话存储系统
- **isLiteLog**: 检测日志是否为精简格式（只包含元信息，不包含完整消息）
- **loadFullLog**: 从磁盘异步加载完整会话内容
- **getSessionIdFromLog**: 从日志对象提取会话 ID

### 快捷键系统
- 使用 "Confirmation" 上下文，优先级高于全局
- 绑定 `confirm:no`（默认 Esc）和 `confirm:yes`（默认 Enter）

### 消息渲染系统
- **Messages 组件**: 核心消息渲染组件，支持多种消息类型
- **Tools**: 传递所有基础工具定义用于工具消息渲染
- **verbose 模式**: 启用详细显示模式

### 时间格式化
- 使用 `formatRelativeTimeAgo` 显示人性化时间（如 "2 hours ago"）
- 基于 `displayLog.modified` 时间戳

## 风险、边界与改进建议

### 边界情况

1. **加载状态**: 当 `isLiteLog(log)` 为 true 且 `fullLog` 为 null 时显示加载状态
2. **空消息**: 如果 `displayLog.messages` 为空数组，Messages 组件应优雅处理
3. **会话 ID 缺失**: `getSessionIdFromLog` 可能返回空字符串，需要容错

### 潜在风险

1. **内存泄漏**: 如果组件卸载时 `loadFullLog` 仍在进行，可能导致状态更新警告
2. **竞态条件**: 快速切换会话时，旧的加载请求可能覆盖新的结果
3. **大会话加载**: 大型会话文件可能导致加载缓慢，缺少进度指示

### 改进建议

1. **添加取消机制**:
   ```typescript
   useEffect(() => {
     const abortController = new AbortController();
     if (isLiteLog(log)) {
       loadFullLog(log, abortController.signal).then(setFullLog);
     }
     return () => abortController.abort();
   }, [log]);
   ```

2. **添加加载进度**:
   - 对于大文件，显示加载进度百分比
   - 添加超时处理

3. **错误处理**:
   - 添加加载失败状态显示
   - 提供重试机制

4. **性能优化**:
   - 使用虚拟列表渲染大量消息
   - 延迟加载图片和附件

5. **UX 改进**:
   - 添加会话统计信息（工具使用次数、token 消耗等）
   - 支持消息搜索/过滤
   - 显示会话标签和自定义标题

6. **代码重构**:
   - 提取加载逻辑为独立 hook (`useFullLog`)
   - 将元信息行提取为独立组件
