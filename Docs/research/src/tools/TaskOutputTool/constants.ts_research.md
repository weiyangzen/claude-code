# 研究文档：src/tools/TaskOutputTool/constants.ts

## 场景与职责

`constants.ts` 是 `TaskOutputTool` 模块的**单一常量出口文件**，职责极其明确：
- 定义并导出 `TaskOutputTool` 的**规范内部名称字符串** `TASK_OUTPUT_TOOL_NAME`。
- 作为整个代码库中所有引用 `TaskOutputTool` 名称的**单一事实来源（Single Source of Truth）**，避免魔法字符串散落各处。

该文件虽小，但在架构上承担了"工具名解耦"的关键角色：
- `TaskOutputTool.tsx` 通过导入该常量设置 `buildTool({ name: TASK_OUTPUT_TOOL_NAME })`。
- 权限系统、消息系统、工具注册中心、分类器决策、常量聚合文件等数十处代码均通过导入此常量来识别或过滤该工具，而不是硬编码 `'TaskOutput'`。

---

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| `export const TASK_OUTPUT_TOOL_NAME = 'TaskOutput'` | 提供类型安全、可追踪的工具名标识符 |
| 独立文件 | 让不依赖 React/TSX 运行时的纯逻辑模块（如权限解析器、消息处理器）能够零成本引用工具名，而不必拉入整个 `TaskOutputTool.tsx` 及其庞大的 UI/ink 依赖树 |

---

## 具体技术实现

### 1. 代码内容

```ts
export const TASK_OUTPUT_TOOL_NAME = 'TaskOutput'
```

仅一行导出，无任何运行时逻辑、无副作用、无依赖其他模块。

### 2. 设计模式：常量外置（Constants Extraction）

在 `src/tools/<ToolName>/` 目录下，几乎每个工具都遵循同一模式：
- `constants.ts`：存放工具名常量、可能的其他配置常量。
- `prompt.ts`：存放工具的系统提示文本、描述文本（部分工具）。
- 主实现文件（`*.tsx` 或 `*.ts`）：导入常量，避免自包含字符串。

这种模式的好处：
- **打破循环依赖**：`TaskOutputTool.tsx` 引用了大量外部模块（`diskOutput.ts`、`messages.ts`、`framework.ts` 等），如果这些外部模块又需要反向识别该工具名，直接导入 `TaskOutputTool.tsx` 会引入循环依赖。通过 `constants.ts` 这个"叶子节点"文件，循环依赖被打破。
- **Tree-shaking / DCE 友好**：对于只需要判断工具名的模块（如 `src/utils/permissions/permissionRuleParser.ts`），导入一个字符串常量的成本为零，打包器可轻松剔除未使用的代码。
- **重命名安全**：若未来需要调整 `TaskOutputTool` 的 wire name，只需修改此一处常量，所有引用点自动同步。

### 3. 与历史别名机制的配合

`src/utils/permissions/permissionRuleParser.ts` 中维护了一套旧名到新名的映射：

```ts
const LEGACY_TOOL_NAME_ALIASES: Record<string, string> = {
  Task: AGENT_TOOL_NAME,
  KillShell: TASK_STOP_TOOL_NAME,
  AgentOutputTool: TASK_OUTPUT_TOOL_NAME,
  BashOutputTool: TASK_OUTPUT_TOOL_NAME,
  // ...
}
```

这里引用的 `TASK_OUTPUT_TOOL_NAME` 即来自本文件。这意味着：
- 即使用户在权限规则中写了 `AgentOutputTool` 或 `BashOutputTool`，系统也会将其规范化为当前的 `'TaskOutput'`。
- 该常量不仅是当前名称的来源，也是历史兼容映射的锚点。

---

## 关键代码路径与文件引用

### 导入方（直接引用 `TASK_OUTPUT_TOOL_NAME`）

| 文件路径 | 引用行号 | 用途 |
|---------|---------|------|
| `src/tools/TaskOutputTool/TaskOutputTool.tsx` | 29 | 设置 `buildTool({ name: TASK_OUTPUT_TOOL_NAME, ... })` |
| `src/tools.ts` | 54 | 导入工具模块时同步导入名称（虽然 `tools.ts` 主要使用 `TaskOutputTool.name`，但名称常量仍被其他模块引用） |
| `src/constants/tools.ts` | 3 | 将 `TASK_OUTPUT_TOOL_NAME` 加入 `ALL_AGENT_DISALLOWED_TOOLS` 等集合 |
| `src/utils/permissions/permissionRuleParser.ts` | 3 | 旧名别名映射的目标值 |
| `src/utils/permissions/classifierDecision.ts` | 15 | YOLO 安全白名单 |
| `src/utils/messages.ts` | 144 | 消息处理中的工具名识别（如 `normalizeLegacyToolName` 后的比对） |
| `src/utils/api.ts` | 36 | 系统提示/schema 构建中的工具识别 |

### 间接影响路径

- `src/components/agents/ToolSelector.tsx` 虽然没有直接导入本文件，但它导入的是 `TaskOutputTool.name`，而该 `name` 字段的值最终溯源到本常量。因此本文件是工具选择器分类的间接事实来源。
- `src/utils/permissions/PermissionRule.ts` 及相关解析逻辑通过 `normalizeLegacyToolName()` 将旧别名解析为 `TASK_OUTPUT_TOOL_NAME`，影响权限规则的匹配结果。

---

## 依赖与外部交互

### 内部依赖
- **无**：本文件不导入任何其他模块，是依赖树中的"叶子节点"。

### 外部交互
- **编译时**：TypeScript 编译器将该常量内联到所有导入模块的引用点（取决于 `tsconfig` 的 `module` 设置，但通常保留为模块导入）。
- **运行时**：作为字符串常量，在 Node.js/Bun 运行时的内存中仅有一份字符串实例（若模块系统做缓存），各导入方共享同一引用。
- **与权限系统的交互**：`permissionRuleParser.ts` 在解析用户配置的权限规则时，将旧别名映射到该常量值，决定某条规则是否适用于 `TaskOutputTool`。
- **与工具注册表的交互**：`src/constants/tools.ts` 使用该常量构建 `ALL_AGENT_DISALLOWED_TOOLS` 和 `ASYNC_AGENT_ALLOWED_TOOLS`（后者未包含 `TaskOutputTool`，间接实现禁止）。

---

## 风险、边界与改进建议

### 风险

1. **命名与 wire name 不一致的潜在混淆**：
   - 类/变量名是 `TaskOutputTool`。
   - 常量值是 `'TaskOutput'`（没有 `Tool` 后缀）。
   - 旧别名是 `AgentOutputTool`、`BashOutputTool`。
   - 这种不一致可能导致新开发者在日志、权限规则或 MCP 工具名匹配时产生困惑。

2. **常量文件过于单薄带来的过度拆分**：
   - 仅含一个字符串常量的文件在目录中增加了模块数量，对构建系统的模块解析有微小开销。虽然现代打包器可很好处理，但在极端性能敏感场景（如 CLI 冷启动）中，大量 tiny 模块的 `require`/`import` 累积可能产生可测量的延迟。

3. **重命名风险**：
   - 若未来决定将该工具彻底移除（因其 deprecated），需同步清理本文件及所有导入方。由于导入方分散在权限、消息、API、常量聚合等多个子系统，遗漏任何一处都会导致运行时行为异常（如工具名匹配失败、白名单失效）。

### 边界

- **无运行时边界逻辑**：本文件本身不执行任何校验、截断或超时控制，所有边界（如超时上限 600s、输出截断 32K 字符）均在 `TaskOutputTool.tsx` 或 `outputFormatting.ts` 中处理。
- **字符串值不可配置**：`TASK_OUTPUT_TOOL_NAME` 是硬编码的编译期常量，无法通过环境变量或配置文件动态修改。这是有意的设计——LLM 看到的工具名必须在系统提示中稳定出现，动态变化会破坏 prompt-cache 和模型行为一致性。

### 改进建议

1. **考虑合并到同目录的 `prompt.ts` 或新建 `meta.ts`**：
   - 若 `TaskOutputTool` 目录下始终只有一个常量，可将其与工具描述文本、别名列表合并到一个 `meta.ts` 或 `prompt.ts` 中，减少文件碎片。
   - 但需评估是否会影响现有导入方（如 `permissionRuleParser.ts` 只导入常量，合并后可能拉入不必要的字符串资源）。

2. **增加 JSDoc 注释说明历史别名**：
   ```ts
   /** Canonical wire name for TaskOutputTool. Formerly known as AgentOutputTool / BashOutputTool. */
   export const TASK_OUTPUT_TOOL_NAME = 'TaskOutput'
   ```
   这能帮助新开发者快速理解名称历史，减少在权限规则或日志中遇到旧名时的困惑。

3. **若工具最终移除，制定清理清单**：
   - 由于 deprecated 状态已经明确，建议在项目文档或代码注释中维护一个"TaskOutputTool 移除检查清单"，列出必须清理的导入点：
     1. `src/tools.ts` 的 `getAllBaseTools()`
     2. `src/constants/tools.ts` 的禁止列表
     3. `src/utils/permissions/permissionRuleParser.ts` 的别名映射
     4. `src/utils/permissions/classifierDecision.ts` 的白名单
     5. `src/utils/messages.ts` 的工具名引用
     6. `src/utils/api.ts` 的引用
     7. `src/components/agents/ToolSelector.tsx` 的工具 bucket
   - 防止因遗漏而导致权限系统或分类器出现"幽灵工具"引用。

4. **考虑添加 `DEPRECATED` 前缀或注释**：
   - 在常量声明附近显式标注 deprecated，使得在 IDE 中查看导入点时更易识别：
     ```ts
     /** @deprecated TaskOutputTool is deprecated; prefer reading the task output file directly. */
     export const TASK_OUTPUT_TOOL_NAME = 'TaskOutput'
     ```
   - 这样 TypeScript 会在引用该常量的地方发出废弃警告，推动各导入方逐步迁移或清理。
