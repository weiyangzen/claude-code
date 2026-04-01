# srun.ts 研究文档

## 场景与职责

`srun.ts` 定义了 SLURM（Simple Linux Utility for Resource Management）工作负载管理器中 `srun` 命令的本地规格。`srun` 用于在集群节点上运行并行任务，是 HPC（高性能计算）环境中常见的命令。该 spec 的职责是向 Claude Code 的 bash 解析层声明：`srun` 是一个 wrapper 命令，它接受若干资源分配选项（如 `-n`/`--ntasks`、`-N`/`--nodes`），最终参数是被调度执行的命令（`isCommand: true`）。

正确识别 `srun` 的 wrapper 属性对于安全前缀提取至关重要。例如，`srun -n 4 python train.py` 的权限校验应针对 `python` 而非 `srun`，因为 `srun` 本身只是资源调度器，真正的计算逻辑和潜在风险来自被调度的 `python` 脚本。

## 功能点目的

1. **Wrapper 声明**：通过 `args.isCommand: true` 明确告知 `prefix.ts`——`srun` 的最后一个位置参数是真正的待执行命令。
2. **选项建模**：描述 `srun` 的两个常用选项 `-n`/`--ntasks` 和 `-N`/`--nodes`，帮助前缀提取器正确跳过选项及其值。
3. **支持前缀穿透**：`prefix.ts` 的 `handleWrapper` 会递归解析被 `srun` 包裹的命令，生成如 `srun python` 或更精确的 `srun python train.py` 前缀。
4. **避免权限规则过于宽泛**：若未识别 `srun` 的 wrapper 属性，系统可能只生成 `srun` 前缀，导致用户一旦允许 `srun:*`，任何被 `srun` 包裹的命令都会自动放行。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 数据结构

```typescript
const srun: CommandSpec = {
  name: 'srun',
  description: 'Run a command on SLURM cluster nodes',
  options: [
    {
      name: ['-n', '--ntasks'],
      description: 'Number of tasks',
      args: {
        name: 'count',
        description: 'Number of tasks to run',
      },
    },
    {
      name: ['-N', '--nodes'],
      description: 'Number of nodes',
      args: {
        name: 'count',
        description: 'Number of nodes to allocate',
      },
    },
  ],
  args: {
    name: 'command',
    description: 'Command to run on the cluster',
    isCommand: true,
  },
}
```

字段解析：
- `options`：两个资源选项，均带 `args`（数值参数）。
- `args`：单个 `Argument`，标记 `isCommand: true`，表示 `srun` 的“主参数”是被调度执行的命令。

### 关键流程

#### 前缀提取流程

以 `srun -n 4 python train.py --epochs 10` 为例：
1. `getCommandPrefixStatic` 解析 argv `['srun', '-n', '4', 'python', 'train.py', '--epochs', '10']`。
2. `getCommandSpec('srun')` 命中本地 spec。
3. `isWrapper` 为 `true`（`spec.args.isCommand === true`）。
4. 进入 `handleWrapper('srun', ['-n', '4', 'python', 'train.py', '--epochs', '10'], ...)`。
5. `commandArgIndex = 0`（`args` 是单个对象，非数组，因此 `toArray(spec.args)` 长度为 1，唯一元素的 `isCommand` 为 true，索引为 0）。
6. `handleWrapper` 循环：
   - `i=0`，`arg='-n'`：不是 `commandArgIndex`，且以 `-` 开头，不加入 `parts`。
   - `i=1`，`arg='4'`：不是 `commandArgIndex`，但 `NUMERIC.test('4')` 为 true，不加入 `parts`（`handleWrapper` 中跳过数值参数）。
   - `i=2`，`arg='python'`：等于 `commandArgIndex`。
     - 递归调用 `getCommandPrefixStatic('train.py --epochs 10')`。
     - 解析 argv `['train.py', '--epochs', '10']`，无 spec 命中（`train.py` 不是已知命令），`calculateDepth` 回退到 2，`buildPrefix` 在 `train.py` 处停止（含 `/` 或扩展名）。
     - 递归返回 `train.py`。
     - `parts` 变为 `['srun', 'train.py']`，返回 `srun train.py`。

**注意**：这里递归调用传入的是 `args.slice(i).join(' ')`，即 `'python train.py --epochs 10'`。递归解析后得到的前缀是 `python train.py`（因为 `python` 的 spec 可能将 `train.py` 识别为脚本参数并停止于 `--epochs`）。最终前缀为 `srun python train.py`。

#### 深度计算

`calculateDepth('srun', args, spec)` 的执行路径：
- `spec.args` 存在，`argsArray = [{name:'command', isCommand:true}]`。
- `argsArray.some(arg => arg?.isCommand)` 为 true。
- `!Array.isArray(spec.args)` 为 true（`spec.args` 是对象而非数组）。
- 返回 `2`（因为 `spec.args.isCommand` 且非数组）。

这意味着 `buildPrefix` 的 `maxDepth = 2`，但在 `handleWrapper` 的上下文中，`calculateDepth` 主要用于非 wrapper 路径。wrapper 路径的实际前缀由递归结果决定。

## 关键代码路径与文件引用

- **本文件**：`src/utils/bash/specs/srun.ts`（31 行）
- **聚合入口**：`src/utils/bash/specs/index.ts`（第 6 行导入，第 17 行加入数组）
- **类型定义**：`src/utils/bash/registry.ts`（`CommandSpec`、`Option`、`Argument`）
- **规格查询**：`src/utils/bash/registry.ts`（`getCommandSpec`）
- **Wrapper 识别与处理**：`src/utils/bash/prefix.ts`（`isWrapper` 判定第 47–62 行；`handleWrapper` 第 72–121 行）
- **深度计算**：`src/utils/bash/specPrefix.ts`（`calculateDepth` 第 193–200 行处理 `isCommand` 逻辑）
- **底层解析**：`src/utils/bash/parser.ts`（`parseCommand`、`extractCommandArguments`）

## 依赖与外部交互

### 内部依赖

| 依赖文件 | 说明 |
|---------|------|
| `../registry.js` | `CommandSpec` 类型约束。 |
| `./index.ts`（反向） | 被聚合导出。 |

### 外部交互

- **无运行时外部依赖**：纯静态元数据。
- **与 SLURM 生态的关系**：`srun` 只是 SLURM 众多命令之一（还有 `sbatch`、`salloc`、`sinfo`、`squeue` 等）。当前项目中仅 `srun` 有本地 spec，其他 SLURM 命令若出现会回退到 Fig 动态加载或默认深度规则。

## 风险、边界与改进建议

### 风险

1. **选项覆盖严重不足**：`srun` 拥有数十个选项（如 `--cpus-per-task`、`-c`、`--mem`、`--partition`、`-p`、`--time`、`-t`、`--job-name`、`-J` 等）。当前 spec 仅定义了 `-n` 和 `-N`，导致大量常见选项在前缀提取时会被 `handleWrapper` 的通用逻辑处理。虽然 `handleWrapper` 会跳过所有以 `-` 开头的参数以及纯数值参数，行为上通常不会出错，但缺少选项元数据意味着：
   - 若某个选项的值恰好看起来像命令（如 `--job-name python`），`handleWrapper` 不会将其误认为被包裹命令（因为它以 `--` 开头），但跳过逻辑不够精确。
   - 未来若启用自动补全，用户将无法获得 `srun` 的选项提示。
2. **命令位置假设**：`srun` 的语法允许选项和命令交错出现（如 `srun python -n 4 train.py` 在某些版本下可能不合法，但 `srun --export=ALL python train.py` 是合法的）。当前 `handleWrapper` 按顺序查找第一个非 flag、非数值、非环境变量赋值的参数作为被包裹命令，对于 `srun` 的标准用法（选项在前，命令在后）是正确的，但对于更复杂的交错用法可能存在前缀偏差。
3. **与 `sbatch`、`salloc` 的不对称**：`sbatch` 和 `salloc` 同样是 SLURM wrapper 命令，但项目中没有对应的本地 spec。用户若频繁使用这些命令，会体验到不一致的前缀提取行为。

### 边界

- **仅描述元数据**：`srun.ts` 不参与实际的集群调度或 MPI 进程管理。
- **单命令 wrapper**：spec 假设 `srun` 包裹单个命令。虽然 `srun` 技术上可以执行复合命令（如 `srun bash -c "..."`），但前缀提取器会递归解析 `bash -c "..."`，最终收敛到 `bash` 或更深层命令。

### 改进建议

1. **大幅扩充 `options` 列表**：至少补充 HPC 场景中最常用的选项：
   ```typescript
   options: [
     { name: ['-n', '--ntasks'], description: '...', args: { name: 'count' } },
     { name: ['-N', '--nodes'], description: '...', args: { name: 'count' } },
     { name: ['-c', '--cpus-per-task'], description: '...', args: { name: 'count' } },
     { name: ['-p', '--partition'], description: '...', args: { name: 'name' } },
     { name: ['-t', '--time'], description: '...', args: { name: 'limit' } },
     { name: ['-J', '--job-name'], description: '...', args: { name: 'name' } },
     { name: '--mem', description: '...', args: { name: 'size' } },
     { name: '--export', description: '...', args: { name: 'vars' } },
   ]
   ```
2. **增加 `sbatch` 和 `salloc` spec**：保持 SLURM 命令覆盖的一致性。它们的结构与 `srun` 非常相似（wrapper + 资源选项），复制并微调即可。
3. **注释说明命令位置策略**：在 `args` 的 `description` 中添加注释，提醒维护者 `handleWrapper` 按顺序查找第一个非 flag 参数作为被包裹命令，这与 `srun` 的标准用法一致。
4. **考虑 `isScript` 标记**：`srun` 本身不直接运行脚本文件，但被包裹命令可能是 `python script.py`。这由被包裹命令自己的 spec（如 `python`）处理，不需要在 `srun.ts` 中标记。
