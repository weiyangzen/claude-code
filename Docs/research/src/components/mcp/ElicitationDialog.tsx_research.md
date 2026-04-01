# ElicitationDialog.tsx 深度研究文档

## 1. 场景与职责

### 1.1 组件定位

`ElicitationDialog.tsx` 是 Claude Code 中 MCP (Model Context Protocol) 服务器与用户交互的核心 UI 组件。它负责渲染 MCP 服务器向用户发起的**请求输入对话框**（elicitation），支持两种模式：

1. **Form 模式** (`ElicitationFormDialog`): 显示结构化表单，收集用户输入
2. **URL 模式** (`ElicitationURLDialog`): 显示 URL 授权对话框，用于 OAuth 等流程

### 1.2 业务场景

| 场景 | 说明 |
|------|------|
| MCP 工具需要用户输入 | 服务器通过 `ElicitRequest` 请求用户填写表单 |
| OAuth 授权流程 | 服务器请求打开外部浏览器完成授权 |
| 错误恢复重试 | 特定错误码 (-32042) 触发 URL 重试流程 |
| 异步验证等待 | 日期/时间字段的自然语言解析需要异步验证 |

### 1.3 核心职责

- **输入收集**: 支持文本、数字、布尔值、枚举、多选枚举、日期/日期时间等多种字段类型
- **实时验证**: 同步 Zod 验证 + 异步自然语言日期解析
- **键盘导航**: 完整的键盘交互支持（上下箭头、Enter、Space、Esc 等）
- **状态管理**: 管理表单值、验证错误、焦点状态、手风琴展开状态
- **生命周期管理**: 处理 AbortSignal 取消、异步操作清理、组件卸载清理

---

## 2. 功能点目的

### 2.1 双模式架构

```typescript
export function ElicitationDialog(t0) {
  if (event.params.mode === "url") {
    return <ElicitationURLDialog ... />;
  }
  return <ElicitationFormDialog ... />;
}
```

**设计目的**: 
- URL 模式和 Form 模式有完全不同的 UI 流程和状态机
- URL 模式需要处理两阶段流程（prompt → waiting）
- 分离实现避免条件判断污染，提高可维护性

### 2.2 Form 模式字段类型支持

| 字段类型 | 说明 | 特殊交互 |
|---------|------|---------|
| `string` | 文本输入 | 支持 min/maxLength 验证 |
| `number`/`integer` | 数值输入 | 支持 min/max 范围验证 |
| `boolean` | 布尔值 | Space 切换，y/n 快速输入 |
| `enum` (单选) | 单选枚举 | 手风琴展开，支持类型ahead |
| `array` + `items.enum` | 多选枚举 | 复选框，min/maxItems 验证 |
| `date`/`date-time` | 日期/时间 | 自然语言解析（如 "tomorrow at 3pm"） |

### 2.3 URL 模式两阶段流程

```
Phase 1 (prompt): 显示 URL → 用户选择 Accept/Decline
                    ↓ Accept
              打开浏览器
                    ↓
Phase 2 (waiting): 显示等待状态 → 等待服务器完成通知
                    ↓ 完成通知
              自动关闭或显示重试按钮
```

**设计目的**: OAuth 等流程需要用户在外部浏览器完成操作后，服务器确认完成才能继续。

### 2.4 异步日期解析

```typescript
// 用户输入 "tomorrow at 3pm"
const result = await parseNaturalLanguageDateTime(input, 'date-time', signal);
// 返回 "2025-04-02T15:00:00-07:00"
```

- 使用 Haiku 模型进行自然语言理解
- 2 秒防抖 debounce，避免频繁调用
- 失败时保留原始输入并显示错误

---

## 3. 具体技术实现

### 3.1 关键数据结构

#### 3.1.1 Props 定义

```typescript
type Props = {
  event: ElicitationRequestEvent;        // 请求事件对象
  onResponse: (action: ElicitResult['action'], content?: ElicitResult['content']) => void;
  onWaitingDismiss?: (action: 'dismiss' | 'retry' | 'cancel') => void;  // URL 模式专用
};
```

#### 3.1.2 ElicitationRequestEvent

```typescript
type ElicitationRequestEvent = {
  serverName: string;           // MCP 服务器名称
  requestId: string | number;   // JSON-RPC 请求 ID
  params: ElicitRequestParams;  // 请求参数（form 或 url）
  signal: AbortSignal;          // 取消信号
  respond: (response: ElicitResult) => void;  // 响应回调
  waitingState?: ElicitationWaitingState;     // URL 模式等待状态配置
  onWaitingDismiss?: (action: 'dismiss' | 'retry' | 'cancel') => void;
  completed?: boolean;          // URL 模式完成标记
};
```

#### 3.1.3 Form 状态管理

```typescript
// 表单值状态
const [formValues, setFormValues] = useState<Record<string, string | number | boolean | string[]>>(...);

// 验证错误状态
const [validationErrors, setValidationErrors] = useState<Record<string, string>>(...);

// 字段导航状态
const [currentFieldIndex, setCurrentFieldIndex] = useState<number | undefined>(...);
const [focusedButton, setFocusedButton] = useState<'accept' | 'decline' | null>(...);

// 手风琴状态（用于枚举字段）
const [expandedAccordion, setExpandedAccordion] = useState<string | undefined>();
const [accordionOptionIndex, setAccordionOptionIndex] = useState(0);

// 异步解析状态
const [resolvingFields, setResolvingFields] = useState<Set<string>>(() => new Set());
```

### 3.2 关键流程

#### 3.2.1 表单提交流程

```
1. 用户点击 Accept 或按 Enter
2. 调用 validateRequired() 检查必填字段
3. 检查 validationErrors 是否有错误
4. 如有错误，跳转到第一个错误字段
5. 如无错误，调用 onResponse('accept', formValues)
6. elicitationHandler 接收响应，resolve Promise
7. MCP 客户端将结果发送回服务器
```

#### 3.2.2 日期字段异步解析流程

```
1. 用户输入文本（如 "next Monday"）
2. 同步验证失败（不符合 ISO 8601）
3. 启动 2 秒 debounce 定时器
4. 用户停止输入 2 秒后：
   a. 调用 parseNaturalLanguageDateTime()
   b. 显示 ResolvingSpinner
   c. 使用 Haiku 解析自然语言
   d. 验证解析结果
   e. 更新表单值和输入框显示
5. 用户离开字段时立即触发解析（如果 pending）
```

#### 3.2.3 URL 模式状态机

```
[prompt phase]                    [waiting phase]
     │                                   │
     │ 用户按 Accept                     │ 收到 completed 通知
     ▼                                   ▼
┌─────────┐    打开浏览器      ┌─────────────┐    自动关闭
│ Accept  │ ───────────────→ │ 等待服务器   │ ───────────→ (结束)
│ Decline │                  │ 确认完成     │
└─────────┘                  └─────────────┘
     │                            │
     │ 用户按 Decline             │ 用户按 Continue/Retry
     ▼                            ▼
  结束(拒绝)                   结束(继续)
```

### 3.3 键盘交互协议

#### 3.3.1 全局导航

| 按键 | 功能 |
|------|------|
| ↑/↓ | 在字段和按钮之间导航 |
| ←/→ | 在 Accept/Decline 按钮间切换 |
| Enter | 确认当前选择 |
| Esc | 取消对话框 |

#### 3.3.2 文本字段

| 按键 | 功能 |
|------|------|
| 任意字符 | 输入文本 |
| Backspace (空值时) | 清除字段值 |
| Enter | 提交并跳到下一字段 |

#### 3.3.3 布尔字段

| 按键 | 功能 |
|------|------|
| Space | 切换值 |
| y/n | 快速设置为 true/false |
| Backspace | 清除值 |

#### 3.3.4 枚举字段（手风琴）

| 按键 | 功能（折叠状态） | 功能（展开状态） |
|------|-----------------|-----------------|
| → | 展开手风琴 | - |
| ↑/↓ | 跳到上一/下一字段 | 在选项间导航 |
| Space | - | 选择/取消选择 |
| Enter | 跳到下一字段 | 选择并跳到下一字段 |
| ←/Esc | - | 折叠手风琴 |

#### 3.3.5 多选枚举字段

| 按键 | 功能（展开状态） |
|------|-----------------|
| Space | 切换选中状态 |
| Enter | 选中当前项并跳到下一字段 |

### 3.4 验证系统

#### 3.4.1 同步验证 (Zod)

```typescript
// src/utils/mcp/elicitationValidation.ts
export function validateElicitationInput(
  stringValue: string,
  schema: PrimitiveSchemaDefinition,
): ValidationResult {
  const zodSchema = getZodSchema(schema);
  const parseResult = zodSchema.safeParse(stringValue);
  // 返回 { value, isValid: true } 或 { isValid: false, error }
}
```

支持的验证规则：
- String: minLength, maxLength, format (email, uri, date, date-time)
- Number/Integer: minimum, maximum
- Enum: 值必须在枚举列表中
- Array: minItems, maxItems（多选枚举）

#### 3.4.2 异步验证（自然语言日期）

```typescript
export async function validateElicitationInputAsync(
  stringValue: string,
  schema: PrimitiveSchemaDefinition,
  signal: AbortSignal,
): Promise<ValidationResult> {
  const syncResult = validateElicitationInput(stringValue, schema);
  if (syncResult.isValid) return syncResult;
  
  // 尝试自然语言解析
  if (isDateTimeSchema(schema) && !looksLikeISO8601(stringValue)) {
    const parseResult = await parseNaturalLanguageDateTime(...);
    if (parseResult.success) {
      return validateElicitationInput(parseResult.value, schema);
    }
  }
  return syncResult;
}
```

### 3.5 滚动窗口优化

对于字段较多的表单，实现虚拟滚动：

```typescript
const LINES_PER_FIELD = 3;
const DIALOG_OVERHEAD = 14;
const maxVisibleFields = Math.max(2, Math.floor((rows - DIALOG_OVERHEAD) / LINES_PER_FIELD));

const scrollWindow = useMemo(() => {
  // 根据当前焦点字段计算可见范围
  // 保持焦点字段在视口中间
}, [schemaFields.length, maxVisibleFields, currentFieldIndex]);
```

---

## 4. 关键代码路径与文件引用

### 4.1 调用链

```
MCP Server
    ↓ (JSON-RPC ElicitRequest)
@modelcontextprotocol/sdk
    ↓
Client.setRequestHandler(ElicitRequestSchema, ...)
    ↓ (src/services/mcp/elicitationHandler.ts:77)
registerElicitationHandler()
    ↓
setAppState() 将请求加入 queue
    ↓ (src/state/AppStateStore.ts:226-228)
AppState.elicitation.queue: ElicitationRequestEvent[]
    ↓ (src/screens/REPL.tsx:636)
const elicitation = useAppState(s => s.elicitation);
    ↓
ElicitationDialog (取 queue[0] 渲染)
    ↓
用户交互 → onResponse()
    ↓
event.respond(result) → resolve Promise
    ↓
结果返回 MCP Server
```

### 4.2 核心文件引用

| 文件 | 用途 |
|------|------|
| `src/components/mcp/ElicitationDialog.tsx` | 主组件实现 |
| `src/services/mcp/elicitationHandler.ts` | MCP 请求处理、队列管理 |
| `src/utils/mcp/elicitationValidation.ts` | 验证逻辑、Zod schema 构建 |
| `src/utils/mcp/dateTimeParser.ts` | 自然语言日期解析（Haiku） |
| `src/state/AppStateStore.ts` | AppState 类型定义（elicitation.queue） |
| `src/screens/REPL.tsx` | 读取 elicitation 状态并渲染对话框 |
| `src/context/overlayContext.tsx` | 覆盖层管理（Escape 键协调） |
| `src/components/design-system/Dialog.tsx` | 基础对话框组件 |
| `src/utils/browser.ts` | 打开浏览器（openBrowser） |

### 4.3 关键代码段

#### 4.3.1 表单渲染入口

```typescript
// ElicitationDialog.tsx:778-956
function renderFormFields(): React.ReactNode {
  return <Box flexDirection="column">
    {hasFieldsAbove && /* 上方更多指示器 */}
    {schemaFields.slice(scrollWindow.start, scrollWindow.end).map((field, visibleIdx) => {
      // 根据字段类型渲染不同 UI
      if (isMultiSelectEnumSchema(schema)) {
        // 多选枚举渲染
      } else if (isEnumSchema(schema)) {
        // 单选枚举渲染
      } else if (schema.type === 'boolean') {
        // 布尔值渲染
      } else if (isTextField(schema)) {
        // 文本输入渲染
      }
    })}
    {hasFieldsBelow && /* 下方更多指示器 */}
  </Box>;
}
```

#### 4.3.2 键盘输入处理

```typescript
// ElicitationDialog.tsx:483-730
useInput((_input, key) => {
  // 1. 展开的多选手风琴处理
  if (expandedAccordion && isMultiSelectEnumSchema(...)) { ... }
  
  // 2. 展开的单选枚举手风琴处理
  if (expandedAccordion && isEnumSchema(...)) { ... }
  
  // 3. Accept/Decline 按钮处理
  if (key.return && focusedButton === 'accept') { ... }
  
  // 4. 上下导航
  if (key.upArrow || key.downArrow) { handleNavigation(...); }
  
  // 5. 当前字段类型特定处理
  if (schema.type === 'boolean') { ... }
  if (isEnumSchema(schema) || isMultiSelectEnumSchema(schema)) { ... }
});
```

#### 4.3.3 清理副作用

```typescript
// ElicitationDialog.tsx:234-246
useEffect(() => () => {
  // 清除 pending debounce 定时器
  if (dateDebounceRef.current !== undefined) {
    clearTimeout(dateDebounceRef.current);
  }
  // 清除 typeahead 定时器
  if (ta.timer !== undefined) {
    clearTimeout(ta.timer);
  }
  // 中止所有进行中的异步验证
  for (const controller of resolveAbortRef.current.values()) {
    controller.abort();
  }
  resolveAbortRef.current.clear();
}, []);
```

---

## 5. 依赖与外部交互

### 5.1 MCP SDK 类型依赖

```typescript
import type { 
  ElicitRequestFormParams, 
  ElicitRequestURLParams, 
  ElicitResult, 
  PrimitiveSchemaDefinition 
} from '@modelcontextprotocol/sdk/types.js';
```

### 5.2 React Hook 依赖

| Hook | 用途 |
|------|------|
| `useRegisterOverlay` | 注册为活动覆盖层，协调 Escape 键处理 |
| `useNotifyAfterTimeout` | 空闲时发送桌面通知 |
| `useTerminalSize` | 获取终端尺寸用于滚动窗口计算 |
| `useKeybinding` | 绑定键盘快捷键 |
| `useInput` (ink) | 原始键盘输入处理 |

### 5.3 工具函数依赖

```typescript
// 验证工具
import { 
  getEnumLabel, getEnumValues, 
  getMultiSelectLabel, getMultiSelectValues,
  isDateTimeSchema, isEnumSchema, isMultiSelectEnumSchema,
  validateElicitationInput, validateElicitationInputAsync 
} from '../../utils/mcp/elicitationValidation.js';

// 浏览器工具
import { openBrowser } from '../../utils/browser.js';

// 字符串工具
import { plural } from '../../utils/stringUtils.js';
```

### 5.4 外部服务交互

```
┌─────────────────────────────────────────────────────────────┐
│                     ElicitationDialog                        │
└─────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        ▼                     ▼                     ▼
┌───────────────┐    ┌─────────────────┐    ┌──────────────┐
│  AppState     │    │  Haiku API      │    │  Browser     │
│  (queue)      │    │  (日期解析)      │    │  (URL 模式)   │
└───────────────┘    └─────────────────┘    └──────────────┘
        │                     │                     │
        ▼                     ▼                     ▼
┌───────────────┐    ┌─────────────────┐    ┌──────────────┐
│  MCP Server   │    │  Claude API     │    │  External    │
│  (via SDK)    │    │  (queryHaiku)   │    │  OAuth       │
└───────────────┘    └─────────────────┘    └──────────────┘
```

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 异步操作清理

**风险**: 组件卸载时进行中的异步验证可能继续执行并尝试更新已卸载组件的状态。

**当前缓解**: 
```typescript
// 使用 AbortController 中止进行中的请求
for (const controller of resolveAbortRef.current.values()) {
  controller.abort();
}
```

**潜在问题**: `validateElicitationInputAsync` 中的 `parseNaturalLanguageDateTime` 可能不响应 AbortSignal。

#### 6.1.2 日期解析失败处理

**风险**: 自然语言日期解析依赖 Haiku API，失败时用户体验不佳。

**当前行为**: 保留原始输入，显示验证错误。

**建议**: 提供更友好的错误提示，引导用户使用标准格式。

#### 6.1.3 大量字段性能

**风险**: 字段数量过多时，滚动窗口计算和渲染可能产生性能问题。

**当前缓解**: 已实现虚拟滚动（scrollWindow），只渲染可见字段。

**边界**: `LINES_PER_FIELD = 3` 是估算值，实际字段可能占用更多行（如展开的多选手风琴）。

### 6.2 边界条件

| 边界条件 | 行为 |
|---------|------|
| 无字段表单 | 直接显示 Accept/Decline 按钮，Accept 默认聚焦 |
| 所有字段可选 | 允许直接提交空表单 |
| 验证失败时提交 | 跳转到第一个错误字段，显示错误信息 |
| AbortSignal 触发 | 调用 onResponse('cancel') 关闭对话框 |
| URL 解析失败 | 显示完整 URL，不高亮域名 |
| 终端尺寸过小 | 最少显示 2 个字段，可能溢出 |

### 6.3 改进建议

#### 6.3.1 性能优化

1. **字段高度动态计算**: 当前使用固定 `LINES_PER_FIELD = 3`，应根据实际内容计算。
   ```typescript
   // 建议：展开的多选手风琴应计算为 N+3 行
   const getFieldHeight = (field, isExpanded) => 
     isMultiSelectEnumSchema(field.schema) && isExpanded 
       ? getMultiSelectValues(field.schema).length + 3 
       : 3;
   ```

2. **React Compiler 优化**: 文件已使用 React Compiler（`_c` 调用），但部分 memoization 可进一步优化。

#### 6.3.2 用户体验改进

1. **日期输入提示**: 当前 placeholder 为通用 "Type something…"，应根据 schema format 显示具体提示。
   ```typescript
   // 建议
   placeholder={getFormatHint(schema) || 'Type something…'}
   ```

2. **枚举搜索**: 多选枚举选项较多时，当前仅支持 typeahead，建议添加显式搜索框。

3. **表单进度指示**: 长表单应显示进度（如 "Field 3 of 10"）。

#### 6.3.3 代码质量

1. **类型安全**: `ElicitationDialog` 使用 `t0` 参数命名（React Compiler 产物），原始代码应使用具名参数。

2. **测试覆盖**: 关键交互逻辑（键盘导航、验证流程）应补充单元测试。

3. **文档**: 复杂的键盘交互协议应在代码中补充更详细的注释。

#### 6.3.4 架构建议

1. **提取通用逻辑**: `ElicitationFormDialog` 和 `ElicitationURLDialog` 有部分重复代码（如 AbortSignal 处理），可提取自定义 hook。

2. **状态机明确化**: URL 模式的 phase 状态机可使用更明确的类型定义：
   ```typescript
   type URLPhase = 
     | { type: 'prompt' }
     | { type: 'waiting'; startTime: number }
     | { type: 'completed'; result: 'success' | 'timeout' };
   ```

---

## 7. 附录

### 7.1 相关 MCP SDK 类型

```typescript
// @modelcontextprotocol/sdk/types.js
interface ElicitRequestFormParams {
  mode: 'form';
  message: string;
  requestedSchema: {
    properties: Record<string, PrimitiveSchemaDefinition>;
    required?: string[];
  };
}

interface ElicitRequestURLParams {
  mode: 'url';
  message: string;
  url: string;
  elicitationId?: string;
}

type ElicitResult = 
  | { action: 'accept'; content?: Record<string, unknown> }
  | { action: 'decline' }
  | { action: 'cancel' };
```

### 7.2 文件变更历史参考

- 虚拟滚动实现：解决大量字段时的性能问题
- ResolvingSpinner 提取：避免整个表单在异步验证时重渲染
- Typeahead 实现：支持枚举字段的快速导航
- 日期异步解析：添加自然语言日期支持

### 7.3 调试建议

```typescript
// 启用 MCP 调试日志
process.env.CLAUDE_CODE_DEBUG_MCP = '1';

// 查看 elicitation 队列状态
// 在 REPL.tsx 中添加：
console.log('Elicitation queue:', elicitation.queue);
```
