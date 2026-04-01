# toolName.ts 深度研究文档

## 场景与职责

`toolName.ts` 是 Claude Code CLI 中 BashTool 的极简常量定义文件，其核心目的是**打破循环依赖**。虽然内容极其简单，但在整个项目的模块依赖图中扮演着关键角色。

### 核心职责

1. **常量定义**：定义 `BASH_TOOL_NAME = 'Bash'` 常量
2. **循环依赖打破**：避免从 `prompt.ts` 导入导致的循环依赖问题
3. **统一引用**：为整个项目提供一致的 Bash 工具名称引用点

## 功能点目的

### 为什么需要单独文件？

在原始设计中，`BASH_TOOL_NAME` 可能直接定义在 `prompt.ts` 或其他文件中。但随着代码演进，出现了以下依赖关系：

```
prompt.ts → 导出 BASH_TOOL_NAME
    ↓
BashTool.ts 导入 BASH_TOOL_NAME
    ↓
其他文件导入 BashTool
    ↓
... 最终依赖回 prompt.ts
```

这种循环依赖会导致：
- 模块加载顺序不确定
- 运行时错误（undefined 导入）
- 打包工具（如 Bun）的优化问题

### 解决方案

将 `BASH_TOOL_NAME` 提取到独立的 `toolName.ts` 文件：

```typescript
// Here to break circular dependency from prompt.ts
export const BASH_TOOL_NAME = 'Bash'
```

这样依赖图变为：

```
toolName.ts （无依赖，最先加载）
    ↓
prompt.ts, BashTool.ts, bashPermissions.ts, sandbox-adapter.ts 等同时导入
```

## 具体技术实现

### 代码内容

```typescript
// 行1：注释说明存在目的
// Here to break circular dependency from prompt.ts

// 行2：常量导出
export const BASH_TOOL_NAME = 'Bash'
```

### 使用场景

该常量在整个项目中被广泛引用：

| 文件 | 用途 |
|------|------|
| `sandbox-adapter.ts` | 在 `convertToSandboxRuntimeConfig` 中识别 Bash 工具权限规则 |
| `bashPermissions.ts` | 在权限检查和规则建议中使用 |
| `prompt.ts` | 原始位置，现在改为导入 |
| 其他权限相关文件 | 统一识别 Bash 工具 |

### 常量值说明

`'Bash'` 是 Claude Code CLI 中 Bash 工具的标识符，用于：
- 权限规则的格式：`Bash(command)` 或 `Bash(prefix:*)`
- 工具注册和识别
- 日志和事件追踪
- 配置文件的解析

## 关键代码路径与文件引用

### 导入关系

```
toolName.ts
    ↓ 被导入
├── src/tools/BashTool/prompt.ts
├── src/tools/BashTool/bashPermissions.ts
├── src/utils/sandbox/sandbox-adapter.ts
└── 其他使用 BASH_TOOL_NAME 的文件
```

### 相关文件

```
src/tools/BashTool/
├── toolName.ts              # 本文件
├── prompt.ts                # 原始位置，现在导入 toolName.ts
├── bashPermissions.ts       # 使用 BASH_TOOL_NAME 进行权限检查
└── ...

src/utils/sandbox/
└── sandbox-adapter.ts       # 使用 BASH_TOOL_NAME 识别 Bash 规则
```

## 依赖与外部交互

### 无外部依赖

`toolName.ts` 是一个纯常量文件，**不导入任何其他模块**：

```typescript
// 文件内容完全独立
export const BASH_TOOL_NAME = 'Bash'
```

这使得它在模块加载顺序中处于最底层，可以被任何其他模块安全导入而不会引入循环依赖。

### 导出内容

| 导出 | 类型 | 值 | 说明 |
|------|------|-----|------|
| `BASH_TOOL_NAME` | `const string` | `'Bash'` | Bash 工具标识符 |

## 风险、边界与改进建议

### 风险分析

1. **极低风险**
   - 文件内容简单，无复杂逻辑
   - 无外部依赖，不会引入循环依赖
   - 常量值稳定，不会频繁变更

2. **潜在风险**
   - 如果常量值需要变更（如重命名工具），需要全局替换
   - 文件虽小，但增加了模块数量

### 边界情况

1. **无边界情况**
   - 纯常量导出，无运行时行为
   - 无输入验证需求
   - 无错误处理需求

### 改进建议

1. **当前设计已足够**
   - 对于单一常量的场景，单独文件是合理的
   - 注释清晰说明了存在目的

2. **可选扩展**
   ```typescript
   // 如果未来需要更多 Bash 工具相关常量
   export const BASH_TOOL_NAME = 'Bash'
   export const BASH_TOOL_DESCRIPTION = 'Execute bash commands'
   export const BASH_TOOL_VERSION = '1.0'
   ```

3. **文档化**
   - 当前文件已有注释说明目的
   - 建议在项目架构文档中记录此类循环依赖解决方案

### 架构启示

该文件展示了处理循环依赖的一种常见模式：

```
问题：A.ts 和 B.ts 互相依赖
解决：提取共同依赖到 C.ts
    A.ts → C.ts
    B.ts → C.ts
```

这种模式在大型 TypeScript 项目中非常常见，值得在代码审查和架构设计中识别和应用。
