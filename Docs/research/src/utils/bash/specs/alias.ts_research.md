# alias.ts 研究文档

## 场景与职责

`alias.ts` 是 Claude Code 本地 bash 命令规格（Command Spec）库中的一员，负责描述 shell 内置命令 `alias` 的元数据。该文件属于 `src/utils/bash/specs/` 目录下的静态规格定义，被 `registry.ts` 的 `getCommandSpec` 函数统一索引，供前缀提取、权限校验、自动补全等安全与交互子系统使用。

在项目整体架构中，bash 命令规格层承担“命令语义字典”的角色：它告诉上层系统某个命令有哪些子命令、选项、参数，以及参数是否代表被包裹的命令（`isCommand`）、是否可变长（`isVariadic`）、是否可选（`isOptional`）等。`alias` 作为典型的 shell 元命令（metacommand），其规格定义直接影响前缀提取器对 `alias ll='ls -la'` 这类语句的解析深度与权限判定。

## 功能点目的

1. **命令识别**：让系统知道 `alias` 是一个合法的、已编目的 bash 命令。
2. **参数语义标注**：声明 `alias` 接受零个或多个 `definition` 参数，每个参数形如 `name=value`，用于创建或列出别名。
3. **支持前缀提取**：通过 `isVariadic: true` 与 `isOptional: true` 告知 `specPrefix.ts` 的 `calculateDepth` 逻辑——`alias` 没有子命令树，参数也是可选且可变的，因此前缀深度应收敛为 `alias` 本身（`depth = 2` 时即 `alias` 一词）。
4. **支持安全校验**：虽然 `alias` 本身不标记 `isCommand`（它不会包裹另一个命令去执行），但准确的参数描述有助于 `ast.ts` 的 `parseForSecurity` 在提取 argv 时正确归类。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 数据结构

文件导出一个符合 `CommandSpec` 接口（定义于 `src/utils/bash/registry.ts`）的对象：

```typescript
const alias: CommandSpec = {
  name: 'alias',
  description: 'Create or list command aliases',
  args: {
    name: 'definition',
    description: 'Alias definition in the form name=value',
    isOptional: true,
    isVariadic: true,
  },
}
```

字段含义：
- `name`：命令关键字，必须与用户输入的 argv[0] 完全匹配（大小写敏感，但 bash 通常为小写）。
- `description`：人类可读的描述，目前主要服务于内部调试与潜在的未来 UI 展示。
- `args`：单个 `Argument` 对象（未使用数组形式，因为仅有一类参数）。
  - `isOptional: true`：允许 `alias` 裸执行（列出当前所有别名）。
  - `isVariadic: true`：允许一次定义多个别名，如 `alias ll='ls -la' la='ls -A'`。

### 关键流程

1. **注册流程**：`src/utils/bash/specs/index.ts` 导入 `alias` 并加入默认导出数组；`registry.ts` 的 `getCommandSpec` 通过 `specs.find(s => s.name === command)` 进行 O(n) 查找（结果被 LRU memoize）。
2. **前缀提取流程**：当用户输入 `alias grep='grep --color=auto'` 时，`prefix.ts` 调用 `getCommandPrefixStatic` → `parseCommand` 提取 argv → `getCommandSpec('alias')` 命中本地 spec → `specPrefix.ts` 的 `buildPrefix` 计算深度。由于 `alias` 无 `subcommands`、args 为 `isVariadic` 且非 `isCommand`，`calculateDepth` 返回 2，前缀为 `alias`。
3. **安全分析流程**：`ast.ts` 的 `parseForSecurity` 对 `alias` 语句进行 AST 解析时，将其视为普通 `simple_command`，不会触发 wrapper 剥离逻辑（因为 `alias` 不在 `time`/`nohup`/`timeout`/`nice` 的硬编码列表中）。

## 关键代码路径与文件引用

- **本文件**：`src/utils/bash/specs/alias.ts`（14 行）
- **聚合入口**：`src/utils/bash/specs/index.ts`（第 2 行导入，第 14 行导出到数组）
- **类型定义**：`src/utils/bash/registry.ts`（`CommandSpec`、`Argument` 类型，第 4–28 行）
- **规格查询**：`src/utils/bash/registry.ts`（`getCommandSpec`，第 44–53 行）
- **前缀构建**：`src/utils/bash/prefix.ts`（`getCommandPrefixStatic`，第 28–70 行）
- **深度计算**：`src/utils/bash/specPrefix.ts`（`calculateDepth`，第 139–209 行）
- **安全解析**：`src/utils/bash/ast.ts`（`parseForSecurity`、`checkSemantics`）
- **底层解析**：`src/utils/bash/parser.ts`（`parseCommand`、`extractCommandArguments`）

## 依赖与外部交互

### 内部依赖

| 依赖文件 | 说明 |
|---------|------|
| `../registry.js` | 引入 `CommandSpec` 类型，确保编译期类型安全。运行时不产生实际导入开销（TypeScript 类型擦除）。 |
| `./index.ts`（反向） | `alias.ts` 被 `index.ts` 静态导入，形成 specs 目录的聚合导出。 |

### 外部交互

- **无运行时外部依赖**：该文件是纯静态 JSON-like 的 TypeScript 对象，不调用网络、文件系统或第三方库。
- **与 `@withfig/autocomplete` 的关系**：`registry.ts` 在本地 spec 未命中时会尝试动态导入 `@withfig/autocomplete/build/${command}.js`。`alias` 作为 shell 内置命令，Fig 官方库中通常不存在对应 spec，因此本地定义是唯一的规格来源。

## 风险、边界与改进建议

### 风险

1. **参数语义不完整**：`alias` 实际支持 `-p` 选项（POSIX 模式打印），但当前 spec 未声明 `options` 数组。这会导致 `alias -p` 在前缀提取时被错误地截断——`buildPrefix` 遇到 `-p` 会将其视为普通 flag 并停止，前缀仍为 `alias`，结果虽然正确，但缺少选项元数据意味着未来若引入自动补全，`-p` 将不会被提示。
2. **等号解析歧义**：`name=value` 中的 `=` 在 bash 中属于 word 字符，当前 `args` 仅声明为字符串类型，没有进一步的结构化解析。`ast.ts` 会把 `alias ll='ls -la'` 的 argv 解析为 `['alias', "ll='ls", '-la'"]`（取决于引号处理方式），这在安全校验层可能产生看起来奇怪的参数，但通常不会导致误判，因为 `alias` 本身不是危险命令。
3. **与 `unalias` 不对称**：`unalias` 未在本地 specs 中定义，若用户频繁使用 `unalias`，系统会回退到 Fig 动态加载或无前缀深度规则，体验不一致。

### 边界

- **仅描述元数据，不执行命令**：`alias.ts` 不参与实际的 shell 执行，只是“描述层”。真正的 `alias` 生效与否由 BashTool 或外部 shell 决定。
- **前缀深度上限**：由于 `isVariadic` 为 true 且无 `isCommand`，`calculateDepth` 不会返回大于 2 的深度，因此 `alias` 永远不会产生 `alias foo=bar` 这类细粒度前缀规则。

### 改进建议

1. **补充 `options` 字段**：添加 `{ name: '-p', description: 'Print all defined aliases in re-usable form' }`，使 spec 更完整。
2. **考虑增加 `unalias` spec**：与 `alias` 成对出现，保持本地内置命令覆盖的完整性。
3. **文档化 `isOptional + isVariadic` 组合语义**：在 `registry.ts` 的 JSDoc 中明确说明“可选且可变长”参数对前缀深度的影响，方便后续维护者理解为什么 `alias` 的前缀只有一词。
