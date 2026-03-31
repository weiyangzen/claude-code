# `src/commands/perf-issue/index.js` 深度研究文档

> 研究范围：代码、脚本、配置、测试及必要实现上下文  
> 执行器：kimi | 模型：k2p5  
> 生成时间：2026-04-01

---

## 1. 场景与职责

### 1.1 文件定位
`src/commands/perf-issue/index.js` 是 Claude Code CLI 命令体系中的一个**内部占位命令（Stub Command）**。该文件仅包含一行导出语句，导出一个最小化的对象：

```js
export default { isEnabled: () => false, isHidden: true, name: 'stub' };
```

### 1.2 职责归纳
1. **占位保留**：以最小代码占用保留 `/perf-issue` 命令的命名空间，防止其他模块抢占该命令标识。
2. **构建隔离**：作为 `INTERNAL_ONLY_COMMANDS` 数组成员，仅在 Anthropic 内部构建（`USER_TYPE === 'ant'`）中被注册到命令系统；外部用户构建中虽然包含该物理文件，但命令永远不会被激活。
3. **功能禁用**：通过 `isEnabled: () => false` 确保即使被加载到命令注册表，也会在 `getCommands()` 的过滤阶段被剔除；`isHidden: true` 确保不会出现在自动补全、帮助文档等用户界面中。

### 1.3 运行场景
- **非 ant 用户**：`INTERNAL_ONLY_COMMANDS` 不会被展开到 `COMMANDS()`，`perf-issue` 完全不可见。用户输入 `/perf-issue` 会走 `processSlashCommand.tsx` 的"Unknown skill"分支。
- **ant 内部用户**：`INTERNAL_ONLY_COMMANDS` 被展开，但 `isEnabled()` 恒为 `false`，`getCommands()` 过滤后仍不可见；用户输入 `/perf-issue` 同样返回"Unknown skill"。
- **演示模式（`IS_DEMO`）**：即使 `USER_TYPE === 'ant'`，`INTERNAL_ONLY_COMMANDS` 也不会被加入命令列表。

---

## 2. 功能点目的

### 2.1 设计意图推断
根据命令名称 `perf-issue`（Performance Issue）及其所在的 `INTERNAL_ONLY_COMMANDS` 集合，可推断其设计意图为：
- **性能问题报告通道**：未来可能用于 Anthropic 内部员工快速上报 Claude Code 的性能回归、延迟异常、内存泄漏等问题。
- **内部诊断数据收集**：可能集成 `heapdump`、`doctor`、`break-cache` 等现有诊断能力，一键打包性能数据并上传至内部系统。

### 2.2 当前状态
该命令目前处于**完全未实现状态**。与 `ant-trace`、`debug-tool-call`、`good-claude`、`ctx_viz`、`env`、`oauth-refresh`、`summary`、`teleport`、`issue`、`onboarding`、`share`、`backfill-sessions`、`autofix-pr`、`bughunter`、`mock-limits`、`bridge-kick`、`commit`、`commitPushPr`、`initVerifiers`、`resetLimits`、`resetLimitsNonInteractive` 等命令同属 `INTERNAL_ONLY_COMMANDS`，其中大部分为 stub 或内部专用命令。

---

## 3. 具体技术实现

### 3.1 数据结构
该文件导出的对象是一个**不完整的 `Command` 类型实例**。根据 `src/types/command.ts` 的定义，一个合法的 `Command` 必须是 `CommandBase & (PromptCommand | LocalCommand | LocalJSXCommand)`，即至少需要：
- `name: string`
- `description: string`
- `type: 'prompt' | 'local' | 'local-jsx'`
- 对应类型的执行属性（如 `load`、`getPromptForCommand`、`progressMessage`、`contentLength` 等）

而 `perf-issue` 的 stub 对象仅包含：

| 字段 | 值 | 说明 |
|------|-----|------|
| `name` | `'stub'` | 占位名称，非实际命令名 |
| `isEnabled` | `() => false` | 强制禁用 |
| `isHidden` | `true` | 强制隐藏 |

**缺失字段**：`type`、`description`、`load`、`getPromptForCommand`、`progressMessage`、`contentLength`、`source` 等。

### 3.2 关键流程

#### 3.2.1 命令注册与过滤流程
```
src/commands/perf-issue/index.js
    ↓ export default { isEnabled: () => false, isHidden: true, name: 'stub' }
    ↓ import perfIssue from './commands/perf-issue/index.js'  (src/commands.ts:148)
    ↓ INTERNAL_ONLY_COMMANDS = [..., perfIssue, ...]  (src/commands.ts:248)
    ↓ COMMANDS() 条件展开:
        ...(process.env.USER_TYPE === 'ant' && !process.env.IS_DEMO ? INTERNAL_ONLY_COMMANDS : [])
    ↓ getCommands(cwd)
        ↓ loadAllCommands(cwd) 加载所有命令源
        ↓ baseCommands = allCommands.filter(_ => meetsAvailabilityRequirement(_) && isCommandEnabled(_))
            ↓ isCommandEnabled(perfIssue) → perfIssue.isEnabled?.() ?? true → false
        ↓ perfIssue 被过滤掉，不会返回给调用方
```

#### 3.2.2 用户输入处理流程
```
用户输入 /perf-issue
    ↓ processSlashCommand.tsx:309 processSlashCommand()
    ↓ parseSlashCommand(inputString) 解析出 commandName = 'perf-issue'
    ↓ hasCommand(commandName, context.options.commands)
        ↓ context.options.commands 来自 getCommands() 的过滤后结果，不含 perfIssue
    ↓ hasCommand 返回 false
    ↓ 返回 "Unknown skill: perf-issue"
```

### 3.3 命令系统协议
- **注册协议**：所有内置命令必须在 `src/commands.ts` 中被静态导入，并显式加入 `COMMANDS()` 或 `INTERNAL_ONLY_COMMANDS` 数组。
- **过滤协议**：`getCommands()` 通过 `meetsAvailabilityRequirement()`（auth/provider 过滤）和 `isCommandEnabled()`（动态启用状态过滤）两道闸门决定命令是否对用户可见。
- **执行协议**：`processSlashCommand.tsx` 的 `getMessagesForSlashCommand()` 通过 `switch (command.type)` 分发到 `local-jsx` / `local` / `prompt` 三种执行路径。由于 `perf-issue` 缺少 `type` 字段，若绕过过滤直接执行，会进入 `switch` 的默认分支（无匹配 case），函数不会显式返回，可能导致调用方异常或挂起。

---

## 4. 关键代码路径与文件引用

### 4.1 目标文件
| 文件 | 行号 | 内容 |
|------|------|------|
| `src/commands/perf-issue/index.js` | 1 | `export default { isEnabled: () => false, isHidden: true, name: 'stub' };` |

### 4.2 调用方/注册方
| 文件 | 行号 | 作用 |
|------|------|------|
| `src/commands.ts` | 148 | `import perfIssue from './commands/perf-issue/index.js'` |
| `src/commands.ts` | 225-254 | `export const INTERNAL_ONLY_COMMANDS = [...]` 数组定义 |
| `src/commands.ts` | 248 | `perfIssue,` 加入 `INTERNAL_ONLY_COMMANDS` |
| `src/commands.ts` | 343-345 | 条件展开：`...(process.env.USER_TYPE === 'ant' && !process.env.IS_DEMO ? INTERNAL_ONLY_COMMANDS : [])` |
| `src/commands.ts` | 483-485 | `getCommands()` 中通过 `isCommandEnabled(_)` 过滤 |

### 4.3 命令执行层
| 文件 | 行号 | 作用 |
|------|------|------|
| `src/utils/processUserInput/processSlashCommand.tsx` | 309-381 | `processSlashCommand()` 解析并校验命令存在性 |
| `src/utils/processUserInput/processSlashCommand.tsx` | 525-776 | `getMessagesForSlashCommand()` 按 `command.type` 分发执行 |

### 4.4 UI 层
| 文件 | 行号 | 作用 |
|------|------|------|
| `src/hooks/useTypeahead.tsx` | 381 | 自动补全过滤 `!cmd.isHidden` |
| `src/components/HelpV2/HelpV2.tsx` | 5 | 导入 `INTERNAL_ONLY_COMMANDS` 符号（**未实际使用**） |

### 4.5 类型定义
| 文件 | 作用 |
|------|------|
| `src/types/command.ts` | `Command`、`CommandBase`、`PromptCommand`、`LocalCommand`、`LocalJSXCommand`、`isCommandEnabled()` 定义 |

---

## 5. 依赖与外部交互

### 5.1 模块依赖图
```
src/commands/perf-issue/index.js
    └── (零运行时依赖)
        ↑ 被 src/commands.ts 静态导入
            ↑ 被 src/main.tsx 调用 getCommands()
            ↑ 被 src/screens/REPL.tsx 使用 isCommandEnabled
            ↑ 被 src/utils/processUserInput/processSlashCommand.tsx 调用 getCommand/hasCommand
            ↑ 被 src/components/HelpV2/HelpV2.tsx 导入 INTERNAL_ONLY_COMMANDS（未使用）
```

### 5.2 外部交互
- **无直接外部交互**：该 stub 文件不调用任何 API、不读写文件、不依赖任何第三方库。
- **间接环境依赖**：
  - `process.env.USER_TYPE`：决定 `INTERNAL_ONLY_COMMANDS` 是否被注入全局命令列表。
  - `process.env.IS_DEMO`：演示模式下屏蔽所有内部命令。

### 5.3 同类 stub 命令列表
项目中存在大量采用相同模式的占位命令，均位于 `src/commands/<name>/index.js` 或同级：

| 命令 | 文件路径 | 状态 |
|------|----------|------|
| `ant-trace` | `src/commands/ant-trace/index.js` | stub |
| `debug-tool-call` | `src/commands/debug-tool-call/index.js` | stub |
| `good-claude` | `src/commands/good-claude/index.js` | stub |
| `ctx_viz` | `src/commands/ctx_viz/index.js` | stub |
| `env` | `src/commands/env/index.js` | stub |
| `oauth-refresh` | `src/commands/oauth-refresh/index.js` | stub |
| `summary` | `src/commands/summary/index.js` | stub |
| `teleport` | `src/commands/teleport/index.js` | stub |
| `issue` | `src/commands/issue/index.js` | stub |
| `onboarding` | `src/commands/onboarding/index.js` | stub |
| `share` | `src/commands/share/index.js` | stub |
| `backfill-sessions` | `src/commands/backfill-sessions/index.js` | stub |
| `autofix-pr` | `src/commands/autofix-pr/index.js` | stub |
| `bughunter` | `src/commands/bughunter/index.js` | stub |
| `mock-limits` | `src/commands/mock-limits/index.js` | stub |
| `perf-issue` | `src/commands/perf-issue/index.js` | stub |

---

## 6. 风险、边界与改进建议

### 6.1 类型安全风险 ⚠️
`perf-issue` 的 stub 对象**不是合法的 `Command` 联合类型成员**，却被强制放入 `INTERNAL_ONLY_COMMANDS: Command[]` 数组。之所以能编译通过，是因为：
1. `INTERNAL_ONLY_COMMANDS` 使用了 `.filter(Boolean)` 进行非空过滤，但**未做字段完整性校验**；
2. TypeScript 对 `.filter(Boolean)` 的类型收窄不够严格，允许结构不完整的对象混入数组。

**潜在后果**：若未来某处代码绕过 `getCommands()` 的过滤逻辑，直接通过 `INTERNAL_ONLY_COMMANDS` 索引访问并执行命令（例如 `command.load()` 或 `switch (command.type)`），运行时会因为缺少 `type` / `load` / `getPromptForCommand` 等字段而抛出 `TypeError` 或导致未定义行为。

### 6.2 死代码与技术债务
- **维护负担**：大量 stub 命令保持相同的占位实现，增加了模块导入开销和代码库噪音。
- **未使用导入耦合**：`src/components/HelpV2/HelpV2.tsx` 导入 `INTERNAL_ONLY_COMMANDS` 却未在组件中实际遍历使用（代码中 `antOnlyCommands` 被硬编码为空数组），增加了不必要的模块耦合，可能影响 tree-shaking 效果。

### 6.3 边界情况
| 场景 | 结果 |
|------|------|
| 非 ant 用户输入 `/perf-issue` | "Unknown skill: perf-issue" |
| ant 用户输入 `/perf-issue` | "Unknown skill: perf-issue"（被 `isEnabled` 过滤） |
| ant 用户查看帮助/自动补全 | 不可见（`isHidden: true`） |
| 动态修改 `process.env.USER_TYPE` 为 `'ant'` | 仍不可见（`isEnabled: () => false`） |
| 代码直接访问 `INTERNAL_ONLY_COMMANDS` 中的 `perfIssue` | 获得 `{ name: 'stub', isEnabled: () => false, isHidden: true }`，缺少 `type` |

### 6.4 改进建议

#### 选项 A：彻底移除（推荐，若短期内无开发计划）
1. 删除 `src/commands/perf-issue/` 目录；
2. 从 `src/commands.ts` 中移除 `import perfIssue from './commands/perf-issue/index.js'`（第 148 行）；
3. 从 `INTERNAL_ONLY_COMMANDS` 数组中移除 `perfIssue,`（第 248 行）；
4. 同步检查并清理 `HelpV2.tsx` 中未使用的 `INTERNAL_ONLY_COMMANDS` 导入。

#### 选项 B：提取统一 stub 工厂函数
若需要保留占位符，可在 `src/commands.ts` 或 `src/types/command.ts` 中定义一个类型安全的工厂函数：

```ts
export function createStubCommand(name: string): Command {
  return {
    type: 'local',
    name,
    description: `${name} (internal stub)`,
    isEnabled: () => false,
    isHidden: true,
    supportsNonInteractive: true,
    load: async () => ({
      call: async () => ({ type: 'text', value: 'Not implemented' }),
    }),
  } as Command;
}
```

这样可以：
- 保证所有 stub 都是合法的 `Command` 类型；
- 消除类型安全风险；
- 避免大量单文件目录的维护负担。

#### 选项 C：补全最小可用实现
若计划近期启用 `/perf-issue`，应至少补全：
- `type: 'prompt'` 或 `'local-jsx'`
- `description`
- 对应类型的执行属性（`getPromptForCommand` 或 `load`）
- 将 `name` 从 `'stub'` 改为 `'perf-issue'`
- 根据功能需求决定是否保留 `isEnabled` 门控

---

## 7. 附录：相关代码摘录

### 7.1 `src/commands/perf-issue/index.js`（完整内容）
```js
export default { isEnabled: () => false, isHidden: true, name: 'stub' };
```

### 7.2 `src/commands.ts` 中相关片段
```ts
// 第 148 行
import perfIssue from './commands/perf-issue/index.js'

// 第 225-254 行
export const INTERNAL_ONLY_COMMANDS = [
  backfillSessions,
  breakCache,
  bughunter,
  commit,
  commitPushPr,
  ctx_viz,
  goodClaude,
  issue,
  initVerifiers,
  ...(forceSnip ? [forceSnip] : []),
  mockLimits,
  bridgeKick,
  version,
  ...(ultraplan ? [ultraplan] : []),
  ...(subscribePr ? [subscribePr] : []),
  resetLimits,
  resetLimitsNonInteractive,
  onboarding,
  share,
  summary,
  teleport,
  antTrace,
  perfIssue,        // <-- 第 248 行
  env,
  oauthRefresh,
  debugToolCall,
  agentsPlatform,
  autofixPr,
].filter(Boolean)

// 第 343-345 行
...(process.env.USER_TYPE === 'ant' && !process.env.IS_DEMO
  ? INTERNAL_ONLY_COMMANDS
  : []),
```

### 7.3 `src/types/command.ts` 中 `isCommandEnabled`
```ts
export function isCommandEnabled(cmd: CommandBase): boolean {
  return cmd.isEnabled?.() ?? true
}
```

### 7.4 `processSlashCommand.tsx` 中命令不存在处理
```ts
if (!hasCommand(commandName, context.options.commands)) {
  // ...
  const unknownMessage = `Unknown skill: ${commandName}`;
  return {
    messages: [/* ... */],
    shouldQuery: false,
    resultText: unknownMessage
  };
}
```
