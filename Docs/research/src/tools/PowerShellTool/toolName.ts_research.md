# PowerShellTool/toolName.ts 深度研究文档

## 一、场景与职责

### 1.1 核心定位

`toolName.ts` 是 PowerShell 工具中最简单的模块，仅包含一个常量定义：`POWERSHELL_TOOL_NAME`。它的主要目的是 **打破循环依赖**。

### 1.2 存在的必要性

在模块化设计中，循环依赖是一个常见问题：

```
A.ts ──imports──> B.ts
  ↑               │
  └───imports─────┘
```

在 PowerShell 工具中，具体的循环依赖场景是：

```
prompt.ts ──imports──> readOnlyValidation.ts ──imports──> ...
    ↑                                                      │
    └────────────────── 需要获取工具名称 ────────────────────┘
```

`prompt.ts` 需要工具名称来生成提示信息，但如果工具名称定义在 `prompt.ts` 或依赖 `prompt.ts` 的模块中，就会形成循环依赖。

### 1.3 设计模式

这是 **"Dependency Inversion"（依赖倒置）** 模式的一种应用：

1. 将共享常量提取到独立的低层模块
2. 所有需要该常量的模块都依赖这个低层模块
3. 避免高层模块之间的相互依赖

---

## 二、功能点目的

### 2.1 POWERSHELL_TOOL_NAME 常量

```typescript
export const POWERSHELL_TOOL_NAME = 'PowerShell' as const
```

**设计要点**：

| 特性 | 说明 | 目的 |
|------|------|------|
| `as const` | TypeScript 常量断言 | 确保类型为字面量 `'PowerShell'` 而非 `string`，支持精确类型匹配 |
| 大写命名 | 常量命名规范 | 表明这是不可变的配置值 |
| 单独文件 | 单职责原则 | 最小化依赖，避免循环引用 |

### 2.2 使用场景

该常量在整个 PowerShell 工具中被广泛使用：

| 使用位置 | 用途 |
|----------|------|
| `powershellPermissions.ts` | 生成权限检查错误消息、规则匹配 |
| `prompt.ts` | 生成用户提示信息 |
| 其他模块 | 任何需要引用工具名称的场景 |

**代码示例**（来自 `powershellPermissions.ts`）：

```typescript
import { POWERSHELL_TOOL_NAME } from './toolName.js'

// 生成拒绝消息
return {
  behavior: 'deny',
  message: `Permission to use ${POWERSHELL_TOOL_NAME} with command ${command} has been denied.`,
  decisionReason: { type: 'rule', rule: matchingDenyRules[0] },
}

// 生成权限请求消息
return {
  behavior: 'ask',
  message: createPermissionRequestMessage(POWERSHELL_TOOL_NAME),
  decisionReason: { type: 'rule', rule: matchingAskRules[0] },
}
```

---

## 三、具体技术实现

### 3.1 代码分析

```typescript
// Here to break circular dependency from prompt.ts
export const POWERSHELL_TOOL_NAME = 'PowerShell' as const
```

**注释说明**：
- 明确指出了文件存在的原因：打破来自 `prompt.ts` 的循环依赖
- 这是维护性注释，帮助开发者理解设计意图

### 3.2 TypeScript 类型推断

| 写法 | 推断类型 | 说明 |
|------|----------|------|
| `const x = 'PowerShell'` | `'PowerShell'` | 基础类型推断 |
| `const x = 'PowerShell' as const` | `'PowerShell'` | 显式常量断言 |
| `let x = 'PowerShell'` | `string` | 可变变量推断 |

使用 `as const` 的好处：
1. **类型安全**：防止意外修改（虽然 const 已经防止了重新赋值）
2. **精确匹配**：在类型级别确保是特定的字面量值
3. **IDE 支持**：更好的自动完成和重构支持

### 3.3 模块导出策略

```typescript
export const POWERSHELL_TOOL_NAME = 'PowerShell' as const
```

- **命名导出**（Named Export）：使用 `export const`
- 消费者使用 `import { POWERSHELL_TOOL_NAME } from './toolName.js'`

为什么不使用默认导出？
```typescript
// 默认导出（不推荐）
export default 'PowerShell'
// 消费：import toolName from './toolName.js'

// 命名导出（实际使用）
export const POWERSHELL_TOOL_NAME = 'PowerShell' as const
// 消费：import { POWERSHELL_TOOL_NAME } from './toolName.js'
```

命名导出的优势：
1. **显式性**：调用方必须明确知道导入的名称
2. **Tree Shaking**：更好的静态分析支持
3. **一致性**：与其他工具（BashTool 等）的命名风格保持一致

---

## 四、关键代码路径与文件引用

### 4.1 模块依赖关系

```
toolName.ts (低层常量模块)
    ↑
    ├── 被导入 ──> prompt.ts
    ├── 被导入 ──> powershellPermissions.ts
    ├── 被导入 ──> 其他需要工具名称的模块
    │
    └── 不依赖任何其他 PowerShellTool 模块（打破循环的关键）
```

### 4.2 导入方示例

#### 4.2.1 prompt.ts
```typescript
import { POWERSHELL_TOOL_NAME } from './toolName.js'
// 使用 POWERSHELL_TOOL_NAME 生成提示
```

#### 4.2.2 powershellPermissions.ts
```typescript
import { POWERSHELL_TOOL_NAME } from './toolName.js'
// 使用 POWERSHELL_TOOL_NAME 生成权限相关消息
```

### 4.3 与 BashTool 的对比

| 方面 | PowerShellTool | BashTool |
|------|----------------|----------|
| 工具名称常量 | `POWERSHELL_TOOL_NAME = 'PowerShell'` | `BASH_TOOL_NAME = 'Bash'` |
| 文件位置 | `toolName.ts` | `toolName.ts` |
| 存在原因 | 打破循环依赖 | 打破循环依赖 |
| 设计模式 | 完全相同 | 完全相同 |

这种对称设计表明这是项目范围内的统一架构决策。

---

## 五、依赖与外部交互

### 5.1 上游依赖

**无**。这是设计上的刻意选择：

```typescript
// 文件内容只有：
export const POWERSHELL_TOOL_NAME = 'PowerShell' as const
```

没有 `import` 语句，确保：
1. 不会引入循环依赖
2. 加载顺序无关
3. 最小化启动开销

### 5.2 下游调用方

| 调用方 | 用途 |
|--------|------|
| `prompt.ts` | 生成用户提示和确认消息 |
| `powershellPermissions.ts` | 权限检查错误消息、规则匹配 |
| 可能的其他模块 | 日志、遥测、调试信息等 |

### 5.3 跨工具一致性

项目中的其他工具也遵循相同的模式：

```typescript
// BashTool/toolName.ts
export const BASH_TOOL_NAME = 'Bash' as const

// PowerShellTool/toolName.ts  
export const POWERSHELL_TOOL_NAME = 'PowerShell' as const

// 其他工具...
```

---

## 六、风险、边界与改进建议

### 6.1 风险评估

| 风险 | 等级 | 说明 |
|------|------|------|
| 循环依赖 | 低 | 该文件专门用于解决此问题 |
| 命名不一致 | 低 | 常量名称清晰，与文件路径一致 |
| 值被篡改 | 极低 | `const` + 模块系统保护 |
| 国际化问题 | 中 | 硬编码英文名称，但属于内部标识符 |

### 6.2 边界情况

| 场景 | 行为 |
|------|------|
| 重复导入 | 正常，ES 模块系统处理 |
| 动态导入 | 支持 `import('./toolName.js')` |
| 测试 mock | 可能需要重新设计以支持 mock |

### 6.3 改进建议

#### 建议 1：考虑集中式工具注册表
**当前**：每个工具有自己的 `toolName.ts`
**建议**：考虑在更高层创建工具注册表

```typescript
// 可能的改进
// tools/registry.ts
export const TOOL_NAMES = {
  bash: 'Bash',
  powershell: 'PowerShell',
  // ...
} as const
```

**权衡**：
- 优点：集中管理，便于查看所有工具
- 缺点：可能引入新的依赖关系

#### 建议 2：添加工具元数据
**当前**：只有名称字符串
**建议**：扩展为工具元数据对象

```typescript
export const POWERSHELL_TOOL_METADATA = {
  name: 'PowerShell',
  displayName: 'PowerShell',
  description: 'Execute PowerShell commands',
  platforms: ['windows', 'linux', 'macos'],
  // ...
} as const
```

**注意**：这可能需要重新考虑文件位置和依赖关系。

#### 建议 3：文档化循环依赖图
**建议**：在项目文档中维护循环依赖图，帮助开发者理解架构约束。

```
循环依赖说明：
- prompt.ts -> readOnlyValidation.ts -> ... -> prompt.ts
- 解决方案：提取 POWERSHELL_TOOL_NAME 到 toolName.ts
```

### 6.4 测试建议

虽然该模块非常简单，但仍建议：

| 测试类型 | 验证点 |
|----------|--------|
| 类型测试 | `POWERSHELL_TOOL_NAME` 类型为 `'PowerShell'` |
| 值测试 | 值为 `'PowerShell'` |
| 导入测试 | 可从其他模块正确导入 |
| 不可变性 | 尝试重新赋值应导致编译错误 |

---

## 七、总结

`toolName.ts` 是一个 **看似简单但架构上重要** 的模块：

1. **单一职责**：只定义工具名称常量
2. **打破循环**：解决 `prompt.ts` 与其他模块的循环依赖
3. **类型安全**：使用 `as const` 确保精确类型
4. **设计模式**：依赖倒置原则的具体应用
5. **跨工具一致**：与 BashTool 等其他工具保持相同模式

虽然代码只有两行，但它体现了良好的软件工程实践：**通过适当的模块划分解决依赖问题**，而不是通过复杂的依赖注入或运行时解决方案。
