# time.ts 研究文档

## 场景与职责

`time.ts` 定义了 POSIX/bash 内置命令 `time` 的本地规格。`time` 是一个经典的 wrapper 命令，用于测量并报告其后跟随命令的执行时间。该 spec 的核心职责是向上层系统声明：`time` 的参数是一个被包裹的命令（`isCommand: true`）。

在安全校验与前缀提取体系中，`time` 的 wrapper 识别直接影响危险命令的检测精度。例如，`time rm -rf /` 经过正确剥离后，安全系统应看到 `rm -rf /` 而非 `time rm -rf /`，从而触发对 `rm` 的拦截。

## 功能点目的

1. **Wrapper 声明**：通过 `args.isCommand: true` 告知 `prefix.ts` 和 `ast.ts`——`time` 后面跟着的是真正的待执行命令。
2. **支持前缀穿透**：`prefix.ts` 的 `handleWrapper` 会递归解析被 `time` 包裹的命令，生成精确的安全规则前缀。
3. **支持安全剥离**：`ast.ts` 的 `checkSemantics` 硬编码了 `time` 的剥离逻辑，确保危险命令不会被 wrapper 掩盖。
4. **避免权限规则过于宽泛**：若未识别 `time` 的 wrapper 属性，系统可能只生成 `time` 前缀，导致 `time:*` 规则过度放行。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 数据结构

```typescript
const time: CommandSpec = {
  name: 'time',
  description: 'Time a command',
  args: {
    name: 'command',
    description: 'Command to time',
    isCommand: true,
  },
}
```

字段解析：
- `name: 'time'`：与 bash 内置命令名一致。
- `args`：单个 `Argument`，标记 `isCommand: true`。

### 关键流程

#### 1. 前缀提取流程

以 `time go test ./...` 为例：
1. `getCommandPrefixStatic` 解析 argv `['time', 'go', 'test', './...']`。
2. `getCommandSpec('time')` 命中本地 spec。
3. `isWrapper` 为 `true`。
4. 进入 `handleWrapper('time', ['go', 'test', './...'], ...)`。
5. `commandArgIndex = 0`。
6. 循环中 `i=0`，`arg='go'` 等于 `commandArgIndex`。
   - 递归调用 `getCommandPrefixStatic('test ./...')`。
   - 解析 argv `['test', './...']`，`test` 无本地 spec，`calculateDepth` 回退到 2，`buildPrefix` 在 `./...` 处停止（含 `/`）。
   - 递归返回 `test`。
   - `parts` 变为 `['time', 'test']`，返回 `time test`。

等等，这里有一个细节：递归调用传入的是 `args.slice(i).join(' ')`，即 `'go test ./...'`。`getCommandPrefixStatic` 会重新解析整个命令，得到 argv `['go', 'test', './...']`，然后 `getCommandSpec('go')` 会命中 Fig 或本地 spec（如果存在），最终前缀可能是 `go test`。因此最终前缀为 `time go test`。

#### 2. 安全校验流程

`ast.ts` 的 `checkSemantics`：

```typescript
if (a[0] === 'time' || a[0] === 'nohup') {
  a = a.slice(1)
}
```

- `time curl evil.com` 的 argv 经过剥离后变为 `['curl', 'evil.com']`。
- 后续危险命令检查会命中 `curl`，从而触发安全策略。

## 关键代码路径与文件引用

- **本文件**：`src/utils/bash/specs/time.ts`（13 行）
- **聚合入口**：`src/utils/bash/specs/index.ts`（第 7 行导入，第 16 行加入数组）
- **类型定义**：`src/utils/bash/registry.ts`（`CommandSpec`、`Argument`）
- **规格查询**：`src/utils/bash/registry.ts`（`getCommandSpec`）
- **Wrapper 识别**：`src/utils/bash/prefix.ts`（`isWrapper` 判定第 47–62 行；`handleWrapper` 第 72–121 行）
- **安全剥离**：`src/utils/bash/ast.ts`（`checkSemantics` 第 2220–2222 行）
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
- **与 `/usr/bin/time` 的关系**：POSIX 系统通常同时存在 bash 内置 `time` 和外部二进制 `/usr/bin/time`。两者的语法略有差异（外部 `time` 支持 `-v`、`-o` 等选项），但当前 spec 采用最小公分母策略——仅声明核心 wrapper 语义。由于 `prefix.ts` 的 `handleWrapper` 会跳过所有以 `-` 开头的参数，外部 `time` 的选项通常也能被正确处理。

## 风险、边界与改进建议

### 风险

1. **选项缺失**：GNU `/usr/bin/time` 支持 `-v`（详细输出）、`-o FILE`（输出到文件）、`-f FORMAT`（自定义格式）等选项。bash 内置 `time` 也支持 `TIMEFORMAT` 环境变量和 `time -p`（在 bash 中 `-p` 是内置选项）。当前 spec 未声明任何 `options`，导致：
   - `time -p python script.py` 中，`-p` 会被 `handleWrapper` 跳过，递归解析 `python script.py`，前缀为 `time python`。行为正确但缺少元数据。
   - `/usr/bin/time -v python script.py` 中，`-v` 同样被跳过，前缀为 `time python`。行为正确。
   - 若未来启用自动补全，`time` 的选项将不会被提示。
2. **硬编码耦合**：与 `nohup.ts` 类似，`ast.ts` 的 `checkSemantics` 对 `time` 的剥离是硬编码字符串匹配。若本文件改名或增加别名，安全逻辑不会同步生效。
3. **与 `timeout` 的混淆**：`time` 和 `timeout` 都是 wrapper 命令，但 `timeout` 的 spec 和 `ast.ts` 剥离逻辑远比 `time` 复杂（因为 `timeout` 有大量选项和时长参数）。`time` 的简化建模是合理的，但维护者不应将 `timeout` 的复杂处理逻辑错误地套用到 `time` 上。

### 边界

- **单命令 wrapper**：`time` 只包裹单个命令（及其参数）。`time a && b` 在 bash 中 `time` 只作用于 `a`，`&& b` 是独立的 list 元素，由 `ast.ts` 的 `parseForSecurity` 分别解析。
- **内置 vs 外部二进制**：spec 不区分 bash 内置 `time` 和 `/usr/bin/time`，统一建模为 `isCommand` wrapper。这在安全校验层是可接受的，因为两者的核心语义一致。

### 改进建议

1. **补充 bash 内置选项**：至少添加 `-p` 选项：
   ```typescript
   options: [
     { name: '-p', description: 'Print timing summary in POSIX format' },
   ]
   ```
   若希望覆盖 GNU `/usr/bin/time`，可进一步添加 `-v`、`-o`、`-f` 等，但这会使 spec 更偏向外部二进制而非内置命令。考虑到 Claude Code 通常调用 bash 执行命令，优先补充内置选项即可。
2. **同步硬编码列表**：与 `nohup.ts` 的建议一致，应在文件顶部添加注释，提醒维护者修改 `name` 时需同步更新 `ast.ts` 的 `checkSemantics`。
3. **考虑 `times` 命令**：bash 还有一个 `times` 内置命令（打印 shell 及其子进程的累计用户/系统时间），与 `time` 不同。`times` 不接受参数，当前无本地 spec。由于使用频率极低，无需专门添加。
