# ToolSelector.tsx 深度研究文档

## 1. 场景与职责

### 1.1 核心场景
`ToolSelector.tsx` 是 Claude Code 中用于**工具选择**的交互式 UI 组件，主要应用于以下场景：

1. **Agent 创建向导** (`ToolsStep.tsx`): 在创建新 Agent 时选择该 Agent 可用的工具
2. **Agent 编辑器** (`AgentEditor.tsx`): 编辑现有 Agent 时修改其工具权限

### 1.2 核心职责
- 提供**分类展示**的工具选择界面（只读工具、编辑工具、执行工具、MCP工具、其他工具）
- 支持**批量选择/取消选择**（按分类或全选）
- 支持**MCP服务器级别的工具管理**
- 提供**键盘导航**交互（上下箭头、回车确认、ESC取消）
- 支持"高级选项"展开显示单个工具

---

## 2. 功能点目的

### 2.1 工具分类系统
组件将工具分为5个逻辑桶（Bucket）：

| 分类 | 包含工具 | 目的 |
|------|---------|------|
| `READ_ONLY` | GlobTool, GrepTool, FileReadTool, WebFetchTool, WebSearchTool, TodoWriteTool, TaskStopTool, TaskOutputTool, ListMcpResourcesTool, ReadMcpResourceTool, ExitPlanModeV2Tool | 只读操作，安全无风险 |
| `EDIT` | FileEditTool, FileWriteTool, NotebookEditTool | 文件修改操作 |
| `EXECUTION` | BashTool, TungstenTool (ant-only) | 代码执行，高风险 |
| `MCP` | 动态 MCP 工具 | 外部 MCP 服务器提供的工具 |
| `OTHER` | 未分类工具 | 捕获所有其他工具 |

### 2.2 选择模式
- **`undefined` 或 `['*']`**: 表示"所有工具"（通配符语义）
- **具体工具名数组**: 限制为指定工具
- **空数组**: 无工具权限

### 2.3 MCP 工具特殊处理
- MCP 工具以 `mcp__` 前缀标识
- 支持按 MCP 服务器分组显示
- 使用 `mcpInfoFromString()` 解析服务器名和工具名

---

## 3. 具体技术实现

### 3.1 核心数据结构

```typescript
// 工具桶定义
type ToolBucket = {
  name: string;
  toolNames: Set<string>;
  isMcp?: boolean;
};

// 导航项（UI渲染用）
type NavigableItem = {
  id: string;
  label: string;
  action: () => void;
  isHeader?: boolean;
  isToggle?: boolean;
  isContinue?: boolean;
};
```

### 3.2 关键流程

#### 工具分类流程
```
1. 调用 filterToolsForAgent() 过滤可用工具
2. 遍历工具，使用 isMcpTool() 检测 MCP 工具
3. 非 MCP 工具按 getToolBuckets() 定义的静态列表分类
4. 生成 toolsByBucket 对象（readOnly/edit/execution/mcp/other）
```

#### 选择状态管理
```
1. 接收 initialTools（可能包含 '*' 通配符）
2. 展开通配符为具体工具列表用于显示
3. 用户选择后，validSelectedTools 过滤掉已不存在工具
4. 提交时：如果全选则返回 undefined，否则返回选中工具名数组
```

### 3.3 关键算法

#### 全选检测
```typescript
const isAllSelected = validSelectedTools.length === customAgentTools.length && customAgentTools.length > 0;
```

#### MCP 服务器分组
```typescript
function getMcpServerBuckets(tools: Tools) {
  const serverMap = new Map<string, Tool[]>();
  tools.forEach(tool => {
    if (isMcpTool(tool)) {
      const mcpInfo = mcpInfoFromString(tool.name);
      if (mcpInfo?.serverName) {
        const existing = serverMap.get(mcpInfo.serverName) || [];
        existing.push(tool);
        serverMap.set(mcpInfo.serverName, existing);
      }
    }
  });
  // 返回按服务器名排序的数组
}
```

### 3.4 React Compiler 优化
代码使用 React Compiler（`_c` 函数）进行自动记忆化：
- 使用 `$[n]` 作为记忆化槽位
- `Symbol.for("react.memo_cache_sentinel")` 作为未初始化标记
- 依赖数组比较决定是否复用缓存

---

## 4. 关键代码路径与文件引用

### 4.1 入口点
| 文件 | 用途 |
|------|------|
| `src/components/agents/new-agent-creation/wizard-steps/ToolsStep.tsx` | 创建 Agent 向导中的工具选择步骤 |
| `src/components/agents/AgentEditor.tsx` | 编辑 Agent 时的工具修改 |

### 4.2 依赖文件
| 文件 | 用途 |
|------|------|
| `src/tools/AgentTool/agentToolUtils.ts` | `filterToolsForAgent()` - 过滤 Agent 可用工具 |
| `src/services/mcp/utils.ts` | `isMcpTool()` - 检测 MCP 工具 |
| `src/services/mcp/mcpStringUtils.ts` | `mcpInfoFromString()` - 解析 MCP 工具名 |
| `src/tools/AgentTool/constants.ts` | `AGENT_TOOL_NAME` - Agent 工具名常量 |
| `src/utils/array.ts` | `count()` - 数组计数工具 |
| `src/utils/stringUtils.ts` | `plural()` - 复数形式处理 |

### 4.3 工具类引用
组件直接引用多个工具类获取其 name：
- `BashTool`, `ExitPlanModeV2Tool`, `FileEditTool`, `FileReadTool`
- `FileWriteTool`, `GlobTool`, `GrepTool`, `ListMcpResourcesTool`
- `NotebookEditTool`, `ReadMcpResourceTool`, `TaskOutputTool`
- `TaskStopTool`, `TodoWriteTool`, `TungstenTool`, `WebFetchTool`, `WebSearchTool`

---

## 5. 依赖与外部交互

### 5.1 Props 接口
```typescript
type Props = {
  tools: Tools;                          // 所有可用工具
  initialTools: string[] | undefined;    // 初始选择（可能含 '*'）
  onComplete: (selectedTools: string[] | undefined) => void;  // 完成回调
  onCancel?: () => void;                 // 取消回调
};
```

### 5.2 外部状态交互
- 使用 `useKeybinding` 注册键盘快捷键
- 通过 `figures` 库显示复选框符号（☐/☑）
- 使用 Ink 组件（`Box`, `Text`）进行终端 UI 渲染

### 5.3 依赖关系图
```
ToolSelector.tsx
├── filterToolsForAgent (agentToolUtils.ts)
│   ├── ALL_AGENT_DISALLOWED_TOOLS (constants/tools.ts)
│   ├── CUSTOM_AGENT_DISALLOWED_TOOLS (constants/tools.ts)
│   └── ASYNC_AGENT_ALLOWED_TOOLS (constants/tools.ts)
├── isMcpTool (services/mcp/utils.ts)
├── mcpInfoFromString (services/mcp/mcpStringUtils.ts)
└── 各 Tool 类 (src/tools/*Tool/*.ts)
```

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 风险1: MCP 工具名解析歧义
```typescript
// mcpInfoFromString 的已知限制：
// "mcp__my__server__tool" 会错误解析为 server="my", tool="server__tool"
// 而非预期的 server="my__server", tool="tool"
```
**影响**: 如果 MCP 服务器名包含 `__`，分组显示会出错。

#### 风险2: 硬编码工具分类
工具分类是静态定义的：
```typescript
READ_ONLY: {
  toolNames: new Set([GlobTool.name, GrepTool.name, ...])
}
```
**影响**: 新增工具类型需要修改此文件，容易遗漏。

#### 风险3: React Compiler 生成的代码可读性差
编译后的代码使用大量 `t0`, `t1`, `T0`, `T1` 等临时变量，调试困难。

### 6.2 边界情况

| 场景 | 行为 |
|------|------|
| `initialTools` 包含已不存在的工具 | 通过 `validSelectedTools` 过滤，静默忽略 |
| 所有工具都被禁用 | 显示空列表，仍可继续 |
| MCP 服务器无工具 | 该服务器不显示在列表中 |
| 同时按上下箭头 | 通过 `Math.max`/`Math.min` 限制范围 |

### 6.3 改进建议

#### 建议1: 动态工具分类
将工具分类配置化，从工具元数据读取：
```typescript
// 建议：工具类添加 category 属性
class FileReadTool {
  static category = 'read-only';
}
```

#### 建议2: MCP 工具名编码改进
使用更可靠的编码方案（如 URL 编码）：
```typescript
// 当前: mcp__server__tool
// 建议: mcp://server/tool 或 mcp__server__base64(tool)
```

#### 建议3: 添加搜索/过滤功能
当工具数量很多时（>50），提供搜索框快速定位工具。

#### 建议4: 添加工具悬停提示
显示工具的描述信息，帮助用户理解每个工具的用途。

### 6.4 测试建议
- 测试 MCP 工具分组逻辑
- 测试通配符 `*` 的展开和收缩
- 测试键盘导航在边界情况下的行为
- 测试工具被删除后的兼容性
