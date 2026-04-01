# constants.ts 研究文档

## 场景与职责

constants.ts 是 SendMessageTool 的常量定义文件，负责定义工具的名称常量。这是该文件的唯一职责，保持简单和单一。

## 功能点目的

### 1. 工具名称常量
- 定义 `SEND_MESSAGE_TOOL_NAME` 常量，值为 `'SendMessage'`
- 用于工具注册、查找和引用

## 具体技术实现

### 代码内容
```typescript
export const SEND_MESSAGE_TOOL_NAME = 'SendMessage'
```

### 使用场景

1. **工具定义** (`SendMessageTool.ts`):
   ```typescript
   import { SEND_MESSAGE_TOOL_NAME } from './constants.js'
   
   export const SendMessageTool: Tool = buildTool({
     name: SEND_MESSAGE_TOOL_NAME,
     // ...
   })
   ```

2. **Mailbox 系统** (`teammateMailbox.ts`):
   ```typescript
   import { SEND_MESSAGE_TOOL_NAME } from '../tools/SendMessageTool/constants.js'
   
   // 用于标识消息来源或工具类型
   ```

## 关键代码路径与文件引用

### 核心文件
- `src/tools/SendMessageTool/constants.ts` - 常量定义（1 行）

### 引用该常量的文件
- `src/tools/SendMessageTool/SendMessageTool.ts` - 工具主实现
- `src/utils/teammateMailbox.ts` - Mailbox 系统

## 依赖与外部交互

无外部依赖，纯常量导出。

## 风险、边界与改进建议

### 风险评估

1. **极低风险**
   - 文件仅包含一个字符串常量
   - 修改常量值会影响工具名称，但不会影响功能逻辑

### 改进建议

1. **扩展常量定义（可选）**
   - 当前文件过于简单，可以考虑合并到主文件中
   - 或者扩展为包含其他相关常量：
     ```typescript
     export const SEND_MESSAGE_TOOL_NAME = 'SendMessage'
     export const BROADCAST_TARGET = '*'
     export const MAX_SUMMARY_LENGTH = 100
     ```

2. **保持现状**
   - 遵循项目中其他工具的惯例（如 `AgentTool/constants.ts`）
   - 为未来可能的扩展预留空间

### 代码组织

该文件体现了项目中工具模块的组织惯例：
```
tools/
  SendMessageTool/
    SendMessageTool.ts    # 主实现
    UI.tsx                # UI 组件
    constants.ts          # 常量
    prompt.ts             # 提示模板
```

这种分离使得：
- 常量可以在不加载主实现的情况下被引用（避免循环依赖）
- 工具名称在编译时确定，可用于类型推断
