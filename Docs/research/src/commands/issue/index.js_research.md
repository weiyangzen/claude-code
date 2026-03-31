# 研究文档: `src/commands/issue/index.js`

> **执行器**: kimi  
> **模型**: k2p5  
> **研究日期**: 2026-04-01  
> **文件路径**: `/home/sansha/Github/claude-code-instructkr/src/commands/issue/index.js`

---

## 1. 场景与职责

### 1.1 功能定位

`src/commands/issue/index.js` 是 Claude Code CLI 中 `/issue` 命令的**占位符（Stub）实现**。当前该命令被**完全禁用**，不再向用户暴露。

**当前状态**：
```javascript
export default { isEnabled: () => false, isHidden: true, name: 'stub' };
```

### 1.2 历史职责

从历史代码痕迹来看，`/issue` 曾是一个 **ANT-ONLY 的内部调试/反馈命令**，用于让 Anthropic 内部员工（`USER_TYPE === 'ant'`）在发现模型行为异常时，快速提交包含会话上下文、API 请求记录和 Git 状态的详细问题报告。

其原始职责包括：
- **模型问题收集**：当用户遇到幻觉、错误工具选择、拒绝回答等模型层问题时，提供结构化的反馈通道
- **上下文自动附加**：自动抓取最近几次 API 请求（prompt dump）、当前会话 transcript、Git 仓库状态
- **GitHub Issue 草稿生成**：将收集到的信息编码为 GitHub issue URL，一键在浏览器中打开草稿页

### 1.3 当前状态说明

| 属性 | 值 | 说明 |
|------|-----|------|
| `isEnabled` | `() => false` | 命令在任何环境下都默认禁用 |
| `isHidden` | `true` | 命令在帮助文档和自动补全中隐藏 |
| `name` | `'stub'` | 桩名称，满足 `CommandBase` 的最小类型约束 |

---

## 2. 功能点目的

### 2.1 当前目的

该文件当前仅作为**功能占位符**存在，目的包括：

1. **构建时隔离**：通过 `USER_TYPE` 环境变量在编译时隔离内部功能
2. **保留重新启用可能性**：代码库中保留了大量相关支撑逻辑，未来可通过替换 stub 重新启用
3. **类型系统兼容**：导出的对象满足 `CommandBase` 的最小约束，确保命令注册系统不会崩溃

### 2.2 与 `/feedback` 命令的区别

| 特性 | `/issue` (已禁用) | `/feedback` |
|------|-------------------|-------------|
| **目标用户** | Ant 内部员工 | 普通用户 |
| **数据收集** | 完整会话记录、API prompt dump、Git 状态 | 用户描述 + 基础环境信息 |
| **GitHub 集成** | 内部仓库 | 公开仓库 |
| **自动触发** | 支持（摩擦检测、反馈调查） | 不支持 |
| **当前状态** | 完全禁用 | 正常使用 |

---

## 3. 具体技术实现

### 3.1 Stub 命令的数据结构

`src/commands/issue/index.js` 导出的对象属于 `CommandBase` 的最小子集（定义于 `src/types/command.ts`）：

```typescript
// src/types/command.ts (line 175-203)
export type CommandBase = {
  availability?: CommandAvailability[]
  description: string        // 缺失，但 stub 不进入实际使用路径
  isEnabled?: () => boolean  // () => false
  isHidden?: boolean         // true
  name: string               // 'stub'
  aliases?: string[]
  // ... 其他可选字段
}
```

由于 `isEnabled` 返回 `false`，在 `src/commands.ts` 的 `getCommands()` 中会被 `isCommandEnabled(_)` 过滤掉：

```typescript
// src/commands.ts (line 214-216)
export function isCommandEnabled(cmd: CommandBase): boolean {
  return cmd.isEnabled?.() ?? true
}
```

### 3.2 命令注册与分发流程

在 `src/commands.ts` 中：

**1. 导入**（第 7 行）：
```typescript
import issue from './commands/issue/index.js'
```

**2. 内部命令列表**（第 225-254 行）：
```typescript
export const INTERNAL_ONLY_COMMANDS = [
  // ...
  issue,              // <-- 此处注册
  // ...
].filter(Boolean)
```

**3. 条件加载**（第 343-345 行）：
```typescript
...(process.env.USER_TYPE === 'ant' && !process.env.IS_DEMO
  ? INTERNAL_ONLY_COMMANDS
  : []),
```

**关键逻辑**：
- 只有当 `USER_TYPE === 'ant'` 且 `IS_DEMO` 未设置时，内部命令才会被加载到 `COMMANDS`
- 但即使被加载，`issue.isEnabled()` 恒为 `false`，会在 `getCommands()` 中被过滤
- 对外版本（`USER_TYPE === 'external'`）不会加载这些命令

### 3.3 残留支撑代码

虽然 `/issue` 命令本身已死，但以下功能仍明确引用或为其而设：

| 功能 | 文件 | 说明 |
|------|------|------|
| API 请求缓存 / Prompt Dump | `src/services/api/dumpPrompts.ts` | 为 `ant` 用户缓存最近 5 个 API 请求，注释写明了 "e.g., for /issue command" |
| 自动运行 `/issue` 通知 | `src/utils/autoRunIssue.tsx` | 当用户在反馈调查中选择 "bad" 时，可自动触发 `/issue`（当前所有分支返回 `false`） |
| Issue 标记横幅 | `src/components/PromptInput/IssueFlagBanner.tsx` | 在对话中检测到摩擦时提示用户使用 `/issue`（当前组件直接 `return null`） |
| 摩擦信号检测 | `src/hooks/useIssueFlagBanner.ts` | 检测用户表达不满的关键词（如 "that's wrong", "try again" 等） |
| Git 状态保留 | `src/utils/git.ts` | 专门有 "Preserved git state for issue submission" 的函数 |

### 3.4 摩擦信号检测机制

```typescript
// src/hooks/useIssueFlagBanner.ts (line 27-43)
const FRICTION_PATTERNS = [
  /^no[,!]\s/i,  // "No," 或 "No!" 开头
  /\bthat'?s (wrong|incorrect|not (what|right|correct))\b/i,
  /\bnot what I (asked|wanted|meant|said)\b/i,
  /\bI (said|asked|wanted|told you|already said)\b/i,
  /\bwhy did you\b/i,
  /\byou should(n'?t| not)? have\b/i,
  /\btry again\b/i,
  /\b(undo|revert) (that|this|it|what you)\b/i,
];
```

**触发条件**（`useIssueFlagBanner` 函数）：
1. `USER_TYPE === 'ant'`（编译时常量检查）
2. 会话兼容性检查（排除包含外部命令的会话）
3. 摩擦信号检测（匹配上述正则）
4. 冷却期控制（30 分钟）
5. 最小提交次数（3 次）

### 3.5 Prompt Dump 实现

```typescript
// src/services/api/dumpPrompts.ts (line 13-14)
// Cache last few API requests for ant users (e.g., for /issue command)
const MAX_CACHED_REQUESTS = 5
const cachedApiRequests: Array<{ timestamp: string; request: unknown }> = []

// line 48-57
export function addApiRequestToCache(requestData: unknown): void {
  if (process.env.USER_TYPE !== 'ant') return
  cachedApiRequests.push({
    timestamp: new Date().toISOString(),
    request: requestData,
  })
  if (cachedApiRequests.length > MAX_CACHED_REQUESTS) {
    cachedApiRequests.shift()
  }
}
```

---

## 4. 关键代码路径与文件引用

### 4.1 核心文件

| 文件路径 | 职责 |
|---------|------|
| `src/commands/issue/index.js` | `/issue` 命令的当前 stub 实现 |
| `src/commands.ts` | 命令注册中心，导入 `issue` 并将其加入 `INTERNAL_ONLY_COMMANDS` |
| `src/types/command.ts` | `CommandBase`、`isCommandEnabled`、`getCommandName` 等类型定义 |

### 4.2 支撑与残留文件

| 文件路径 | 职责 |
|---------|------|
| `src/services/api/dumpPrompts.ts` | 为 `ant` 用户缓存 API 请求和响应流，明确为 `/issue` 提供调试数据 |
| `src/utils/git.ts` | 提供 `getGitStateForIssueSubmission()` 等函数，保留 Git 状态用于 issue 提交 |
| `src/utils/autoRunIssue.tsx` | 自动运行 `/issue` 的通知组件和状态判断逻辑 |
| `src/components/PromptInput/IssueFlagBanner.tsx` | 对话中的 `/issue` 提示横幅（当前返回 `null`） |
| `src/hooks/useIssueFlagBanner.ts` | 判断何时显示 issue flag banner 的 hook |
| `src/constants/prompts.ts` | 系统提示中推荐用户使用 `/issue` 或 `/share` |

### 4.3 代码引用链

```
src/commands/issue/index.js
    ↑ 被导入
src/commands.ts:7
    ↑ 加入 INTERNAL_ONLY_COMMANDS
src/commands.ts:225-254
    ↑ 条件展开到 COMMANDS()
src/commands.ts:343-345 (USER_TYPE === 'ant' && !IS_DEMO)
    ↑ 被 getCommands() 过滤
src/commands.ts:476-517 (isCommandEnabled)
    ↑ 用于 REPL 渲染和 SkillTool
src/screens/REPL.tsx

[并行残留路径]
src/services/api/dumpPrompts.ts
    → 持续为 ant 用户记录 API 请求（注释说明用于 /issue）
src/utils/autoRunIssue.tsx
    → 反馈调查后可自动触发 /issue（当前全部关闭）
src/components/PromptInput/IssueFlagBanner.tsx
    → 原用于在对话中提示 /issue（当前直接 return null）
```

---

## 5. 依赖与外部交互

### 5.1 内部依赖

| 依赖模块 | 用途 |
|---------|------|
| `src/commands.ts` | 命令注册表，是 `issue` stub 的唯一直接调用方 |
| `src/types/command.ts` | 类型系统，stub 必须至少满足 `CommandBase` 的 `name` 字段约束 |
| `src/services/api/dumpPrompts.ts` | 原本 `/issue` 会依赖此模块读取 `getLastApiRequests()` |
| `src/utils/git.ts` | 原本 `/issue` 会调用 `getGitStateForIssueSubmission()` |

### 5.2 外部交互

| 服务/系统 | 用途 |
|----------|------|
| 文件系统 | `dumpPrompts.ts` 将 prompt dump 写入 `~/.claude/dump-prompts/<sessionId>.jsonl` |
| GitHub | `Feedback.tsx` 中的 `createGitHubIssueUrl()` 构造指向 GitHub Issues 的 URL |
| Anthropic API | `dumpPrompts.ts` 通过包装 `fetch` 拦截所有 API 请求和响应 |

### 5.3 环境变量约束

| 变量 | 影响 |
|------|------|
| `USER_TYPE === 'ant'` | `INTERNAL_ONLY_COMMANDS` 仅在此时暴露；`dumpPrompts.ts` 也仅在此时记录数据 |
| `IS_DEMO` | 如果设置，即使 `USER_TYPE === 'ant'`，`INTERNAL_ONLY_COMMANDS` 也不会被加入 |

---

## 6. 风险、边界与改进建议

### 6.1 当前风险

#### 6.1.1 僵尸代码与维护负担

`dumpPrompts.ts`、`autoRunIssue.tsx`、`IssueFlagBanner.tsx`、`useIssueFlagBanner.ts` 等模块持续存在，但服务的命令已死。这些代码增加了：
- 编译体积
- 认知负担
- `dumpPrompts.ts` 仍为所有 `ant` 用户执行文件 I/O（虽然是异步的）

#### 6.1.2 系统提示误导

`src/constants/prompts.ts` 中的系统提示仍然教导 Claude 向用户推荐 `/issue`。由于该命令已被禁用，如果用户尝试输入 `/issue`，CLI 会返回"命令未找到"，导致糟糕的用户体验。

**相关代码**（`src/constants/prompts.ts` line 245）：
```typescript
If the user reports a bug, slowness, or unexpected behavior with Claude Code itself 
(as opposed to asking you to fix their own code), recommend the appropriate slash command: 
/issue for model-related problems (odd outputs, wrong tool choices, hallucinations, refusals), 
or /share to upload the full session transcript for product bugs, crashes, slowness, 
or general issues.
```

#### 6.1.3 类型安全缺口

`issue` stub 缺少 `description` 字段，而 `CommandBase` 中 `description` 是必填的。虽然 `isEnabled: () => false` 确保它不会进入任何实际使用路径，但这是一个潜在的类型不一致。

#### 6.1.4 Prompt Dump 磁盘占用

`dumpPrompts.ts` 为每个 session 写入一个 `.jsonl` 文件，但没有清理逻辑。长期运行的 `ant` 用户可能会在 `~/.claude/dump-prompts/` 中积累大量历史文件。

### 6.2 边界情况

| 边界情况 | 说明 |
|---------|------|
| 命令查找 | 用户手动输入 `/issue` 时，`findCommand()` 会返回 `Command issue not found` 错误 |
| 动态技能冲突 | 如果插件注册了名为 `issue` 的命令，由于内置 `issue` 已被过滤，不会发生冲突 |
| 意外启用风险 | 如果某人修改了 `isEnabled` 却忘了改 `name`，可能出现命名冲突 |

### 6.3 改进建议

#### 短期（清理债务）

1. **更新系统提示**：从 `src/constants/prompts.ts` 中移除或修改推荐 `/issue` 的段落，统一引导用户使用 `/feedback` 或 `/share`

2. **删除或注释残留 UI 组件**：
   - `src/components/PromptInput/IssueFlagBanner.tsx` 当前已是 `return null`，可考虑彻底删除
   - `src/utils/autoRunIssue.tsx` 中所有 `shouldAutoRunIssue` 分支恒为 `false`，可考虑简化或移除

3. **评估 `dumpPrompts.ts` 的必要性**：如果 `/issue` 已死且没有替代命令消费 prompt dump，应停止为 `ant` 用户持续写入磁盘，或至少增加自动清理策略

#### 中期（架构整理）

4. **统一 Stub 管理**：当前 17+ 个 stub 命令分散在 `src/commands/*/` 中，每个都是一行相同的代码。可考虑引入一个统一的 `createStubCommand(name)` 工厂函数，或将它们合并到一个 `src/commands/stubs.ts` 中，减少目录噪音

5. **补齐类型安全**：如果 stub 文件需要保留，应确保它们满足 `Command` 类型的最小约束（至少提供 `description` 和 `type`），或调整类型定义以允许这种占位符

#### 长期（功能决策）

6. **决定 `/issue` 的命运**：
   - **选项 A - 彻底移除**：删除 `src/commands/issue/` 目录，从 `src/commands.ts` 和 `INTERNAL_ONLY_COMMANDS` 中移除引用，清理所有残留模块
   - **选项 B - 恢复功能**：将 `/issue` 重新实现为一个真正的 `local-jsx` 或 `prompt` 命令，复用 `Feedback.tsx` 的 UI 和 `dumpPrompts.ts` 的数据收集能力
   - **选项 C - 合并到 `/feedback`**：将 `/issue` 的能力（特别是 prompt dump 自动附加）整合进 `/feedback` 命令，然后彻底删除 `/issue` 的独立入口

**推荐路径**：选项 A 或 C。既然 `/issue` 已被系统性地禁用，且 `/feedback` 已能覆盖大部分用户反馈场景，继续维护一个独立的僵尸命令入口没有明显价值。

---

## 7. 总结

`src/commands/issue/index.js` 是一个**功能占位符**，其核心价值在于：

1. **构建时隔离**：通过 `USER_TYPE` 环境变量在编译时隔离内部功能
2. **保留重新启用可能性**：代码库中保留了大量相关支撑逻辑
3. **类型系统兼容**：导出的对象满足 `CommandBase` 的最小约束

该命令当前处于**完全禁用状态**，实际实现分散在 `dumpPrompts.ts`、`autoRunIssue.tsx`、`useIssueFlagBanner.ts` 等残留模块中。建议进行代码清理，要么彻底移除相关功能，要么将能力整合到 `/feedback` 命令中。
