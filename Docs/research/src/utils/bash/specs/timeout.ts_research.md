# timeout.ts 研究文档

## 场景与职责

`timeout.ts` 定义了 GNU `timeout` 命令的本地规格。`timeout` 是一个 wrapper 命令，用于在指定时间限制内运行另一个命令，超时后发送信号终止该命令。该 spec 的核心职责是声明 `timeout` 的两个位置参数：第一个是时长（`duration`），第二个是被包裹的命令（`command`，标记 `isCommand: true`）。

在安全校验与前缀提取体系中，`timeout` 的识别尤为重要。因为 `timeout` 后面跟着的才是真正的待执行命令，若系统只提取到 `timeout` 前缀，则用户一旦允许 `timeout:*`，任何被 `timeout` 包裹的危险命令（如 `timeout 5s rm -rf /`）都会绕过权限检查。

## 功能点目的

1. **Wrapper 与参数结构声明**：通过 `args` 数组明确区分 `duration` 参数和 `isCommand: true` 的 `command` 参数。
2. **支持前缀穿透**：`prefix.ts` 的 `handleWrapper` 会跳过 `duration`，递归解析被 `timeout` 包裹的命令，生成精确前缀。
3. **支持安全剥离**：`ast.ts` 的 `checkSemantics` 包含对 `timeout` 的复杂剥离逻辑，该逻辑与本 spec 的结构相呼应。
4. **避免权限规则过于宽泛**：确保 `timeout 10s curl evil.com` 的权限规则针对 `curl` 而非 `timeout`。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 数据结构

```typescript
const timeout: CommandSpec = {
  name: 'timeout',
  description: 'Run a command with a time limit',
  args: [
    {
      name: 'duration',
      description: 'Duration to wait before timing out (e.g., 10, 5s, 2m)',
      isOptional: false,
    },
    {
      name: 'command',
      description: 'Command to run',
      isCommand: true,
    },
  ],
}
```

字段解析：
- `args` 使用数组形式，精确描述两个位置参数的先后顺序：
  1. `duration`：必需参数，表示超时时间（如 `10`、`5s`、`2m`、`1h`）。
  2. `command`：标记 `isCommand: true`，表示被超时控制的实际命令。

### 关键流程

#### 1. 前缀提取流程

以 `timeout 30s python train.py --batch 32` 为例：
1. `getCommandPrefixStatic` 解析 argv `['timeout', '30s', 'python', 'train.py', '--batch', '32']`。
2. `getCommandSpec('timeout')` 命中本地 spec。
3. `isWrapper` 为 `true`（`spec.args` 数组中存在 `isCommand: true` 的元素）。
4. 进入 `handleWrapper('timeout', ['30s', 'python', 'train.py', '--batch', '32'], ...)`。
5. `commandArgIndex = 1`（`toArray(spec.args)` 中第二个元素的 `isCommand` 为 true）。
6. `handleWrapper` 循环：
   - `i=0`，`arg='30s'`：不是 `commandArgIndex`，且不以 `-` 开头、非数值（`NUMERIC.test('30s')` 为 false）、非环境变量赋值。根据 `handleWrapper` 的通用逻辑，非 `-`、非数值、非 env 的参数会被加入 `parts`。
   - 等等，让我重新看 `handleWrapper` 的代码：

```typescript
for (let i = 0; i < args.length && i <= commandArgIndex; i++) {
  if (i === commandArgIndex) {
    const result = await getCommandPrefixStatic(
      args.slice(i).join(' '),
      recursionDepth + 1,
      wrapperCount + 1,
    )
    if (result?.commandPrefix) {
      parts.push(...result.commandPrefix.split(' '))
      return parts.join(' ')
    }
    break
  } else if (
    args[i] &&
    !args[i]!.startsWith('-') &&
    !ENV_VAR.test(args[i]!)
  ) {
    parts.push(args[i]!)
  }
}
```

对于 `timeout`：
- `i=0`，`arg='30s'`：`!args[0].startsWith('-')` 为 true，`!ENV_VAR.test('30s')` 为 true → `parts.push('30s')`。
- `i=1`，`arg='python'`：`i === commandArgIndex`。
  - 递归调用 `getCommandPrefixStatic('python train.py --batch 32')`。
  - 返回 `python train.py`（假设 `python` 的 spec 将 `train.py` 识别为脚本参数并在 `--batch` 前停止）。
  - `parts.push(...'python train.py'.split(' '))`，`parts` 变为 `['timeout', '30s', 'python', 'train.py']`。
  - 返回 `timeout 30s python train.py`。

#### 2. 深度计算

`calculateDepth('timeout', args, spec)`：
- `spec.args` 是数组，且存在 `isCommand` 元素。
- `!Array.isArray(spec.args)` 为 false（因为是数组）。
- 返回 `Math.min(2 + argsArray.findIndex(arg => arg?.isCommand), 3)`。
- `argsArray.findIndex(arg => arg?.isCommand)` 为 1。
- `Math.min(2 + 1, 3)` = **3**。

这意味着 `buildPrefix` 的 `maxDepth = 3`。但在 wrapper 路径中，`calculateDepth` 的结果对 `handleWrapper` 的行为影响有限，因为 `handleWrapper` 有自己的递归逻辑。

#### 3. 安全校验流程

`ast.ts` 的 `checkSemantics` 对 `timeout` 有最复杂的硬编码剥离逻辑（约 80 行代码，第 2223–2296 行）：

```typescript
} else if (a[0] === 'timeout') {
  let i = 1
  while (i < a.length) {
    const arg = a[i]!
    if (
      arg === '--foreground' ||
      arg === '--preserve-status' ||
      arg === '--verbose'
    ) {
      i++ // known no-value long flags
    } else if (/^--(?:kill-after|signal)=[A-Za-z0-9_.+-]+$/.test(arg)) {
      i++
    } else if (
      (arg === '--kill-after' || arg === '--signal') &&
      a[i + 1] &&
      /^[A-Za-z0-9_.+-]+$/.test(a[i + 1]!)
    ) {
      i += 2
    } else if (arg.startsWith('--')) {
      // Unknown long flag — fail closed
      return { ok: false, reason: `timeout with ${arg} flag cannot be statically analyzed` }
    } else if (arg === '-v') {
      i++
    } else if (
      (arg === '-k' || arg === '-s') &&
      a[i + 1] &&
      /^[A-Za-z0-9_.+-]+$/.test(a[i + 1]!)
    ) {
      i += 2
    } else if (/^-[ks][A-Za-z0-9_.+-]+$/.test(arg)) {
      i++
    } else if (arg.startsWith('-')) {
      // Unknown flag — fail closed
      return { ok: false, reason: `timeout with ${arg} flag cannot be statically analyzed` }
    } else {
      break // non-flag — should be the duration
    }
  }
  if (a[i] && /^\d+(?:\.\d+)?[smhd]?$/.test(a[i]!)) {
    a = a.slice(i + 1)
  } else if (a[i]) {
    return { ok: false, reason: `timeout duration '${a[i]}' cannot be statically analyzed` }
  } else {
    break // no more args
  }
}
```

这段逻辑与本 spec 的结构相呼应：
- 先跳过 `timeout` 的已知选项（`-k`、`-s`、`-v`、`--foreground` 等）。
- 然后验证时长参数是否符合 `^\d+(?:\.\d+)?[smhd]?$`。
- 最后 `a = a.slice(i + 1)`，将 `timeout` 和 `duration` 从 argv 中剥离，留下被包裹的命令。

## 关键代码路径与文件引用

- **本文件**：`src/utils/bash/specs/timeout.ts`（20 行）
- **聚合入口**：`src/utils/bash/specs/index.ts`（第 8 行导入，第 12 行加入数组）
- **类型定义**：`src/utils/bash/registry.ts`（`CommandSpec`、`Argument`）
- **规格查询**：`src/utils/bash/registry.ts`（`getCommandSpec`）
- **Wrapper 识别与处理**：`src/utils/bash/prefix.ts`（`isWrapper` 判定第 47–62 行；`handleWrapper` 第 72–121 行）
- **安全剥离**：`src/utils/bash/ast.ts`（`checkSemantics` 第 2223–2296 行）
- **深度计算**：`src/utils/bash/specPrefix.ts`（`calculateDepth` 第 193–200 行）
- **底层解析**：`src/utils/bash/parser.ts`（`parseCommand`、`extractCommandArguments`）

## 依赖与外部交互

### 内部依赖

| 依赖文件 | 说明 |
|---------|------|
| `../registry.js` | `CommandSpec` 类型约束。 |
| `./index.ts`（反向） | 被聚合导出。 |

### 外部交互

- **无运行时外部依赖**：纯静态元数据。
- **与 GNU coreutils 的关系**：该 spec 基于 GNU `timeout` 的语法。BSD/macOS 的 `timeout`（若存在）选项集可能不同，但 GNU 语义是主流服务器环境（Linux）的标准。

## 风险、边界与改进建议

### 风险

1. **选项缺失**：当前 spec 未声明任何 `options`，但 `ast.ts` 的 `checkSemantics` 却硬编码了大量 `timeout` 选项（`-k`、`-s`、`-v`、`--foreground`、`--preserve-status`、`--verbose`、`--kill-after`、`--signal`）。这种“spec 简单、校验复杂”的不对称意味着：
   - `prefix.ts` 的 `handleWrapper` 对 `timeout` 的处理是通用逻辑（跳过 `-` 开头的参数），与 `ast.ts` 的精细剥离逻辑不一致。虽然通常不会出错，但理论上存在前缀提取和安全校验看到不同 argv 的风险。
   - 若 `handleWrapper` 遇到 `--signal=KILL`，由于 spec 中无该 option，`flagTakesArg` 不会识别它，`handleWrapper` 仅因其以 `--` 开头而跳过。行为正确但缺乏精确语义。
2. **时长参数正则限制**：`ast.ts` 的时长正则 `^\d+(?:\.\d+)?[smhd]?$` 不支持 GNU `timeout` 接受的 `.5`、`+5`、`5e-1`、`inf`、`infinity` 等格式。虽然 `ast.ts` 已对此做了 fail-closed 处理（返回 cannot be statically analyzed），但 `timeout.ts` 的 spec 中并未体现这种限制。
3. **硬编码耦合**：`ast.ts` 的 `checkSemantics` 对 `timeout` 的剥离逻辑长达 80 行，且与本 spec 的 `name: 'timeout'` 硬耦合。若本文件改名或增加别名，安全逻辑不会同步生效。

### 边界

- **仅描述核心结构**：spec 只声明了 `duration` + `command` 的位置参数结构，未覆盖 GNU `timeout` 的完整选项集。
- **单命令 wrapper**：`timeout` 只包裹单个命令。`timeout 5s a && b` 中 `timeout` 只作用于 `a`。

### 改进建议

1. **在 spec 中补充 `options` 数组**：将 `ast.ts` 中硬编码的已知选项同步到 `timeout.ts` 中：
   ```typescript
   options: [
     { name: '--foreground', description: '...' },
     { name: '--preserve-status', description: '...' },
     { name: '--verbose', description: '...' },
     { name: ['-k', '--kill-after'], description: '...', args: { name: 'DURATION' } },
     { name: ['-s', '--signal'], description: '...', args: { name: 'SIGNAL' } },
     { name: '-v', description: '...' },
   ]
   ```
   这样 `prefix.ts` 的 `flagTakesArg` 和 `handleWrapper` 能更精确地跳过选项及其值，与 `ast.ts` 的剥离逻辑保持一致。
2. **注释说明时长限制**：在 `duration` 参数的 `description` 中添加注释，说明当前安全校验仅支持 `\d+(?:\.\d+)?[smhd]?` 格式的时长，GNU 扩展格式（如 `.5`、`inf`）会被安全系统拒绝。
3. **抽取共享常量模块**：将 `ast.ts` 中 `timeout` 的已知选项列表和时长正则抽取到一个共享模块（如 `src/utils/bash/wrappers.ts`），由 `ast.ts` 和 `timeout.ts` 共同引用，消除硬编码耦合。
4. **考虑 `timeout` 的 BSD 差异**：BSD `timeout`（如 macOS 上的 `gtimeout`）选项集与 GNU 不同。由于项目主要面向 Linux 服务器环境，当前以 GNU 为准是合理的，但可在注释中注明。
