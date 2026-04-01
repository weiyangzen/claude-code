# constants.ts 深度研究文档

## 文件元数据
- **路径**: `src/tools/ExitPlanModeTool/constants.ts`
- **大小**: 113 bytes
- **类型**: TypeScript 常量定义
- **所属模块**: ExitPlanModeTool

---

## 1. 场景与职责

### 1.1 核心定位
`constants.ts` 是 `ExitPlanModeTool` 模块的**常量定义文件**，负责集中管理工具名称常量，确保：

1. **名称一致性**: 在工具定义、导入、引用处使用相同的名称
2. **可维护性**: 修改名称只需改动一处
3. **类型安全**: 通过 TypeScript 常量获得编译时检查

### 1.2 历史背景

文件中定义了两个常量：
- `EXIT_PLAN_MODE_TOOL_NAME`: 旧版工具名称（保留用于向后兼容）
- `EXIT_PLAN_MODE_V2_TOOL_NAME`: 当前使用的 V2 版本工具名称

两个常量值相同（`'ExitPlanMode'`），说明：
1. V2 版本完全替代了旧版本
2. 从外部视角（API/模型）看，工具名称未变
3. 内部实现已升级到 V2

---

## 2. 功能点目的

### 2.1 常量定义

| 常量名 | 值 | 用途 |
|-------|-----|------|
| `EXIT_PLAN_MODE_TOOL_NAME` | `'ExitPlanMode'` | 旧版工具名称（向后兼容） |
| `EXIT_PLAN_MODE_V2_TOOL_NAME` | `'ExitPlanMode'` | 当前 V2 版本工具名称 |

### 2.2 使用场景

```typescript
// 在 ExitPlanModeV2Tool.ts 中
import { EXIT_PLAN_MODE_V2_TOOL_NAME } from './constants.js'

export const ExitPlanModeV2Tool: Tool<...> = buildTool({
  name: EXIT_PLAN_MODE_V2_TOOL_NAME,  // 使用常量
  // ...
})
```

```typescript
// 在其他文件中引用工具名称
import { EXIT_PLAN_MODE_V2_TOOL_NAME } from '../tools/ExitPlanModeTool/constants.js'

// 用于工具查找、日志记录、分析事件等
if (block.name === EXIT_PLAN_MODE_V2_TOOL_NAME) {
  // 处理 ExitPlanMode 工具调用
}
```

---

## 3. 具体技术实现

### 3.1 代码实现

```typescript
export const EXIT_PLAN_MODE_TOOL_NAME = 'ExitPlanMode'
export const EXIT_PLAN_MODE_V2_TOOL_NAME = 'ExitPlanMode'
```

### 3.2 设计特点

1. **导出方式**: 使用命名导出（named exports），支持 tree-shaking
2. **值类型**: 字符串字面量类型，获得精确类型推断
3. **命名规范**: 
   - 全大写 + 下划线分隔（SCREAMING_SNAKE_CASE）
   - 清晰表达常量语义

---

## 4. 关键代码路径与文件引用

### 4.1 被导入位置

| 文件路径 | 导入的常量 | 用途 |
|---------|-----------|------|
| `src/tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts` | `EXIT_PLAN_MODE_V2_TOOL_NAME` | 工具定义中的 name 字段 |
| `src/tools/ExitPlanModeTool/prompt.ts` | 无（硬编码 AskUserQuestion） | 提示词中引用相关工具 |
| `src/utils/plans.ts` | `EXIT_PLAN_MODE_V2_TOOL_NAME` | 计划恢复时识别工具调用 |

### 4.2 引用代码片段

#### 4.2.1 工具定义中使用
```typescript
// ExitPlanModeV2Tool.ts
import { EXIT_PLAN_MODE_V2_TOOL_NAME } from './constants.js'

export const ExitPlanModeV2Tool = buildTool({
  name: EXIT_PLAN_MODE_V2_TOOL_NAME,
  // ...
})
```

#### 4.2.2 计划恢复中使用
```typescript
// plans.ts
import { EXIT_PLAN_MODE_V2_TOOL_NAME } from '../tools/ExitPlanModeTool/constants.js'

// 在 recoverPlanFromMessages 函数中
if (block.name === EXIT_PLAN_MODE_V2_TOOL_NAME) {
  const plan = input?.plan
  if (typeof plan === 'string' && plan.length > 0) {
    return plan
  }
}
```

---

## 5. 依赖与外部交互

### 5.1 模块关系

```
constants.ts
├── 被导入: ExitPlanModeV2Tool.ts
│   └── 用途: 工具名称定义
├── 被导入: plans.ts
│   └── 用途: 计划恢复逻辑
└── 被导入: 其他需要引用工具名称的文件
```

### 5.2 与其他工具常量的关系

```
src/tools/
├── ExitPlanModeTool/
│   └── constants.ts      # EXIT_PLAN_MODE_V2_TOOL_NAME = 'ExitPlanMode'
├── EnterPlanModeTool/
│   └── constants.ts      # ENTER_PLAN_MODE_TOOL_NAME = 'EnterPlanMode'
├── AgentTool/
│   └── constants.ts      # AGENT_TOOL_NAME = 'Agent'
└── ...
```

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 常量重复定义风险
- **风险**: 两个常量值相同，可能导致混淆
- **现状**: `EXIT_PLAN_MODE_TOOL_NAME` 和 `EXIT_PLAN_MODE_V2_TOOL_NAME` 都等于 `'ExitPlanMode'`
- **建议**: 考虑废弃旧常量，统一使用 V2 版本

#### 6.1.2 命名冲突风险
- **风险**: 如果未来有 V3 版本，工具名称可能需要变更
- **缓解**: 当前命名已包含版本信息（V2），便于未来扩展

### 6.2 边界情况

| 情况 | 处理 |
|------|------|
| 常量值修改 | 需要同步修改所有引用位置（虽然集中定义，但运行时依赖字符串值） |
| 工具重命名 | 需要修改常量值，并考虑向后兼容 |

### 6.3 改进建议

#### 6.3.1 代码清理
```typescript
// 建议: 废弃旧常量，仅保留 V2 版本
/** @deprecated Use EXIT_PLAN_MODE_V2_TOOL_NAME instead */
export const EXIT_PLAN_MODE_TOOL_NAME = 'ExitPlanMode'
export const EXIT_PLAN_MODE_V2_TOOL_NAME = 'ExitPlanMode'
```

#### 6.3.2 类型安全增强
```typescript
// 建议: 使用 const assertion 获得更精确的类型
export const EXIT_PLAN_MODE_V2_TOOL_NAME = 'ExitPlanMode' as const

// 这样可以使用 typeof 提取字面量类型
type ExitPlanModeToolName = typeof EXIT_PLAN_MODE_V2_TOOL_NAME
```

#### 6.3.3 统一常量管理
```typescript
// 建议: 考虑统一所有工具名称常量到一个文件
// src/constants/tools.ts
export const TOOL_NAMES = {
  EXIT_PLAN_MODE: 'ExitPlanMode',
  ENTER_PLAN_MODE: 'EnterPlanMode',
  AGENT: 'Agent',
  // ...
} as const
```

---

## 7. 相关文件索引

### 7.1 同目录文件
- `ExitPlanModeV2Tool.ts` - 工具主逻辑
- `UI.tsx` - UI 渲染组件
- `prompt.ts` - 工具提示词

### 7.2 引用文件
- `src/utils/plans.ts` - 计划恢复逻辑
- `src/constants/tools.ts` - 可能存在的工具常量汇总

### 7.3 类似常量文件
- `src/tools/EnterPlanModeTool/constants.ts`
- `src/tools/AgentTool/constants.ts`
- `src/tools/TeamCreateTool/constants.ts`
- `src/tools/SendMessageTool/constants.ts`
