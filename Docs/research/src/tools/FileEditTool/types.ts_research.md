# types.ts 研究文档

## 场景与职责

`types.ts` 是 `FileEditTool` 模块的类型定义中心，使用 Zod v4 进行运行时 schema 定义与静态类型推导。该文件不仅是 `FileEditTool.ts` 和 `utils.ts` 的类型契约基础，也被大量外部模块引用（如 UI 组件、hooks、权限请求组件、消息选择器等），是文件编辑功能在类型系统层面的“协议规范”。

核心职责：
- 定义 `FileEditTool` 的输入 schema（`inputSchema`）和输出 schema（`outputSchema`）。
- 通过 `z.output` / `z.infer` 导出对应的 TypeScript 类型（`FileEditInput`、`FileEditOutput`）。
- 定义 diff 相关的子 schema：`hunkSchema`（结构化 patch 的 hunk）和 `gitDiffSchema`（Git diff 元数据）。
- 提供 `EditInput`（不含 `file_path` 的单条编辑）和 `FileEdit`（运行时保证 `replace_all` 为布尔值）等辅助类型。

## 功能点目的

| 类型/Schema | 目的 |
|-------------|------|
| `inputSchema` | 校验模型传入的编辑参数：`file_path`、`old_string`、`new_string`、`replace_all`。 |
| `FileEditInput` | 输入 schema 通过 Zod 解析后的输出类型，供 `validateInput` 和 `call` 使用。 |
| `EditInput` | 去掉 `file_path` 的单条编辑类型，用于批量编辑场景（如 `FileEditToolDiff`）。 |
| `FileEdit` | 运行时类型，保证 `replace_all` 必为 `boolean`，用于内部 diff 计算。 |
| `hunkSchema` | 定义结构化 diff patch 中单个 hunk 的字段：`oldStart`、`oldLines`、`newStart`、`newLines`、`lines`。 |
| `gitDiffSchema` | 定义 Git diff 附件的元数据：文件名、状态、增删行数、patch 字符串、仓库信息等。 |
| `outputSchema` | 校验 `FileEditTool` 执行结果的数据结构，确保返回给模型/UI 的数据完整且类型安全。 |
| `FileEditOutput` | 输出 schema 的推断类型，被 UI 组件、hooks、消息处理器广泛消费。 |

## 具体技术实现

### 1. Zod v4 与懒加载 Schema

文件使用 `zod/v4` 和自定义的 `lazySchema` 工具函数包装所有 schema：

```ts
import { z } from 'zod/v4'
import { lazySchema } from '../../utils/lazySchema.js'
```

`lazySchema` 的作用是将 schema 的构建延迟到首次访问时，避免在模块顶层实例化大量复杂 schema 导致的启动性能损耗（尤其在工具数量众多的情况下）。

### 2. 输入 Schema

```ts
const inputSchema = lazySchema(() =>
  z.strictObject({
    file_path: z.string().describe('The absolute path to the file to modify'),
    old_string: z.string().describe('The text to replace'),
    new_string: z
      .string()
      .describe(
        'The text to replace it with (must be different from old_string)',
      ),
    replace_all: semanticBoolean(
      z.boolean().default(false).optional(),
    ).describe('Replace all occurrences of old_string (default false)'),
  }),
)
```

关键点：
- `z.strictObject`：拒绝未声明的额外字段，防止模型传入未知参数。
- `semanticBoolean`：来自 `src/utils/semanticBoolean.js`，对布尔值进行语义化预处理（可能支持 `"true"` / `"false"` 字符串等形式）。
- `replace_all` 默认 `false`，且通过 `semanticBoolean` 增强兼容性。

类型推导：
```ts
type InputSchema = ReturnType<typeof inputSchema>
export type FileEditInput = z.output<InputSchema>
```

注意注释说明：
> "Parsed output — what call() receives. z.output not z.input: with semanticBoolean the input side is unknown (preprocess accepts anything)."

由于 `semanticBoolean` 是 `preprocess`，`z.input` 会是 `unknown`，因此统一使用 `z.output` 作为内部类型。

### 3. 辅助编辑类型

```ts
export type EditInput = Omit<FileEditInput, 'file_path'>

export type FileEdit = {
  old_string: string
  new_string: string
  replace_all: boolean
}
```

- `EditInput` 用于多编辑场景（如 `FileEditToolDiff` 组件接收 `edits: FileEdit[]`）。
- `FileEdit` 是更严格的运行时类型，确保 `replace_all` 不会是 `undefined`。

### 4. Hunk Schema

```ts
export const hunkSchema = lazySchema(() =>
  z.object({
    oldStart: z.number(),
    oldLines: z.number(),
    newStart: z.number(),
    newLines: z.number(),
    lines: z.array(z.string()),
  }),
)
```

对应 `diff` 库生成的 `StructuredPatchHunk` 结构，被 `outputSchema` 和外部 diff 渲染组件共同使用。

### 5. Git Diff Schema

```ts
export const gitDiffSchema = lazySchema(() =>
  z.object({
    filename: z.string(),
    status: z.enum(['modified', 'added']),
    additions: z.number(),
    deletions: z.number(),
    changes: z.number(),
    patch: z.string(),
    repository: z
      .string()
      .nullable()
      .optional()
      .describe('GitHub owner/repo when available'),
  }),
)
```

用于远程模式（`CLAUDE_CODE_REMOTE`）下 `fetchSingleFileGitDiff` 返回的元数据封装。`repository` 字段为可选，用于标识 GitHub 仓库的 `owner/repo` 格式。

### 6. 输出 Schema

```ts
const outputSchema = lazySchema(() =>
  z.object({
    filePath: z.string().describe('The file path that was edited'),
    oldString: z.string().describe('The original string that was replaced'),
    newString: z.string().describe('The new string that replaced it'),
    originalFile: z.string().describe('The original file contents before editing'),
    structuredPatch: z.array(hunkSchema()).describe('Diff patch showing the changes'),
    userModified: z.boolean().describe('Whether the user modified the proposed changes'),
    replaceAll: z.boolean().describe('Whether all occurrences were replaced'),
    gitDiff: gitDiffSchema().optional(),
  }),
)
```

类型推导：
```ts
type OutputSchema = ReturnType<typeof outputSchema>
export type FileEditOutput = z.infer<OutputSchema>
```

注意：输出 schema 使用 `z.object` 而非 `z.strictObject`，因此允许额外字段（向前兼容）。

## 关键代码路径与文件引用

- **定义文件**：`src/tools/FileEditTool/types.ts`
- **直接引用方**（部分列举）：
  - `src/tools/FileEditTool/FileEditTool.ts` — `FileEditInput`、`FileEditOutput`
  - `src/tools/FileEditTool/utils.ts` — `FileEdit`、`EditInput`
  - `src/tools/FileEditTool/UI.tsx` — `FileEditOutput`
  - `src/tools/FileWriteTool/FileWriteTool.ts` — `gitDiffSchema`、`hunkSchema`
  - `src/components/FileEditToolDiff.tsx` — `FileEdit`
  - `src/hooks/useDiffInIDE.ts` — `FileEdit`
  - `src/hooks/useTurnDiffs.ts` — `FileEditOutput`
  - `src/components/MessageSelector.tsx` — `FileEditOutput`
  - `src/utils/diff.ts` — `FileEdit`
  - `src/utils/api.ts` — `FileEditInput`（通过 `normalizeFileEditInput`）
  - `src/utils/sessionFileAccessHooks.ts` — `editInputSchema`（从 `inputSchema` 导入并重命名）

## 依赖与外部交互

| 依赖模块 | 交互方式 | 说明 |
|----------|----------|------|
| `zod/v4` | 库导入 | 运行时校验与类型推导。 |
| `utils/lazySchema.js` | 函数调用 | 延迟 schema 构建，优化启动性能。 |
| `utils/semanticBoolean.js` | 函数调用 | 增强布尔字段的语义兼容性。 |
| `FileEditTool.ts` | 类型消费 | 输入/输出类型约束工具实现。 |
| `FileWriteTool.ts` | Schema 消费 | 复用 `gitDiffSchema` 和 `hunkSchema`。 |
| `FileEditToolDiff.tsx` | 类型消费 | `FileEdit` 类型用于 diff 渲染。 |
| `useTurnDiffs.ts` / `MessageSelector.tsx` | 类型消费 | `FileEditOutput` 用于消息结果展示。 |

## 风险、边界与改进建议

### 风险与边界

1. **`semanticBoolean` 导致的 `z.input` 丢失**：由于 `preprocess` 的存在，`z.input<InputSchema>` 为 `unknown`，这意味着无法对 API 原始输入进行静态类型约束。虽然当前通过 `z.output` 规避了问题，但在需要精确知道“模型可能发什么”的场景（如提示工程、测试 fixture）中不够直观。
2. **`FileEdit` 与 `EditInput` 的语义重叠**：两者结构几乎相同，只是 `replace_all` 的可选性不同。这种细微差别增加了开发者选择类型的认知负担，也可能导致类型转换时的冗余断言。
3. **`gitDiffSchema` 的 `status` 枚举不完整**：仅包含 `'modified'` 和 `'added'`，未包含 `'deleted'`、`'renamed'` 等 Git 标准状态。虽然当前 `fetchSingleFileGitDiff` 可能只返回这两种，但 schema 的狭窄定义会在未来扩展时成为阻碍。
4. **输出 schema 未使用 `strictObject`**：允许额外字段在向前兼容上有好处，但也意味着如果工具实现意外多返回了字段，Zod 不会报错，可能掩盖数据泄露或结构错误。
5. **`lazySchema` 的线程/并发安全性**：`lazySchema` 内部通常使用模块级变量做缓存，若在首次访问时发生并发调用，可能触发多次 schema 构建（虽然无状态，但存在微小竞态窗口）。

### 改进建议

1. **统一编辑类型，减少碎片化**：考虑将 `EditInput` 和 `FileEdit` 合并为一个类型，通过 `Required<Pick<...>>` 或泛型参数控制 `replace_all` 的可选性，例如：
   ```ts
   export type FileEdit<ReplaceAll = boolean> = {
     old_string: string
     new_string: string
     replace_all: ReplaceAll
   }
   export type EditInput = FileEdit<boolean | undefined>
   ```
2. **扩展 `gitDiffSchema` 的 `status` 枚举**：预先加入 `'deleted'`、`'renamed'`、`'copied'` 等标准 Git 状态，即使当前未使用，也能避免未来 schema 变更的破坏性更新。
3. **评估输出 schema 是否应启用 `strictObject`**：在开发/测试环境中使用严格模式，生产环境可放宽。或者增加一个 `passthrough()` 与 `strict()` 的双版本 schema 用于不同环境。
4. **为输入 schema 增加 `old_string !== new_string` 的 refine 校验**：当前该校验在 `FileEditTool.ts` 的 `validateInput` 中以运行时逻辑实现。若将其下沉到 Zod schema 的 `.refine()` 中，可让类型系统与运行时校验更紧密地结合，同时改善 API 错误反馈结构。
5. **文档化 `lazySchema` 的缓存语义**：在 `utils/lazySchema.ts` 中增加注释，说明其线程安全假设和缓存失效策略（如果有），便于其他工具开发者正确使用。
