# DiagnosticsDisplay.tsx 研究文档

## 场景与职责

`DiagnosticsDisplay.tsx` 是 Claude Code CLI 中用于**显示代码诊断信息**的组件。它接收来自 IDE（如 VS Code）的诊断数据（错误、警告等），并以用户友好的格式展示在终端界面中。

### 核心职责
1. **诊断信息展示**：显示代码文件中的错误、警告等诊断信息
2. **简洁/详细模式**：根据 `verbose` 参数切换展示粒度
3. **文件路径处理**：解析和简化文件 URI 显示
4. **交互提示**：提示用户可以展开查看详细信息

## 功能点目的

### 1. 简洁模式（非 verbose）
- **触发条件**：`verbose = false`
- **显示内容**：
  - 总问题数（如 "Found 3 new diagnostic issues in 2 files"）
  - 展开提示（`<CtrlOToExpand />`）
- **用途**：在紧凑的对话中提供诊断概览

### 2. 详细模式（verbose）
- **触发条件**：`verbose = true`
- **显示内容**：
  - 每个文件的相对路径
  - 协议标识（file:// 或 claude_fs_right）
  - 每个诊断的详细信息：
    - 严重程度符号（✖, ⚠, ℹ, ★）
    - 位置（行:列）
    - 消息内容
    - 错误代码（如有）
    - 来源（如有）
- **用途**：提供完整的诊断详情供用户查看

### 3. 文件路径处理
- **URI 协议解析**：
  - `file://`：普通文件系统文件
  - `_claude_fs_right:`：右侧（修改后）文件系统视图
  - `_claude_fs_left:`：左侧（原始）文件系统视图
- **路径简化**：使用 `relative(getCwd(), ...)` 显示相对路径
- **协议标识**：在路径后显示协议类型

### 4. 严重程度符号
通过 `DiagnosticTrackingService.getSeveritySymbol()` 获取：
- `Error`：✖ (figures.cross)
- `Warning`：⚠ (figures.warning)
- `Info`：ℹ (figures.info)
- `Hint`：★ (figures.star)

## 具体技术实现

### 组件接口

```typescript
import type { Attachment } from '../utils/attachments.js';

type DiagnosticsAttachment = Extract<Attachment, { type: 'diagnostics' }>;

type DiagnosticsDisplayProps = {
  attachment: DiagnosticsAttachment;
  verbose: boolean;
};

export function DiagnosticsDisplay({ attachment, verbose }: DiagnosticsDisplayProps): React.ReactNode
```

### 数据结构

```typescript
// 来自 attachments.ts
interface DiagnosticsAttachment {
  type: 'diagnostics';
  files: DiagnosticFile[];
  isNew: boolean;
}

interface DiagnosticFile {
  uri: string;           // 文件 URI
  diagnostics: Diagnostic[];
}

interface Diagnostic {
  message: string;
  severity: 'Error' | 'Warning' | 'Info' | 'Hint';
  range: {
    start: { line: number; character: number };
    end: { line: number; character: number };
  };
  source?: string;
  code?: string;
}
```

### 核心算法

#### 总问题数计算
```typescript
const totalIssues = attachment.files.reduce(
  (sum, file) => sum + file.diagnostics.length, 
  0
);
```

#### 文件路径处理
```typescript
function formatFilePath(uri: string): string {
  // 移除协议前缀
  const cleanPath = uri
    .replace('file://', '')
    .replace('_claude_fs_right:', '');
  
  // 转换为相对路径
  return relative(getCwd(), cleanPath);
}

function getProtocolLabel(uri: string): string {
  if (uri.startsWith('file://')) return '(file://)';
  if (uri.startsWith('_claude_fs_right:')) return '(claude_fs_right)';
  return `(${uri.split(':')[0]})`;
}
```

#### 诊断格式化
```typescript
function formatDiagnostic(diag: Diagnostic, index: number): ReactNode {
  const symbol = DiagnosticTrackingService.getSeveritySymbol(diag.severity);
  const line = diag.range.start.line + 1;      // 0-based to 1-based
  const char = diag.range.start.character + 1;
  
  return (
    <Text>
      {symbol} [Line {line}:{char}] {diag.message}
      {diag.code && ` [${diag.code}]`}
      {diag.source && ` (${diag.source})`}
    </Text>
  );
}
```

### React Compiler 优化

- `$[0-1]`：缓存 `totalIssues` 计算结果（依赖 `attachment.files`）
- `$[2-5]`：缓存详细模式下的文件列表渲染
- `$[6-7]`：缓存简洁模式下的问题数显示
- `$[8]`：缓存 `<CtrlOToExpand />` 组件
- `$[9-13]`：缓存简洁模式的完整消息

## 关键代码路径与文件引用

### 当前文件
- `/home/sansha/Github/claude-code-instructkr/src/components/DiagnosticsDisplay.tsx`

### 直接依赖
| 导入路径 | 用途 |
|---------|------|
| `path` | `relative()` 计算相对路径 |
| `react` | React 核心 API |
| `../ink.js` | Ink UI 组件（Box, Text） |
| `../services/diagnosticTracking.js` | `DiagnosticTrackingService` |
| `../utils/attachments.js` | `Attachment` 类型 |
| `../utils/cwd.js` | `getCwd()` 获取当前目录 |
| `./CtrlOToExpand.js` | 展开提示组件 |
| `./MessageResponse.js` | 消息响应容器 |

### 相关依赖文件

#### diagnosticTracking.ts (`/home/sansha/Github/claude-code-instructkr/src/services/diagnosticTracking.ts`)

**DiagnosticTrackingService 核心功能**：
```typescript
export class DiagnosticTrackingService {
  // 获取严重程度符号
  static getSeveritySymbol(severity: Diagnostic['severity']): string {
    return {
      Error: figures.cross,    // ✖
      Warning: figures.warning, // ⚠
      Info: figures.info,       // ℹ
      Hint: figures.star,       // ★
    }[severity] || figures.bullet;
  }
  
  // 格式化诊断摘要
  static formatDiagnosticsSummary(files: DiagnosticFile[]): string;
  
  // 单例实例
  static getInstance(): DiagnosticTrackingService;
  
  // 其他方法：initialize, shutdown, reset, beforeFileEdited, getNewDiagnostics 等
}

export const diagnosticTracker = DiagnosticTrackingService.getInstance();
```

**诊断获取流程**：
1. `beforeFileEdited()`：在编辑前捕获基线诊断
2. `getNewDiagnostics()`：获取相对于基线的新诊断
3. 对比 `file://` 和 `_claude_fs_right:` 协议的文件

#### attachments.ts (`/home/sansha/Github/claude-code-instructkr/src/utils/attachments.ts`)

**DiagnosticsAttachment 定义**：
```typescript
export type Attachment = 
  | { type: 'diagnostics'; files: DiagnosticFile[]; isNew: boolean }
  | // ... 其他附件类型
```

**生成逻辑**：
```typescript
// 在主线程附件处理中
maybe('diagnostics', async () => getDiagnosticAttachments(toolUseContext));
```

#### CtrlOToExpand.tsx (`/home/sansha/Github/claude-code-instructkr/src/components/CtrlOToExpand.tsx`)

```typescript
export function CtrlOToExpand(): React.ReactNode;
export function ctrlOToExpand(): string;
```

在简洁模式下显示 `(ctrl+o to expand)` 提示。

#### MessageResponse.tsx (`/home/sansha/Github/claude-code-instructkr/src/components/MessageResponse.tsx`)

```typescript
export function MessageResponse({ children, height }: Props): React.ReactNode;
```

提供统一的响应消息容器，包含缩进和样式。

## 依赖与外部交互

### 与 IDE 的交互

1. **MCP 协议**：通过 MCP（Model Context Protocol）与 IDE 通信
2. **诊断获取**：调用 IDE 的 `getDiagnostics` 方法
3. **文件打开**：通过 `openFile` 确保文件已加载到 IDE

### 与文件系统的交互

1. **路径解析**：处理 `file://` 和 `_claude_fs_*` 协议
2. **相对路径**：使用 `getCwd()` 计算相对路径用于显示

### 与诊断追踪服务的交互

```typescript
// 使用静态方法获取符号
DiagnosticTrackingService.getSeveritySymbol(diagnostic.severity)
```

## 风险、边界与改进建议

### 已知风险

1. **URI 解析复杂性**
   - 风险：不同协议的 URI 格式可能变化
   - 现状：硬编码处理 `file://` 和 `_claude_fs_*`
   - 建议：使用 URL 解析库处理更健壮

2. **路径计算错误**
   - 风险：`relative()` 在跨文件系统时可能出错
   - 现状：直接调用，无错误处理
   - 建议：添加 try-catch 和 fallback

3. **大量诊断性能**
   - 风险：如果文件有很多诊断，渲染可能变慢
   - 现状：无分页或虚拟化
   - 建议：考虑分页或限制显示数量

### 边界情况

1. **空诊断列表**
   - 处理：`if (attachment.files.length === 0) return null;`
   - 行为：不渲染任何内容

2. **零个问题**
   - 场景：`files` 非空但所有 `diagnostics` 为空数组
   - 结果：`totalIssues = 0`，显示 "Found 0 new diagnostic issues"
   - 建议：考虑隐藏零问题的文件

3. **行号转换**
   - IDE 使用 0-based 行号
   - 显示使用 1-based（`line + 1`）
   - 注意：确保所有地方一致转换

4. **特殊字符消息**
   - 诊断消息可能包含特殊字符
   - Ink 的 Text 组件会自动处理转义

5. **文件不存在**
   - 诊断文件可能已被删除
   - `relative()` 仍可计算路径，但可能不准确

### 改进建议

1. **分组显示**
   - 按严重程度分组（Errors 优先）
   - 帮助用户优先处理重要问题

2. **文件折叠**
   - 允许展开/折叠单个文件的诊断
   - 减少视觉噪音

3. **快速修复提示**
   - 如果 IDE 提供快速修复，显示提示
   - 例如："Press Enter to apply quick fix"

4. **诊断过滤**
   - 允许按严重程度过滤
   - 例如：只显示 Errors，隐藏 Warnings

5. **代码片段显示**
   - 显示诊断相关的代码片段
   - 帮助用户快速定位问题

6. **导航支持**
   - 点击诊断在 IDE 中打开对应位置
   - 需要与 IDE 集成

7. **国际化**
   - 当前 "issue/issues" 是硬编码英文
   - 支持复数形式（1 issue / 2 issues）
   - 建议：完整的多语言支持

8. **诊断统计**
   - 显示按严重程度的统计
   - 例如："3 errors, 2 warnings, 1 info"

### 测试建议

1. **单元测试**
   - 测试 `totalIssues` 计算
   - 测试文件路径格式化
   - 测试诊断格式化
   - 测试单复数处理

2. **集成测试**
   - 测试与 DiagnosticTrackingService 的集成
   - 测试不同 URI 协议的处理

3. **视觉测试**
   - 测试长诊断消息的截断
   - 测试多文件情况下的布局

4. **边界测试**
   - 测试空文件列表
   - 测试零个问题
   - 测试超长文件路径
