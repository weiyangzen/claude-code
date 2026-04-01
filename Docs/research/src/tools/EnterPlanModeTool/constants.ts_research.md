# constants.ts 研究文档

## 场景与职责

constants.ts 是 EnterPlanModeTool 的**常量定义模块**，负责声明工具相关的常量值。该模块遵循单一职责原则，将常量定义与业务逻辑分离，便于维护和引用。

### 核心职责
1. **工具名称定义**：声明 EnterPlanMode 工具的正式名称
2. **集中管理**：提供单一来源的常量引用，避免魔法字符串
3. **跨模块共享**：允许其他模块（如提示生成、权限系统）引用工具名称

---

## 功能点目的

### 1. 工具名称常量
- **常量名**：`ENTER_PLAN_MODE_TOOL_NAME`
- **值**：`'EnterPlanMode'`
- **目的**：
  - 在工具注册时使用
  - 在提示文本中引用
  - 在权限规则中识别该工具
  - 在分析日志中标识工具使用

---

## 具体技术实现

### 代码内容

```typescript
export const ENTER_PLAN_MODE_TOOL_NAME = 'EnterPlanMode'
```

### 使用场景

**1. 工具定义中**
```typescript
// EnterPlanModeTool.ts
import { ENTER_PLAN_MODE_TOOL_NAME } from './constants.js'

export const EnterPlanModeTool = buildTool({
  name: ENTER_PLAN_MODE_TOOL_NAME,
  // ...
})
```

**2. 提示文本中**
```typescript
// prompt.ts
import { ENTER_PLAN_MODE_TOOL_NAME } from './constants.js'

// 可用于动态构建提示内容
```

**3. 权限系统中**
```typescript
// 其他模块可能引用此名称进行权限检查
import { ENTER_PLAN_MODE_TOOL_NAME } from './constants.js'

// 例如：检查是否允许使用该工具
```

---

## 依赖与外部交互

### 导出内容

| 导出 | 类型 | 值 | 用途 |
|------|------|-----|------|
| `ENTER_PLAN_MODE_TOOL_NAME` | `string` | `'EnterPlanMode'` | 工具唯一标识符 |

### 引用方

该常量被以下模块引用：

1. **EnterPlanModeTool.ts** - 工具定义
2. **prompt.ts** - 提示内容（可能引用）
3. **其他工具模块** - 如需引用 EnterPlanMode
4. **权限系统** - 如需针对此工具设置规则

---

## 风险、边界与改进建议

### 已知限制

**1. 单一常量**
- 当前：仅定义工具名称
- 对比：其他工具可能有更多常量（如版本、限制值等）
- 评估：对于 EnterPlanModeTool 来说，单一常量足够，因为该工具相对简单

**2. 命名约定**
- 当前：使用大写下划线命名（`ENTER_PLAN_MODE_TOOL_NAME`）
- 符合：项目常量命名规范

### 改进建议

**1. 添加工具描述常量（可选）**
```typescript
// 如果需要多处使用相同的描述
export const ENTER_PLAN_MODE_DESCRIPTION = 
  'Requests permission to enter plan mode for complex tasks'
```

**2. 添加相关工具名称常量（可选）**
```typescript
// 如果经常与 ExitPlanMode 一起引用
export const EXIT_PLAN_MODE_TOOL_NAME = 'ExitPlanMode'  // 或从对应模块导入
```

**3. 类型安全（可选）**
```typescript
// 如果需要更严格的类型检查
export const ENTER_PLAN_MODE_TOOL_NAME = 'EnterPlanMode' as const
type EnterPlanModeToolName = typeof ENTER_PLAN_MODE_TOOL_NAME  // 'EnterPlanMode'
```

### 相关文件引用

```
src/tools/EnterPlanModeTool/
├── constants.ts            # 本文件 - 常量定义
├── EnterPlanModeTool.ts    # 工具主逻辑，引用本常量
├── UI.tsx                  # UI 渲染
└── prompt.ts               # 提示内容
```

### 设计模式

该模块体现了以下设计原则：

1. **DRY（Don't Repeat Yourself）**：避免魔法字符串重复
2. **单一职责**：仅负责常量定义
3. **显式导出**：明确声明哪些值是公开的

### 文件大小说明

该文件非常小（57 bytes），这是有意为之：
- 简单工具不需要复杂常量
- 保持模块职责单一
- 便于 tree-shaking 优化
