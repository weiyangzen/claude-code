# sleep.ts 研究文档

## 场景与职责

`sleep.ts` 定义了 POSIX/GNU `sleep` 命令的本地规格。`sleep` 是一个简单但高频使用的命令，用于在脚本中引入指定时长的延迟。该 spec 的职责是向 Claude Code 的 bash 解析子系统声明：`sleep` 接受一个必需的时长参数，且该参数不包裹其他命令、不是路径、不是子命令。

在安全与前缀体系中的作用相对简单：确保 `sleep 5` 这类命令能被正确识别为无害操作，避免生成过度宽泛的前缀（如 `sleep:*`），也不会被误判为未知命令而触发不必要的权限提示。

## 功能点目的

1. **命令识别与归类**：将 `sleep` 注册为已编目的内置/外部命令。
2. **参数语义标注**：声明 `sleep` 的 `duration` 参数是必需的（`isOptional: false`），帮助 `calculateDepth` 决定前缀深度。
3. **支持精确前缀规则**：`sleep 5` 应生成前缀 `sleep 5`（或 `sleep`），而不是 `sleep:*`。这取决于 `calculateDepth` 的返回值。
4. **避免安全误报**：由于 `sleep` 是只读/无副作用命令，准确的 spec 有助于系统将其快速归类为低风险操作。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 数据结构

```typescript
const sleep: CommandSpec = {
  name: 'sleep',
  description: 'Delay for a specified amount of time',
  args: {
    name: 'duration',
    description: 'Duration to sleep (seconds or with suffix like 5s, 2m, 1h)',
    isOptional: false,
  },
}
```

字段解析：
- `args.isOptional: false`：明确声明必须提供一个时长参数。
- 无 `options`、无 `subcommands`、无 `isCommand`：典型的单参数叶子命令。

### 关键流程

#### 前缀提取流程

以 `sleep 5m` 为例：
1. `getCommandPrefixStatic` 解析 argv `['sleep', '5m']`。
2. `getCommandSpec('sleep')` 命中本地 spec。
3. `isWrapper` 为 `false`。
4. `buildPrefix('sleep', ['5m'], spec)` 迭代：
   - `5m` 不以 `-` 开头。
   - `shouldStopAtArg('5m', [], spec)` 检查：无 `/`、无文件扩展名、无 URL 协议 → 返回 `false`。
   - `parts` 加入 `5m`。
5. 返回前缀 `sleep 5m`。

#### 深度计算

`calculateDepth('sleep', ['5m'], spec)` 的执行路径：
- `DEPTH_RULES` 无 `sleep` 条目。
- `spec.args` 存在，`argsArray = [{name:'duration', isOptional:false}]`。
- 无 `isCommand`。
- `!spec.subcommands?.length` 为 true。
- `argsArray.some(arg => arg?.isVariadic)` 为 false（`sleep` 的 args 未标记 `isVariadic`）。
- `argsArray[0] && !argsArray[0].isOptional` 为 **true** → **返回 2**。

因此 `maxDepth = 2`，`buildPrefix` 在 `parts.length >= 2` 时就会 break。等等，这与上面描述的 `buildPrefix` 行为矛盾了！让我重新仔细看 `buildPrefix` 的代码：

```typescript
for (let i = 0; i < args.length; i++) {
  const arg = args[i]
  if (!arg || parts.length >= maxDepth) break
  // ...
  if (await shouldStopAtArg(arg, args.slice(0, i), spec)) break
  // ...
  parts.push(arg)
}
```

如果 `maxDepth = 2`，初始 `parts = ['sleep']`（长度为 1）。
- 迭代 `i=0`，`arg='5m'`：`parts.length` 为 1，小于 2，不 break。
- `shouldStopAtArg('5m', [], spec)` 返回 false。
- `parts.push('5m')`，`parts` 变为 `['sleep', '5m']`，长度为 2。

下一次迭代 `i=1` 不存在，循环结束。最终前缀为 `sleep 5m`。

如果用户输入 `sleep 5m 10s`（虽然这在实际 `sleep` 中是无效的，但假设输入了）：
- `i=0` 后 `parts.length = 2`。
- `i=1`，`arg='10s'`：`parts.length >= maxDepth`（2 >= 2），break。
- 前缀仍为 `sleep 5m`。

这与 `calculateDepth` 返回 2 一致：前缀包含命令名 + 第一个参数。

## 关键代码路径与文件引用

- **本文件**：`src/utils/bash/specs/sleep.ts`（13 行）
- **聚合入口**：`src/utils/bash/specs/index.ts`（第 5 行导入，第 13 行加入数组）
- **类型定义**：`src/utils/bash/registry.ts`（`CommandSpec`、`Argument`）
- **规格查询**：`src/utils/bash/registry.ts`（`getCommandSpec`）
- **前缀构建**：`src/utils/bash/specPrefix.ts`（`buildPrefix`、`calculateDepth`）
- **参数停止判断**：`src/utils/bash/specPrefix.ts`（`shouldStopAtArg`）
- **底层解析**：`src/utils/bash/parser.ts`（`parseCommand`、`extractCommandArguments`）

## 依赖与外部交互

### 内部依赖

| 依赖文件 | 说明 |
|---------|------|
| `../registry.js` | `CommandSpec` 类型约束。 |
| `./index.ts`（反向） | 被聚合导出。 |

### 外部交互

- **无运行时外部依赖**：纯静态元数据。
- **与 Fig 库的关系**：`sleep` 作为基础 POSIX 命令，Fig 官方库中可能有 spec，但本地定义确保了在所有构建环境下都能命中，避免异步加载延迟。

## 风险、边界与改进建议

### 风险

1. **GNU `sleep` 支持多个参数**：GNU coreutils 的 `sleep` 允许 `sleep 5m 30s`（累加时长的语法）。当前 spec 的 `args` 未标记 `isVariadic: true`，因此 `calculateDepth` 返回 2，前缀在第一个参数后截断。这在安全上并无大碍（`sleep 5m` 和 `sleep 5m 30s` 都是无害的），但如果用户依赖前缀规则做批量允许（如 `Bash(sleep 5m:*)`），第二个参数的差异不会体现在规则中。
2. **浮点时长支持**：GNU `sleep` 支持 `sleep 0.5`（小数秒）。`shouldStopAtArg` 中 `5.5` 不含 `/`、扩展名或 URL，因此不会触发停止，前缀会包含 `sleep 5.5`。行为正确。
3. **与 `usleep`、`nanosleep` 的遗漏**：项目中只有 `sleep.ts`，没有 `usleep`（已废弃）或更现代的替代命令 spec。这些命令使用频率极低，不构成实际问题。

### 边界

- **单参数（非变长）语义**：当前 spec 将 `sleep` 建模为单参数命令。对于 `sleep 5m` 这种标准用法完全足够；对于 GNU 的多参数扩展，语义上略有偏差，但安全体系不依赖精确的参数数量。
- **无选项**：POSIX `sleep` 确实不接受任何选项，因此 `options` 数组的缺失是准确的。

### 改进建议

1. **标记 `isVariadic: true`（可选）**：若希望完全对齐 GNU `sleep` 的语法，可将 `args` 改为数组形式：
   ```typescript
   args: {
     name: 'duration',
     description: '...',
     isOptional: false,
     isVariadic: true,
   }
   ```
   这样 `calculateDepth` 会返回 1（因为 `isVariadic` 且 `!spec.subcommands?.length`），前缀收敛为 `sleep`。这反而比当前更宽泛。因此**不建议**修改，保持当前 `isOptional: false` 单参数建模更有利于生成 `sleep 5m` 这种较精确的前缀。
2. **增加注释说明 GNU 扩展**：在 `args` 的 `description` 或文件顶部添加注释，说明 GNU `sleep` 支持多个 duration 参数，但 spec 为前缀精确性起见建模为单参数。
3. **考虑与 `BASH_POLICY_SPEC` 对齐**：`src/utils/bash/commands.ts` 中的 `BASH_POLICY_SPEC` 明确举例 `sleep 3 => sleep`。注意这里 AI 提取的前缀示例是 `sleep`（不含参数），而 `specPrefix.ts` 的 `calculateDepth` 返回 2，意味着前缀应为 `sleep 5m`。这种差异是因为 `BASH_POLICY_SPEC` 是用于 LLM 前缀提取的“策略文档示例”，而 `specPrefix.ts` 是程序化前缀提取的“精确实现”。两者在 `sleep` 上的轻微差异（`sleep` vs `sleep 5m`）通常不会导致安全问题，因为 `sleep` 本身无害，但维护者应意识到策略文档与代码实现之间的不完全对应。
