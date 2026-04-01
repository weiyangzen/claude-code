# constants.ts 研究文档

## 场景与职责

`constants.ts` 是 `TaskGetTool` 模块的**常量定义文件**，职责极其单一且明确：导出工具的唯一名称字符串常量 `TASK_GET_TOOL_NAME`。该文件的存在意义在于消除代码中的“魔法字符串”，为工具注册、权限控制、分类器决策、延迟加载匹配等场景提供类型安全的引用点。

## 功能点目的

1. **集中定义工具标识符**：将 `TaskGet` 这一名称作为模块级常量导出，避免在注册、权限、提示词等各处硬编码字符串。
2. **支持跨模块引用**：`src/constants/tools.ts`、`src/utils/permissions/classifierDecision.ts`、`src/utils/swarm/inProcessRunner.ts` 等多个文件通过导入该常量来引用 TaskGet 工具，确保命名一致性。
3. **便于重构与重命名**：若未来需要调整工具名称，只需修改此一处常量，所有依赖点自动同步（在 TypeScript 编译期即可捕获未同步的引用）。

## 具体技术实现

文件内容仅一行有效导出：

```typescript
export const TASK_GET_TOOL_NAME = 'TaskGet'
```

这是一个普通的字符串常量，无运行时逻辑、无状态、无副作用。模块加载时即完成求值，内存开销可忽略。

## 关键代码路径与文件引用

### 本文件
- `src/tools/TaskGetTool/constants.ts` — 常量定义源文件。

### 直接引用方
- `src/tools/TaskGetTool/TaskGetTool.ts` — 在 `buildTool({ name: TASK_GET_TOOL_NAME, ... })` 中作为工具名称使用。
- `src/tools.ts` — 导入 `TaskGetTool` 本身，不直接引用常量，但工具名称最终来源于此。
- `src/constants/tools.ts` — 导入 `TASK_GET_TOOL_NAME`，用于：
  - 定义 `IN_PROCESS_TEAMMATE_ALLOWED_TOOLS`（允许 in-process teammates 使用的工具集合）。
- `src/utils/permissions/classifierDecision.ts` — 导入 `TASK_GET_TOOL_NAME`，用于：
  - 定义 `SAFE_YOLO_ALLOWLISTED_TOOLS`（YOLO 自动模式下无需分类器检查的安全工具白名单）。
- `src/utils/swarm/inProcessRunner.ts` — 导入 `TASK_GET_TOOL_NAME`，用于：
  - 在 `resolvedAgentDefinition.tools` 中强制注入 teammate 必备工具白名单。

### 间接引用方
- `src/utils/permissions/permissions.ts` 及相关权限系统 — 通过 `src/constants/tools.ts` 中的集合间接判断工具权限。
- `src/hooks/useCanUseTool.ts`、`src/tools/ToolSearchTool/ToolSearchTool.ts` 等 — 通过运行时工具名称匹配间接涉及。

## 依赖与外部交互

该文件**无任何外部依赖**，也不产生任何外部交互。它是一个纯常量模块，遵循项目内常见的“每个工具一个 `constants.ts`”的约定。

## 风险、边界与改进建议

### 风险

1. **命名漂移风险极低但需警惕**：若有人直接在某个文件中硬编码 `'TaskGet'` 而不引用 `TASK_GET_TOOL_NAME`，会导致重命名时不同步。不过项目整体通过 `satisfies ToolDef` 和 TypeScript 类型检查，已大幅降低了此类风险。
2. **无类型保护**：当前导出的是 `string` 类型而非字面量类型 `'TaskGet'`。虽然 TypeScript 在 `buildTool` 的 `name` 字段处会推断出具体字面量，但跨模块引用时类型会拓宽为 `string`，在某些严格依赖字面量类型的场景中可能失去精确性。

### 边界

- **无运行时边界**：该文件不涉及任何条件分支、环境变量或平台差异。
- **无扩展性边界**：作为常量文件，其内容天然受限，不适合承载复杂逻辑。若未来需要为 TaskGet 增加版本号、别名等元数据，应考虑扩展为配置对象或迁移到专门的元数据文件。

### 改进建议

1. **字面量类型化**：可将导出声明改为：
   ```typescript
   export const TASK_GET_TOOL_NAME = 'TaskGet' as const
   ```
   这样 `typeof TASK_GET_TOOL_NAME` 会推断为字面量 `'TaskGet'`，在需要精确匹配的集合类型（如 `Set<typeof TASK_GET_TOOL_NAME>`）中能提供更强的类型安全。
2. **统一常量文件模式**：项目内存在部分工具将常量放在 `prompt.ts` 中（如 `ASK_USER_QUESTION_TOOL_NAME`），而 TaskGetTool 等则使用独立的 `constants.ts`。建议统一为独立 `constants.ts` 模式，保持模块职责清晰。
3. **别名支持（可选）**：若未来 TaskGet 需要重命名但又要保持向后兼容，可在此文件中增加 `TASK_GET_TOOL_ALIASES` 数组，并在 `buildTool` 的 `aliases` 字段中引用，避免旧提示中的名称匹配失效。
