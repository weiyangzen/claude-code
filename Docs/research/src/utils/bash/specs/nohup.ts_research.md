# nohup.ts 研究文档

## 场景与职责

`nohup.ts` 定义了 shell 命令 `nohup` 的本地规格（Command Spec）。`nohup` 是一个典型的“wrapper command”——它本身不执行特定功能，而是包裹并运行另一个命令，使其忽略 SIGHUP 信号。该 spec 的核心职责是向上层系统声明：`nohup` 的第一个（也是主要）参数是一个被包裹的命令（`isCommand: true`）。

在前缀提取与安全校验体系中，正确识别 `nohup` 的 wrapper 属性至关重要。例如，用户输入 `nohup python train.py &` 时，系统需要提取的前缀应为 `python train.py`（或更精确的 `python`），而不是 `nohup` 本身；权限校验也应针对 `python` 而非 `nohup`。

## 功能点目的

1. **Wrapper 声明**：通过 `args.isCommand: true` 明确告知 `prefix.ts` 和 `ast.ts`——`nohup` 后面跟着的是真正的待执行命令。
2. **支持前缀穿透**：`prefix.ts` 的 `handleWrapper` 逻辑会递归解析被 `nohup` 包裹的命令，从而生成更精确的安全规则前缀。
3. **支持安全剥离**：`ast.ts` 的 `checkSemantics` 函数硬编码了 `nohup` 的剥离逻辑（`if (a[0] === 'nohup') a = a.slice(1)`），使得 `nohup eval "..."` 这类危险组合能被正确识别并拦截。
4. **避免过度宽泛的权限规则**：如果没有该 spec，`nohup python ...` 可能只会生成 `nohup` 前缀，导致用户一旦允许 `nohup:*`，后续任何被 `nohup` 包裹的命令都会自动通过，形成权限放大漏洞。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 数据结构

```typescript
const nohup: CommandSpec = {
  name: 'nohup',
  description: 'Run a command immune to hangups',
  args: {
    name: 'command',
    description: 'Command to run with nohup',
    isCommand: true,
  },
}
```

字段解析：
- `name: 'nohup'`：与 bash 内置/外部命令名一致。
- `args`：单个 `Argument` 对象，标记 `isCommand: true`。这意味着该参数位置预期出现另一个命令名称及其参数。

### 关键流程

#### 1. 前缀提取流程（prefix.ts）

当用户输入 `nohup npm run build` 时：
1. `getCommandPrefixStatic` 解析命令，得到 argv `['nohup', 'npm', 'run', 'build']`。
2. 调用 `getCommandSpec('nohup')`，命中本地 spec。
3. `isWrapper` 判定为 `true`（因为 `spec.args.isCommand === true`）。
4. 进入 `handleWrapper('nohup', ['npm', 'run', 'build'], ...)`。
5. `handleWrapper` 找到 `commandArgIndex = 0`（`nohup` 后的第一个非 flag 参数即为被包裹命令）。
6. 递归调用 `getCommandPrefixStatic('npm run build')`，最终返回 `npm run build`（或根据 `npm` 的 spec 进一步收敛为 `npm run`）。
7. 最终前缀为 `nohup npm run build` → 实际返回的是递归结果拼接上 `nohup `前缀，即 `nohup npm run`（因为 `npm run` 是 `npm` 的 spec 决定的前缀）。

#### 2. 安全校验流程（ast.ts）

`checkSemantics` 函数在遍历 `SimpleCommand[]` 时执行以下逻辑：

```typescript
if (a[0] === 'time' || a[0] === 'nohup') {
  a = a.slice(1)
}
```

这意味着：
- `nohup rm -rf /` 的 argv 经过剥离后变为 `['rm', '-rf', '/']`。
- 后续的危险命令名单（如 `rm`、`eval`、`curl` 等）检查会命中 `rm`，从而触发相应的安全策略。

## 关键代码路径与文件引用

- **本文件**：`src/utils/bash/specs/nohup.ts`（13 行）
- **聚合入口**：`src/utils/bash/specs/index.ts`（第 3 行导入，第 15 行加入数组）
- **类型定义**：`src/utils/bash/registry.ts`（`CommandSpec`、`Argument`，第 4–21 行）
- **规格查询**：`src/utils/bash/registry.ts`（`getCommandSpec`，第 44–53 行）
- **Wrapper 识别**：`src/utils/bash/prefix.ts`（第 47–62 行，`isWrapper` 判定与 `handleWrapper`）
- **安全剥离**：`src/utils/bash/ast.ts`（`checkSemantics`，第 2220–2222 行）
- **前缀构建**：`src/utils/bash/specPrefix.ts`（`buildPrefix`、`calculateDepth`）

## 依赖与外部交互

### 内部依赖

| 依赖文件 | 说明 |
|---------|------|
| `../registry.js` | 引入 `CommandSpec` 类型。 |
| `./index.ts`（反向） | 被聚合导出。 |

### 外部交互

- **无运行时外部依赖**：纯静态对象。
- **与 `nice` 的对比**：`nice` 也是一个 wrapper 命令，但由于其选项（`-n`）会改变被包裹命令的位置，`prefix.ts` 将其硬编码在 `WRAPPER_COMMANDS` 集合中，而不是依赖 spec。`nohup` 的语法更规则（无复杂选项干扰命令位置），因此仅靠 `isCommand: true` 即可正确处理。

## 风险、边界与改进建议

### 风险

1. **选项缺失**：GNU `nohup` 支持 `--help`、`--version` 以及重定向文件参数（如 `nohup command > output.log 2>&1`）。当前 spec 未声明 `options`，也未处理 `nohup` 后的输出重定向参数。虽然 `prefix.ts` 的 `handleWrapper` 会跳过以 `-` 开头的参数，但如果未来出现类似 `nohup --foreground cmd` 的 GNU 扩展，递归解析可能会把 `--foreground` 误认为被包裹命令的一部分，导致前缀错误。
2. **硬编码耦合**：`ast.ts` 的 `checkSemantics` 中，`nohup` 的剥离逻辑是硬编码字符串匹配。如果本文件将 `name` 改为 `'nohup-gnu'` 或增加别名，安全剥离逻辑不会同步生效，产生规格与校验逻辑不一致的风险。
3. **深度限制**：`prefix.ts` 设置了 `wrapperCount > 2` 的递归上限。多层 wrapper 如 `nohup timeout 10s python script.py` 虽然通常能处理，但在极端嵌套（如 `nohup nice -n 10 timeout 5s eval "..."`）时可能因 `wrapperCount` 或 `recursionDepth` 超限而回退到保守策略。

### 边界

- **仅描述元数据**：`nohup.ts` 不执行任何信号处理或进程守护逻辑，真正的 `nohup` 行为由 BashTool 调用外部 shell 实现。
- **单参数 wrapper**：spec 只声明了一个 `isCommand` 参数。对于 `nohup` 而言这是足够的，因为它在标准实现中只接受一个命令（及其参数）。

### 改进建议

1. **补充 `--help` / `--version` 选项**：
   ```typescript
   options: [
     { name: '--help', description: 'Display help' },
     { name: '--version', description: 'Output version' },
   ]
   ```
   这样 `prefix.ts` 的 `handleWrapper` 在跳过 flag 时更有依据，也能为未来可能的自动补全提供元数据。
2. **同步 `ast.ts` 的硬编码列表**：在 `nohup.ts` 顶部添加注释，提示维护者若修改 `name` 字段，必须同步更新 `ast.ts` 中的 `checkSemantics` 和 `prefix.ts` 中的相关逻辑。或者考虑将 `time`/`nohup`/`timeout`/`nice` 的剥离列表抽取到一个共享常量模块中，由 `ast.ts` 和 specs 共同引用。
3. **考虑 `nohup` 的 BSD/macOS 差异**：BSD `nohup` 不接受任何选项，而 GNU `nohup` 接受 `--help`/`--version`。由于 Claude Code 可能在 macOS 和 Linux 上运行，spec 的设计应以“更宽松的 GNU 语义”为准，因为跳过未知 flag 比错误解析更安全。
