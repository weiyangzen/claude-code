# ParsedCommand.ts 深度研究文档

> 文件路径：`src/utils/bash/ParsedCommand.ts`  
> 研究时间：2026-04-01  
> 文件大小：~9.2 KB（318 行）

---

## 一、场景与职责

`ParsedCommand.ts` 是 Claude Code CLI 中 **Bash 命令结构化解析** 的核心模块。它的职责是将用户输入的原始 Bash 命令字符串转化为一个带有语义方法的解析对象，供下游的安全校验、权限检查、管道分段等逻辑使用。

该模块处于 **Bash 工具链的安全前哨位置**：
- 上游接收来自 `BashTool` 的原始命令输入；
- 下游服务于 `bashCommandHelpers.ts`（权限检查）和 `bashSecurity.ts`（安全校验）。

模块采用 **双路径架构**：
1. **Tree-sitter 主路径**（`TreeSitterParsedCommand`）：当 native/pure-TS tree-sitter 解析器可用时，基于 AST 进行精确的、引号感知的命令解析。
2. **Regex Fallback 路径**（`RegexParsedCommand_DEPRECATED`）：当 tree-sitter 不可用时，退化为基于 `shell-quote` + 正则的 legacy 解析。

---

## 二、功能点目的

### 2.1 IParsedCommand 接口

定义了所有解析实现必须暴露的契约：

| 方法 | 目的 |
|------|------|
| `getPipeSegments()` | 将命令按 `\|` 管道符切分为独立段，用于逐段权限检查。 |
| `withoutOutputRedirections()` | 移除输出重定向（`>` / `>>`），避免将文件名误判为待检查的命令。 |
| `getOutputRedirections()` | 提取输出重定向目标路径，供路径约束校验使用。 |
| `getTreeSitterAnalysis()` | 返回 AST 级别的安全分析数据（quoteContext、compoundStructure、dangerousPatterns 等），若 fallback 路径则返回 `null`。 |

### 2.2 TreeSitterParsedCommand（主实现）

基于 tree-sitter AST 节点构建，具备以下优势：
- **引号感知**：能正确区分 `"..."` 和 `'...'` 内的 `\|`，不会错误切分管道。
- **UTF-8 安全**：tree-sitter 的 `startIndex/endIndex` 是 UTF-8 字节偏移，而 JS `String.slice()` 使用 UTF-16 码元。实现中通过 `Buffer.from(command, 'utf8')` 进行字节级切片，避免多字节字符（如 `—` U+2014）导致的错位问题。
- **重定向精确移除**：利用 AST 中 `file_redirect` 节点的字节范围，从后往前拼接 `Buffer.concat`，避免索引漂移。

### 2.3 RegexParsedCommand_DEPRECATED（Fallback）

当 tree-sitter 模块加载失败或被 feature flag 关闭时启用：
- 依赖 `splitCommandWithOperators`（来自 `commands.ts`，基于 `shell-quote`）进行 token 切分。
- 依赖 `extractOutputRedirections`（同样来自 `commands.ts`）进行重定向提取。
- 被明确标记为 `@deprecated`，仅在外部构建或解析器异常时兜底。

### 2.4 解析入口与缓存

- `ParsedCommand.parse(command)`：统一异步入口。
- `getTreeSitterAvailable()`：使用 `lodash-es/memoize` 缓存 tree-sitter 可用性探测结果，避免每次重复 `import('./parser.js')` 和试解析。
- **Size-1 LRU 缓存**：`lastCmd` / `lastResult` 缓存最近一次解析结果，因为 legacy 调用方（如 `bashCommandIsSafeAsync`）可能在短时间内对同一命令重复调用 `parse()`。

---

## 三、具体技术实现

### 3.1 管道分段算法（Tree-sitter 路径）

```ts
function extractPipePositions(rootNode: Node): number[] {
  visitNodes(rootNode, node => {
    if (node.type === 'pipeline') {
      for (const child of node.children) {
        if (child.type === '|') {
          pipePositions.push(child.startIndex)
        }
      }
    }
  })
  return pipePositions.sort((a, b) => a - b)
}
```

关键点：
- 只收集 AST 中 `pipeline` 节点下的 `\|` 叶子节点位置；
- 由于 `visitNodes` 是深度优先，对于嵌套 pipeline（如 `a | b && c | d`），pipe 位置可能乱序，因此必须 `sort`；
- 切片时使用 `commandBytes.subarray(currentStart, pipePos).toString('utf8')` 保证字节级精确。

### 3.2 重定向提取算法

```ts
function extractRedirectionNodes(rootNode: Node): RedirectionNode[] {
  visitNodes(rootNode, node => {
    if (node.type === 'file_redirect') {
      const op = children.find(c => c.type === '>' || c.type === '>>')
      const target = children.find(c => c.type === 'word')
      // ...
    }
  })
}
```

`withoutOutputRedirections()` 的实现细节：
1. 将重定向节点按 `startIndex` 降序排列；
2. 逐个从 `commandBytes` 中切除 `[startIndex, endIndex)` 区间；
3. 最后 `trim().replace(/\s+/g, ' ')` 清理多余空白。

### 3.3 buildParsedCommandFromRoot

提供给已经持有 AST root 的调用方（如 `bashCommandHelpers.ts` 中的 `checkCommandOperatorPermissions`）直接构造 `TreeSitterParsedCommand`，避免二次调用 `parseCommand()` 造成冗余的 native parse。

---

## 四、关键代码路径与文件引用

### 4.1 内部依赖图

```
ParsedCommand.ts
├── ./commands.js           ← splitCommandWithOperators, extractOutputRedirections
├── ./parser.js             ← parseCommand, Node 类型, PARSE_ABORTED
├── ./treeSitterAnalysis.js ← analyzeCommand, TreeSitterAnalysis
└── lodash-es/memoize.js    ← getTreeSitterAvailable 缓存
```

### 4.2 上游调用方

| 调用方文件 | 调用方式 | 用途 |
|-----------|---------|------|
| `src/tools/BashTool/bashCommandHelpers.ts` | `ParsedCommand.parse()` / `buildParsedCommandFromRoot()` | 检查管道/复合命令权限 |
| `src/tools/BashTool/bashSecurity.ts` | `ParsedCommand.parse()` | 安全校验中的命令解析 |

### 4.3 核心类型与符号

- `OutputRedirection`：`{ target: string, operator: '>' | '>>' }`
- `IParsedCommand`：公共接口
- `TreeSitterParsedCommand` / `RegexParsedCommand_DEPRECATED`：两个实现类
- `PARSE_ABORTED`（来自 `parser.ts`）：安全哨兵，表示解析器已加载但主动中止（超时/节点超限），此时 **禁止** 回退到 regex 路径。

---

## 五、依赖与外部交互

### 5.1 parser.ts（tree-sitter 入口）

`src/utils/bash/parser.ts` 提供：
- `parseCommand(command)`：返回 `{ rootNode, envVars, commandNode, originalCommand }`；
- `parseCommandRaw(command)`：仅返回 `rootNode`，供安全路径减少一次 tree walk；
- `PARSE_ABORTED` Symbol：用于区分"模块未加载"和"解析主动失败"。

`parser.ts` 内部通过 `feature('TREE_SITTER_BASH')` 控制是否启用 native NAPI 解析器，并调用 `bashParser.ts` 中的纯 TS 解析器作为 shadow/备用。

### 5.2 treeSitterAnalysis.ts（AST 安全分析）

`analyzeCommand(rootNode, command)` 提取四类安全数据：
1. `quoteContext`：单引号/双引号/heredoc 的精确范围；
2. `compoundStructure`：复合操作符（`&&`、`||`、`;`）、管道、子 shell、命令组；
3. `hasActualOperatorNodes`：消除 `find -exec \;` 的误报；
4. `dangerousPatterns`：命令替换、进程替换、参数扩展、heredoc、注释。

### 5.3 commands.ts（shell-quote 包装）

Fallback 路径依赖的 `splitCommandWithOperators` 和 `extractOutputRedirections` 均在此文件中。该文件基于 `shell-quote` 库，包含大量安全加固（placeholder 随机盐、heredoc 预提取、行继续符处理、静态重定向目标校验等）。

---

## 六、风险、边界与改进建议

### 6.1 已知风险

| 风险点 | 说明 |
|--------|------|
| **PARSE_ABORTED 回退风险** | 若调用方在收到 `PARSE_ABORTED` 后仍回退到 regex 路径，会丧失 `EVAL_LIKE_BUILTINS` 等安全检测，导致 `trap`、`enable`、`hash` 等命令泄露。`ParsedCommand.ts` 的 `doParse` 在 tree-sitter 异常时确实会 fallback 到 regex，但上游 `bashCommandHelpers.ts` 已用 `astRoot !== PARSE_ABORTED` 进行保护。 |
| **UTF-8 字节偏移 vs JS 索引** | TreeSitterParsedCommand 已用 `Buffer` 正确处理；但 RegexParsedCommand 完全基于 JS 字符串索引，若 fallback 路径遇到多字节字符，切分位置可能不准确。 |
| **Size-1 缓存的并发安全** | `lastCmd` / `lastResult` 是模块级变量，无锁。虽然 Node.js 单线程事件循环下通常安全，但快速交替调用不同命令可能导致缓存击穿（影响极小，仅性能损失）。 |
| **无测试覆盖** | 项目内未找到针对 `ParsedCommand.ts` 的单元测试文件，修改时依赖集成测试和人工审计。 |

### 6.2 边界情况

- **空命令**：`doParse('')` 直接返回 `null`。
- **无管道命令**：`getPipeSegments()` 返回 `[originalCommand]`，保证下游始终拿到数组。
- **无重定向命令**：`withoutOutputRedirections()` 和 `getOutputRedirections()` 均直接返回原命令或空数组，避免不必要的字符串操作。
- **嵌套 pipeline 的 pipe 位置排序**：已显式处理，但注释指出深度优先遍历会导致乱序，依赖 `sort` 修正。

### 6.3 改进建议

1. **移除或隔离 Deprecated Fallback**
   - 随着 `bashParser.ts` 中纯 TS tree-sitter 解析器的成熟（无需 WASM 初始化，始终可用），`RegexParsedCommand_DEPRECATED` 的存活价值进一步降低。建议在未来大版本中彻底移除，减少双路径维护成本。

2. **增加单元测试**
   - 针对 `getPipeSegments` 增加覆盖：含引号的管道、嵌套 pipeline、多字节字符、无管道命令。
   - 针对 `withoutOutputRedirections` 增加覆盖：多重重定向、重定向与管道混合、多字节字符位置。

3. **缓存策略优化**
   - 当前 size-1 缓存对并发交替调用不友好。可考虑使用 `Map` 或 `lru-cache` 维护一个容量为 10~20 的轻量缓存，以覆盖权限检查链中对同一命令的多次访问。

4. **暴露 `parseCommandRaw` 的 shortcut**
   - 对于只需要 `rootNode` 的调用方（如安全校验），`ParsedCommand.parse` 仍会触发 `findCommandNode` 和 `extractEnvVars`。可考虑在 `ParsedCommand` 上暴露一个 `parseRaw` 入口，进一步减少不必要的 tree walk。

---

*文档结束*
