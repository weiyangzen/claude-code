# IdeStatusIndicator.tsx 深度研究文档

## 场景与职责

`IdeStatusIndicator` 是一个轻量级的状态指示组件，用于在 Claude Code 的 UI 中显示当前 IDE 的连接状态和用户在 IDE 中的文本选择信息。该组件负责：

1. **IDE 连接状态可视化**：显示 IDE 集成是否已连接
2. **选中文本指示**：显示用户在 IDE 中选中的行数和文本内容
3. **当前文件指示**：显示用户当前在 IDE 中打开的文件名
4. **上下文感知**：仅在 IDE 连接且有有效选择时显示

## 功能点目的

### 1. IDE 选择状态指示
- **目的**：让用户知道 Claude Code 已感知到 IDE 中的文本选择
- **显示内容**：
  - 选中行数（如 "⧉ 5 lines selected"）
  - 选中单行时显示 "line"，多行显示 "lines"
  - 仅文件名（如 "⧉ In filename.ts"）

### 2. 连接状态检查
- **目的**：确保只在 IDE 真正连接且有有效选择时显示指示器
- **条件**：
  - `ideStatus === "connected"`
  - 有 `filePath` 或 (`text` 且 `lineCount > 0`)

### 3. 智能显示逻辑
- **优先级**：
  1. 如果有选中文本和行数，显示行数信息
  2. 如果只有文件路径，显示文件名
  3. 其他情况不显示

## 具体技术实现

### 关键数据结构

```typescript
// 组件 Props
type IdeStatusIndicatorProps = {
  ideSelection: IDESelection | undefined;  // IDE 选择信息
  mcpClients?: MCPServerConnection[];      // MCP 客户端连接列表
};

// IDE 选择信息（来自 src/hooks/useIdeSelection.ts）
export type IDESelection = {
  lineCount: number;      // 选中的行数
  lineStart?: number;     // 起始行号
  text?: string;          // 选中的文本内容
  filePath?: string;      // 文件路径
};

// IDE 连接状态（来自 src/hooks/useIdeConnectionStatus.ts）
export type IdeStatus = 'connected' | 'disconnected' | 'pending' | null;
```

### 关键流程

1. **状态获取流程**：
   ```
   1. 通过 useIdeConnectionStatus(mcpClients) 获取 ideStatus
   2. 检查 shouldShowIdeSelection 条件
   3. 如果 ideSelection.text 和 lineCount 存在，显示行数
   4. 否则如果 ideSelection.filePath 存在，显示文件名
   5. 否则返回 null（不显示）
   ```

2. **显示条件判断**：
   ```typescript
   const shouldShowIdeSelection = 
     ideStatus === 'connected' && 
     (ideSelection?.filePath || 
      (ideSelection?.text && ideSelection.lineCount > 0));
   ```

### 渲染优化

组件使用 React Compiler 进行优化，通过 `$` 缓存对象实现记忆化：

```typescript
const $ = _c(7);  // 7 个缓存槽位

// 条件缓存示例
if ($[0] !== ideSelection.lineCount || $[1] !== t1) {
  t2 = <Text color="ide" key="selection-indicator" wrap="truncate">
    ⧉ {ideSelection.lineCount}{" "}{t1} selected
  </Text>;
  $[0] = ideSelection.lineCount;
  $[1] = t1;
  $[2] = t2;
} else {
  t2 = $[2];
}
```

## 关键代码路径与文件引用

### 本文件关键代码

```typescript
export function IdeStatusIndicator({
  ideSelection,
  mcpClients,
}: IdeStatusIndicatorProps): React.ReactNode {
  const { status: ideStatus } = useIdeConnectionStatus(mcpClients);

  // 检查是否应该显示 IDE 选择指示器
  const shouldShowIdeSelection =
    ideStatus === 'connected' &&
    (ideSelection?.filePath ||
      (ideSelection?.text && ideSelection.lineCount > 0));

  if (ideStatus === null || !shouldShowIdeSelection || !ideSelection) {
    return null;
  }

  // 显示选中的行数
  if (ideSelection.text && ideSelection.lineCount > 0) {
    const label = ideSelection.lineCount === 1 ? 'line' : 'lines';
    return (
      <Text color="ide" key="selection-indicator" wrap="truncate">
        ⧉ {ideSelection.lineCount} {label} selected
      </Text>
    );
  }

  // 显示当前文件
  if (ideSelection.filePath) {
    const filename = basename(ideSelection.filePath);
    return (
      <Text color="ide" key="selection-indicator" wrap="truncate">
        ⧉ In {filename}
      </Text>
    );
  }
}
```

### 依赖文件

| 文件路径 | 用途 |
|---------|------|
| `src/hooks/useIdeConnectionStatus.ts` | 获取 IDE MCP 连接状态 |
| `src/hooks/useIdeSelection.ts` | IDE 文本选择钩子（类型定义） |
| `src/services/mcp/types.ts` | MCP 连接类型定义 |
| `src/ink.tsx` | Ink 渲染组件 (`Text`) |

### 调用方

- `src/hooks/notifs/useIDEStatusIndicator.tsx` - 在状态栏中显示 IDE 指示器
- 可能集成在 PromptInputFooter 或其他状态显示区域

## 依赖与外部交互

### 外部依赖

1. **React Compiler Runtime**：使用 `_c` 函数进行编译时优化
2. **Node.js path**：使用 `basename` 提取文件名
3. **Ink**：终端 UI 渲染库

### 内部服务交互

1. **MCP 连接系统**：
   - 通过 `useIdeConnectionStatus` 监听 IDE MCP 客户端状态
   - 检查 `mcpClients` 中是否存在名为 'ide' 的已连接客户端

2. **IDE 选择系统**：
   - 接收来自 `useIdeSelection` 的选择数据
   - 数据通过 MCP 通知从 IDE 扩展实时推送

### 数据流

```
IDE 扩展 → MCP 通知 (selection_changed) → useIdeSelection hook → 
IdeStatusIndicator props → 条件渲染
```

## 风险、边界与改进建议

### 潜在风险

1. **文件名截断问题**：
   - 使用 `wrap="truncate"` 可能导致长文件名显示不完整
   - 建议：添加悬停提示或点击展开功能

2. **状态同步延迟**：
   - MCP 通知可能有延迟，导致选择状态与实际 IDE 不同步
   - 建议：添加心跳检测或状态超时机制

3. **多 IDE 连接歧义**：
   - 如果有多个 IDE MCP 客户端，`useIdeConnectionStatus` 返回第一个匹配的
   - 风险：用户可能同时在多个 IDE 中工作，状态显示可能混乱

### 边界情况

1. **lineCount 为 0**：
   - 当 `lineCount === 0` 时，即使 `text` 存在也不显示行数
   - 会回退到显示文件名（如果有）

2. **文件路径处理**：
   - 使用 `basename` 提取文件名，可能丢失目录上下文
   - 同名文件在不同目录无法区分

3. **空字符串文本**：
   - 如果 `text === ''` 但 `lineCount > 0`，仍会显示行数
   - 这是预期行为（空行选择）

### 改进建议

1. **添加文件路径提示**：
   ```typescript
   // 建议：显示相对路径或添加悬停提示
   const displayName = basename(ideSelection.filePath);
   const fullPath = ideSelection.filePath;
   // 使用 title 属性或自定义 tooltip 显示完整路径
   ```

2. **支持多文件指示**：
   ```typescript
   // 建议：如果未来支持多文件选择
   if (ideSelection.multipleFiles) {
     return <Text>⧉ {ideSelection.fileCount} files selected</Text>;
   }
   ```

3. **添加点击交互**：
   - 点击指示器可以跳转到 IDE 中的对应位置
   - 需要 IDE 扩展支持相应的 MCP 命令

4. **状态可视化增强**：
   ```typescript
   // 建议：区分连接状态和选择状态
   if (ideStatus === 'pending') {
     return <Text color="warning">⧉ Connecting...</Text>;
   }
   if (ideStatus === 'disconnected') {
     return <Text color="error">⧉ IDE disconnected</Text>;
   }
   ```

5. **性能优化**：
   - 当前每次渲染都调用 `basename`，可以缓存结果
   - 建议：在 `useMemo` 中计算文件名

### 主题与样式

- **颜色**：使用 `"ide"` 主题色，在主题系统中定义
- **图标**：使用 Unicode 字符 `⧉`（文件/选择图标）
- **截断**：长文本使用 `truncate` 包装模式

### 相关主题配置

```typescript
// 主题类型定义中的 ide 颜色
type Theme = {
  ide: string;  // IDE 相关元素的颜色
  // ... 其他颜色
};
```
