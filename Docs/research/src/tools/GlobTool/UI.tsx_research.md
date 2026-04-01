# GlobTool/UI.tsx 研究文档

## 场景与职责

UI.tsx 是 GlobTool 的 UI 渲染层模块，负责定义 GlobTool 在用户界面中的各种展示形态。该模块采用 React + Ink 技术栈，为终端 UI 提供组件化渲染能力。

GlobTool 是一个基于 glob 模式的文件搜索工具，UI.tsx 专门处理以下展示场景：
- 工具使用时的输入参数展示（renderToolUseMessage）
- 工具执行错误的友好提示（renderToolUseErrorMessage）
- 工具执行结果的展示（复用 GrepTool 的渲染逻辑）
- 工具使用摘要的生成（getToolUseSummary）

## 功能点目的

### 1. userFacingName()
返回工具的用户可见名称 `"Search"`，在 UI 中统一显示为"Search"而非内部工具名"Glob"。

### 2. renderToolUseMessage()
渲染工具使用时的参数展示：
- 仅 pattern：显示 `pattern: "xxx"`
- pattern + path：显示 `pattern: "xxx", path: "yyy"`
- 非 verbose 模式下使用 `getDisplayPath()` 简化路径显示
- 无 pattern 时返回 null

### 3. renderToolUseErrorMessage()
渲染工具执行错误时的友好提示：
- 非 verbose 模式下检测 `FILE_NOT_FOUND_CWD_NOTE` 标记，显示简化的"File not found"
- 其他错误显示"Error searching files"
- verbose 模式或无法识别的错误回退到 FallbackToolUseErrorMessage

### 4. renderToolResultMessage
**直接复用 GrepTool.renderToolResultMessage**，保持搜索类工具结果展示的一致性。GlobTool 本身不实现自己的结果渲染，而是通过 `export const renderToolResultMessage = GrepTool.renderToolResultMessage` 直接导出 GrepTool 的实现。

### 5. getToolUseSummary()
生成工具使用的简短摘要：
- 基于 pattern 生成，使用 `truncate()` 截断至 `TOOL_SUMMARY_MAX_LENGTH`（50字符）
- 用于紧凑视图中的工具使用概览

## 具体技术实现

### 关键数据结构

```typescript
// 输入参数类型（Partial 因为参数可能未完全流式传输）
interface ToolInput {
  pattern?: string;
  path?: string;
}

// 渲染选项
interface RenderOptions {
  verbose: boolean;  // 是否详细模式
}
```

### 关键流程

1. **工具使用消息渲染流程**
   ```
   renderToolUseMessage(input, options)
   ├── 检查 pattern 是否存在
   ├── 检查 path 是否存在
   ├── verbose 模式：显示完整路径
   └── 非 verbose 模式：使用 getDisplayPath() 简化路径
   ```

2. **错误消息渲染流程**
   ```
   renderToolUseErrorMessage(result, options)
   ├── 非 verbose 模式？
   │   ├── 提取 tool_use_error 标签内容
   │   ├── 包含 FILE_NOT_FOUND_CWD_NOTE？
   │   │   └── 显示 "File not found"
   │   └── 其他错误
   │       └── 显示 "Error searching files"
   └── 否则回退到 FallbackToolUseErrorMessage
   ```

3. **结果消息渲染流程**
   ```
   直接委托给 GrepTool.renderToolResultMessage
   （代码注释：GlobTool reuses GrepTool's renderToolResultMessage）
   ```

### 依赖导入详解

| 导入路径 | 用途 |
|---------|------|
| `@anthropic-ai/sdk/resources/index.mjs` | ToolResultBlockParam 类型定义 |
| `react` | React 核心库 |
| `src/components/MessageResponse.js` | 消息响应容器组件 |
| `src/utils/messages.js` | extractTag 工具函数（提取 XML 标签内容） |
| `../../components/FallbackToolUseErrorMessage.js` | 错误回退组件 |
| `../../constants/toolLimits.js` | TOOL_SUMMARY_MAX_LENGTH 常量 |
| `../../ink.js` | Text 组件（Ink UI 库） |
| `../../utils/file.js` | FILE_NOT_FOUND_CWD_NOTE, getDisplayPath |
| `../../utils/format.js` | truncate 截断函数 |
| `../GrepTool/GrepTool.js` | 复用 renderToolResultMessage |

## 关键代码路径与文件引用

### 当前文件关键代码

```typescript
// 行 11-13: 用户可见名称
export function userFacingName(): string {
  return 'Search';
}

// 行 14-32: 工具使用消息渲染
export function renderToolUseMessage({ pattern, path }, { verbose }) {
  if (!pattern) return null;
  if (!path) return `pattern: "${pattern}"`;
  return `pattern: "${pattern}", path: "${verbose ? path : getDisplayPath(path)}"`;
}

// 行 33-50: 错误消息渲染
export function renderToolUseErrorMessage(result, { verbose }) {
  if (!verbose && typeof result === 'string' && extractTag(result, 'tool_use_error')) {
    const errorMessage = extractTag(result, 'tool_use_error');
    if (errorMessage?.includes(FILE_NOT_FOUND_CWD_NOTE)) {
      return <MessageResponse><Text color="error">File not found</Text></MessageResponse>;
    }
    return <MessageResponse><Text color="error">Error searching files</Text></MessageResponse>;
  }
  return <FallbackToolUseErrorMessage result={result} verbose={verbose} />;
}

// 行 53: 复用 GrepTool 的结果渲染
export const renderToolResultMessage = GrepTool.renderToolResultMessage;

// 行 54-62: 工具使用摘要
export function getToolUseSummary(input) {
  if (!input?.pattern) return null;
  return truncate(input.pattern, TOOL_SUMMARY_MAX_LENGTH);
}
```

### 相关文件引用

| 文件路径 | 引用关系 | 说明 |
|---------|---------|------|
| `src/tools/GlobTool/GlobTool.ts` | 导入 UI.tsx | 工具定义中使用 UI.tsx 导出的函数 |
| `src/tools/GrepTool/GrepTool.ts` | 导入 renderToolResultMessage | UI.tsx 复用 GrepTool 的结果渲染 |
| `src/tools/GrepTool/UI.tsx` | 被 GlobTool/UI.tsx 复用 | GrepTool 的完整 UI 实现 |
| `src/components/MessageResponse.tsx` | 被导入 | 消息响应容器组件 |
| `src/components/FallbackToolUseErrorMessage.tsx` | 被导入 | 错误回退组件 |
| `src/utils/messages.ts` | 被导入 | extractTag 函数 |
| `src/utils/file.ts` | 被导入 | getDisplayPath, FILE_NOT_FOUND_CWD_NOTE |
| `src/utils/format.ts` | 被导入 | truncate 函数 |
| `src/utils/truncate.ts` | 被 format.ts 导入 | 实际截断实现 |
| `src/constants/toolLimits.ts` | 被导入 | TOOL_SUMMARY_MAX_LENGTH 常量 |
| `src/ink.ts` | 被导入 | Ink UI 组件库 |

## 依赖与外部交互

### 运行时依赖

1. **React / Ink**: 用于终端 UI 渲染
2. **@anthropic-ai/sdk**: 类型定义（ToolResultBlockParam）
3. **GrepTool**: 结果渲染逻辑复用

### 与 GlobTool.ts 的协作

GlobTool.ts 通过以下方式使用 UI.tsx：

```typescript
// GlobTool.ts 行 18-24
import {
  getToolUseSummary,
  renderToolResultMessage,
  renderToolUseErrorMessage,
  renderToolUseMessage,
  userFacingName,
} from './UI.js'

// 在 buildTool 配置中注册
export const GlobTool = buildTool({
  name: GLOB_TOOL_NAME,
  userFacingName,
  getToolUseSummary,
  renderToolUseMessage,
  renderToolUseErrorMessage,
  renderToolResultMessage,
  // ...
})
```

### 与 GrepTool 的关系

GlobTool 和 GrepTool 都是搜索类工具，共享结果渲染逻辑：
- GlobTool：按文件名模式搜索（glob 模式）
- GrepTool：按内容正则搜索

两者在 UI 展示上保持一致，因此 GlobTool 直接复用 GrepTool 的 `renderToolResultMessage`。

## 风险、边界与改进建议

### 潜在风险

1. **与 GrepTool 的紧耦合**
   - GlobTool 直接复用 GrepTool.renderToolResultMessage
   - 如果 GrepTool 的渲染逻辑变更，可能影响 GlobTool 的展示
   - 建议：建立搜索类工具的共享 UI 基类或抽象

2. **错误处理局限性**
   - 仅识别 FILE_NOT_FOUND_CWD_NOTE 标记的错误
   - 其他类型的错误统一显示"Error searching files"
   - 可能掩盖具体的错误信息

3. **verbose 模式切换**
   - 非 verbose 和 verbose 模式下的展示差异较大
   - 用户可能在需要详细信息时未开启 verbose 模式

### 边界情况

1. **空 pattern**: renderToolUseMessage 返回 null
2. **超长 pattern**: getToolUseSummary 使用 truncate 截断至 50 字符
3. **UNC 路径**: 通过 getDisplayPath 处理，但错误处理中可能涉及
4. **无 path 参数**: 仅显示 pattern，不显示路径

### 改进建议

1. **解耦 GrepTool 依赖**
   ```typescript
   // 建议：创建共享的搜索工具 UI 模块
   import { renderSearchResultMessage } from '../shared/SearchToolUI.js'
   export const renderToolResultMessage = renderSearchResultMessage
   ```

2. **增强错误识别**
   - 添加更多错误类型的识别（权限错误、超时等）
   - 提供更具体的用户提示

3. **路径显示优化**
   - 考虑在摘要中也显示路径信息（当 path 与 cwd 不同时）
   - 对于深层嵌套目录，考虑使用 truncatePathMiddle

4. **类型安全**
   - 当前使用 Partial<ToolInput>，可以考虑更精确的类型定义
   - 添加对输入参数的验证

### 测试建议

- 测试各种 pattern 和 path 组合的渲染输出
- 测试错误消息的各种场景（文件不存在、权限错误等）
- 测试 verbose 和非 verbose 模式的差异
- 测试超长 pattern 的截断行为
