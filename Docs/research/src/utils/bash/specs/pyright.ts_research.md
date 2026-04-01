# pyright.ts 研究文档

## 场景与职责

`pyright.ts` 是本地 bash 命令规格库中内容最丰富的一个 spec，用于描述 Microsoft Pyright（Python 静态类型检查器）的命令行接口。Pyright 作为 Node.js 生态中广泛使用的 Python 工具，其 CLI 拥有大量选项（`--watch`、`-p`/`--project`、`--createstub`、`--pythonpath` 等）。该文件的存在确保 Claude Code 在解析 `pyright` 命令时，能够准确识别 flag、参数边界以及子命令结构，从而生成合理的安全前缀并支持潜在的自动补全。

与 `nohup.ts`、`time.ts` 等 wrapper spec 不同，`pyright` 是一个“叶子命令”spec——它自身不包裹其他命令，而是直接执行类型检查任务。因此其规格重点在于**选项矩阵**的完整描述，而非 `isCommand` 的 wrapper 语义。

## 功能点目的

1. **精确选项建模**：将 Pyright CLI 的 20+ 个选项逐一映射为 `Option[]`，包括短选项、长选项、是否带参数、参数名称等元数据。
2. **支持前缀深度计算**：`specPrefix.ts` 的 `calculateDepth` 和 `buildPrefix` 会利用 `options` 数组判断 flag 是否带值。例如 `-p /path/to/project` 中的 `/path/to/project` 会被识别为 `-p` 的参数而非子命令，从而正确跳过。
3. **支持文件路径参数识别**：`args` 声明为 `isVariadic: true, isOptional: true`，表示 `pyright` 接受零个或多个文件/目录路径作为待分析目标。`buildPrefix` 遇到路径参数时会根据 `shouldStopAtArg` 停止，前缀通常收敛到 `pyright` 本身（深度 2）。
4. **为安全校验提供上下文**：虽然 `pyright` 不是危险命令，但准确的 spec 能避免其被误分类为未知命令，从而减少不必要的用户确认提示。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 数据结构

导出一个匿名对象，通过 `satisfies CommandSpec` 约束：

```typescript
export default {
  name: 'pyright',
  description: 'Type checker for Python',
  options: [
    { name: ['--help', '-h'], description: 'Show help message' },
    { name: '--version', description: 'Print pyright version and exit' },
    { name: ['--watch', '-w'], description: 'Continue to run and watch for changes' },
    { name: ['--project', '-p'], description: 'Use the configuration file at this location', args: { name: 'FILE OR DIRECTORY' } },
    { name: '-', description: 'Read file or directory list from stdin' },
    { name: '--createstub', description: 'Create type stub file(s) for import', args: { name: 'IMPORT' } },
    // ... 更多选项
  ],
  args: {
    name: 'files',
    description: 'Specify files or directories to analyze (overrides config file)',
    isVariadic: true,
    isOptional: true,
  },
} satisfies CommandSpec
```

关键字段解析：
- `options`：共 19 个选项条目，涵盖帮助、版本、watch 模式、项目配置、typeshed 路径、Python 解释器路径、虚拟环境路径、JSON 输出、verbose、stats、依赖分析、诊断级别、线程数等。
- `args`：
  - `isVariadic: true`：允许分析多个路径，如 `pyright src/ tests/`。
  - `isOptional: true`：允许无路径参数（此时 Pyright 使用当前目录或 `pyrightconfig.json` 中的配置）。

### 关键流程

#### 1. 前缀提取流程

以 `pyright --project ./configs/pyright.json src/` 为例：
1. `getCommandPrefixStatic` 解析得到 argv `['pyright', '--project', './configs/pyright.json', 'src/']`。
2. `getCommandSpec('pyright')` 命中本地 spec。
3. `isWrapper` 为 `false`（无 `isCommand` 参数）。
4. `buildPrefix('pyright', args, spec)` 开始迭代：
   - `--project` 是已知 option，且 `option.args` 存在 → `flagTakesArg` 返回 `true`，跳过 `./configs/pyright.json`。
   - `src/` 是路径参数，`shouldStopAtArg` 返回 `true`（因为包含 `/`）。
5. 前缀收敛为 `pyright`（深度 2）。

#### 2. 与 `calculateDepth` 的交互

`specPrefix.ts` 的 `calculateDepth` 逻辑：
- `DEPTH_RULES` 中无 `pyright` 条目。
- `spec.args` 存在，且 `toArray(spec.args)` 中无 `isCommand`、无 `isVariadic`（虽然 `args.isVariadic` 为 true，但 `calculateDepth` 在 `spec.subcommands` 不存在时，对 `isVariadic` 返回 1）。
- 但 `spec.args` 的 `isOptional` 为 true，因此 `argsArray[0] && !argsArray[0].isOptional` 不成立。
- 最终返回默认值 2，前缀为 `pyright`。

**注意**：`calculateDepth` 中 `isVariadic` 分支返回 1 的条件是 `!spec.subcommands?.length`。对于 `pyright`，`spec.subcommands` 不存在，因此理论上会进入 `return 1`。但实际代码路径中 `argsArray[0] && !argsArray[0].isOptional` 为 false（因为 `isOptional: true`），所以不会走到该分支之前的 `return 2`。等等，让我重新检查逻辑：

```typescript
if (spec.args) {
  const argsArray = toArray(spec.args)
  if (argsArray.some(arg => arg?.isCommand)) { ... }
  if (!spec.subcommands?.length) {
    if (argsArray.some(arg => arg?.isVariadic)) return 1
    if (argsArray[0] && !argsArray[0].isOptional) return 2
  }
}
return spec.args && toArray(spec.args).some(arg => arg?.isDangerous) ? 3 : 2
```

对于 `pyright`：
- `spec.args` 存在。
- `argsArray` 为 `[{name:'files', isVariadic:true, isOptional:true}]`。
- 无 `isCommand`。
- `!spec.subcommands?.length` 为 true。
- `argsArray.some(arg => arg?.isVariadic)` 为 true → **返回 1**。

这意味着 `calculateDepth` 返回 1，前缀只有 `pyright`（深度 1 即只保留 `command` 一词，不加入任何参数）。这与 `buildPrefix` 的实际行为一致：前缀就是 `pyright`。

## 关键代码路径与文件引用

- **本文件**：`src/utils/bash/specs/pyright.ts`（91 行）
- **聚合入口**：`src/utils/bash/specs/index.ts`（第 4 行导入，第 11 行加入数组）
- **类型定义**：`src/utils/bash/registry.ts`（`CommandSpec`、`Option`、`Argument`）
- **规格查询**：`src/utils/bash/registry.ts`（`getCommandSpec`）
- **前缀构建**：`src/utils/bash/specPrefix.ts`（`buildPrefix` 第 88–137 行；`calculateDepth` 第 139–209 行）
- **Flag 参数判断**：`src/utils/bash/specPrefix.ts`（`flagTakesArg` 第 51–68 行）
- **参数停止判断**：`src/utils/bash/specPrefix.ts`（`shouldStopAtArg` 第 211–241 行）
- **底层解析**：`src/utils/bash/parser.ts`（`parseCommand`、`extractCommandArguments`）

## 依赖与外部交互

### 内部依赖

| 依赖文件 | 说明 |
|---------|------|
| `../registry.js` | `CommandSpec` 类型约束。 |
| `./index.ts`（反向） | 被聚合到本地 spec 数组。 |

### 外部交互

- **无运行时外部依赖**：纯静态元数据。
- **与 Pyright 版本演进的关系**：该 spec 基于某一版本的 Pyright CLI 编写。若 Pyright 新增选项（如 `--skipunannotated` 在较新版本中已存在），本地 spec 不会自动同步，可能导致新选项在前缀提取时被误识别为普通参数。但由于 `buildPrefix` 对未知 flag 的处理是“停止前缀扩展”，这实际上是一种 fail-safe 行为——前缀会保守地收敛为 `pyright`，不会导致安全问题。

## 风险、边界与改进建议

### 风险

1. **选项列表与上游不同步**：Pyright 作为活跃项目，CLI 选项可能随版本变化。例如 `--skipunannotated` 和 `--warnings` 已存在，但若未来新增 `--something-new`，本地 spec 不会感知。虽然前缀提取会保守收敛，但自动补全（若未来启用）将缺失新选项。
2. **`-` 选项的歧义**：spec 中有一条 `{ name: '-', description: 'Read file or directory list from stdin' }`。在 `flagTakesArg` 中，`-` 会被识别为已知 option，但它不带参数。若用户输入 `pyright - src/`，`buildPrefix` 会将 `-` 视为已知 flag 并继续，随后 `src/` 被识别为路径并停止。行为正确，但 `-` 作为 option name 在数组匹配逻辑中需要精确等于 `'-'`，不会与文件路径混淆。
3. **`--threads` 参数可选性**：`--threads` 的 args 标记为 `isOptional: true`（`args: { name: 'N', isOptional: true }`）。`flagTakesArg` 只看 `option.args` 是否存在（truthy），不看 `isOptional`，因此 `--threads` 始终被视为“带参数”。若用户输入 `pyright --threads src/`，`buildPrefix` 会将 `src/` 当作 `--threads` 的参数并跳过，随后无更多参数，前缀仍为 `pyright`。这符合预期，因为 `src/` 确实可能被误解为线程数，但系统保守处理是可接受的。

### 边界

- **无子命令**：Pyright 是一个扁平 CLI，没有子命令层级，因此 `spec.subcommands` 未定义。`calculateDepth` 和 `buildPrefix` 均按无子命令路径处理。
- **无 wrapper 语义**：`isCommand` 未标记，因此 `prefix.ts` 不会递归进入 `handleWrapper`。

### 改进建议

1. **增加版本注释**：在文件顶部添加注释，注明该 spec 基于的 Pyright 版本（如 `Based on pyright 1.1.xxx`），方便未来维护者判断是否需同步更新。
2. **补充 `isDangerous` 标记（可选）**：虽然 `pyright` 本身不危险，但某些选项（如 `--pythonpath` 指向自定义解释器）可能间接影响行为。不过当前安全体系不依赖 spec 的 `isDangerous` 进行命令拦截，而是依赖 `ast.ts` 的 argv 分析，因此此改进优先级较低。
3. **自动化同步机制**：考虑在 CI 中增加一个轻量检查脚本，定期对比 `pyright --help` 的输出与本地 spec 的选项列表，提醒维护者更新。由于 `pyright` 是外部工具，该脚本可作为可选的 linter 运行。
4. **整理选项顺序**：当前选项顺序大致按功能分组，但不够系统。可重新排列为：帮助/版本 → 输入控制（project、files、stdin） → 环境配置（pythonpath、venvpath、typeshedpath） → 输出控制（outputjson、verbose、stats） → 行为控制（watch、level、skipunannotated、threads） → 高级（createstub、verifytypes、dependencies）。提升可读性。
