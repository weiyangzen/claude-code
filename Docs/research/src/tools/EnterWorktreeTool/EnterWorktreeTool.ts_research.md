# EnterWorktreeTool.ts 研究文档

## 场景与职责

EnterWorktreeTool 是 Claude Code CLI 中负责创建并进入 Git Worktree（工作树）的核心工具。它允许用户在同一会话中创建一个隔离的 Git 工作目录，用于并行开发、测试或实验性工作，而不会影响主分支的工作状态。

**核心场景：**
1. 用户明确请求使用 worktree（如 "start a worktree", "work in a worktree"）
2. 需要在隔离环境中进行实验性开发
3. 并行处理多个功能分支而不切换主工作目录

**职责边界：**
- 仅响应用户显式提及 "worktree" 的请求
- 不处理普通的分支切换或创建请求（由 git 命令处理）
- 必须在 Git 仓库中运行，或配置了 WorktreeCreate/WorktreeRemove hooks

## 功能点目的

### 1. 工作树创建与进入
- 创建新的 Git worktree 在 `.claude/worktrees/<slug>/` 目录下
- 基于当前 HEAD 创建新分支（命名格式：`worktree-<flattened-slug>`）
- 自动切换会话的工作目录到新的 worktree

### 2. 命名与验证
- 支持可选的自定义名称参数 `name`
- 名称验证：每段只能包含字母、数字、点、下划线和破折号；总长度不超过 64 字符
- 支持嵌套路径（如 `user/feature`），内部会扁平化为 `user+feature`

### 3. 会话状态管理
- 保存原始工作目录到 `originalCwd`
- 记录 worktree 会话信息（路径、分支、会话ID等）
- 清除系统提示词缓存，确保环境信息重新计算
- 清除依赖于 CWD 的内存文件缓存

### 4. 防重复进入保护
- 检查 `getCurrentWorktreeSession()`，防止在已处于 worktree 会话时再次进入

## 具体技术实现

### 关键数据结构

```typescript
// 输入模式（Input Schema）
{
  name?: string  // 可选的 worktree 名称
}

// 输出模式（Output Schema）
{
  worktreePath: string
  worktreeBranch?: string
  message: string
}
```

### 核心流程

1. **前置检查**
   ```typescript
   // 验证不在已有的 worktree 会话中
   if (getCurrentWorktreeSession()) {
     throw new Error('Already in a worktree session')
   }
   ```

2. **解析主仓库根目录**
   ```typescript
   const mainRepoRoot = findCanonicalGitRoot(getCwd())
   if (mainRepoRoot && mainRepoRoot !== getCwd()) {
     process.chdir(mainRepoRoot)
     setCwd(mainRepoRoot)
   }
   ```

3. **生成/使用 Slug**
   ```typescript
   const slug = input.name ?? getPlanSlug()
   ```

4. **创建 Worktree 会话**
   ```typescript
   const worktreeSession = await createWorktreeForSession(getSessionId(), slug)
   ```

5. **切换工作目录**
   ```typescript
   process.chdir(worktreeSession.worktreePath)
   setCwd(worktreeSession.worktreePath)
   setOriginalCwd(getCwd())
   saveWorktreeState(worktreeSession)
   ```

6. **清除缓存**
   ```typescript
   clearSystemPromptSections()
   clearMemoryFileCaches()
   getPlansDirectory.cache.clear?.()
   ```

7. **记录分析事件**
   ```typescript
   logEvent('tengu_worktree_created', { mid_session: true })
   ```

### 工具配置

```typescript
{
  name: ENTER_WORKTREE_TOOL_NAME,  // 'EnterWorktree'
  searchHint: 'create an isolated git worktree and switch into it',
  maxResultSizeChars: 100_000,
  shouldDefer: true,  // 延迟加载工具
  userFacingName: () => 'Creating worktree'
}
```

## 关键代码路径与文件引用

### 当前文件
- `/src/tools/EnterWorktreeTool/EnterWorktreeTool.ts` - 主工具实现

### 依赖文件

| 文件路径 | 用途 |
|---------|------|
| `/src/tools/EnterWorktreeTool/constants.ts` | 工具名称常量定义 |
| `/src/tools/EnterWorktreeTool/prompt.ts` | 工具提示词（使用指南） |
| `/src/tools/EnterWorktreeTool/UI.tsx` | UI 渲染组件 |
| `/src/utils/worktree.ts` | Worktree 创建/管理核心逻辑 |
| `/src/utils/sessionStorage.ts` | 会话状态持久化 |
| `/src/utils/plans.ts` | Plan slug 生成与管理 |
| `/src/utils/Shell.ts` | 工作目录切换 (`setCwd`) |
| `/src/utils/git.ts` | Git 根目录查找 (`findCanonicalGitRoot`) |
| `/src/utils/claudemd.ts` | 内存文件缓存清除 |
| `/src/bootstrap/state.ts` | 会话状态管理 (`getSessionId`, `setOriginalCwd`) |
| `/src/constants/systemPromptSections.ts` | 系统提示词缓存清除 |
| `/src/services/analytics/index.ts` | 分析事件记录 |
| `/src/Tool.ts` | 工具基类和 `buildTool` 工厂 |
| `/src/utils/lazySchema.ts` | 懒加载 Zod schema |

### 关联工具
- `/src/tools/ExitWorktreeTool/ExitWorktreeTool.ts` - 退出 worktree 的对应工具

## 依赖与外部交互

### 外部系统交互

1. **Git 命令执行**
   - 通过 `createWorktreeForSession` 间接调用 `git worktree add`
   - 分支命名：`worktree-${flattenSlug(slug)}`
   - 工作树路径：`.claude/worktrees/${flattenSlug(slug)}`

2. **文件系统操作**
   - `process.chdir()` - 切换 Node.js 进程工作目录
   - 会话状态持久化到项目配置

3. **分析事件**
   - `tengu_worktree_created` - 记录 worktree 创建事件

### 内部模块依赖图

```
EnterWorktreeTool.ts
├── constants.ts          # ENTER_WORKTREE_TOOL_NAME
├── prompt.ts             # getEnterWorktreeToolPrompt()
├── UI.tsx                # renderToolUseMessage, renderToolResultMessage
├── ../../bootstrap/state.ts
│   ├── getSessionId()
│   ├── setOriginalCwd()
│   └── getCwd() (via ../../utils/cwd.js)
├── ../../constants/systemPromptSections.ts
│   └── clearSystemPromptSections()
├── ../../services/analytics/index.ts
│   └── logEvent()
├── ../../Tool.ts
│   ├── buildTool()
│   └── Tool type
├── ../../utils/claudemd.ts
│   └── clearMemoryFileCaches()
├── ../../utils/cwd.ts
│   └── getCwd()
├── ../../utils/git.ts
│   └── findCanonicalGitRoot()
├── ../../utils/lazySchema.ts
│   └── lazySchema()
├── ../../utils/plans.ts
│   └── getPlanSlug()
├── ../../utils/Shell.ts
│   └── setCwd()
└── ../../utils/sessionStorage.ts
    └── saveWorktreeState()
        └── ../../utils/worktree.ts
            └── createWorktreeForSession()
```

## 风险、边界与改进建议

### 已知风险

1. **嵌套 Worktree 限制**
   - 工具明确禁止在已有 worktree 会话中再次进入 worktree
   - 这是为了防止状态混乱和资源竞争

2. **Git 仓库依赖**
   - 必须在 Git 仓库中运行，或配置 WorktreeCreate hooks
   - 非 Git 环境会抛出错误

3. **路径扁平化副作用**
   - 嵌套 slug（如 `user/feature`）会被扁平化为 `user+feature`
   - 这避免了 D/F 冲突（目录/文件冲突）和嵌套 worktree 删除问题

4. **缓存清除影响**
   - 进入 worktree 会清除系统提示词缓存和文件缓存
   - 这会导致下一次 API 调用时重新计算环境信息，可能影响性能

### 边界情况

1. **从 Worktree 内部调用**
   - 如果当前已在 git worktree（非 EnterWorktree 创建的），会找到 canonical git root 并切换过去
   - 这确保了 worktree 创建在主仓库上下文中进行

2. **Slug 冲突**
   - 如果同名 worktree 已存在，会复用现有 worktree（快速恢复路径）
   - 具体逻辑在 `getOrCreateWorktree()` 中实现

3. **会话恢复**
   - 通过 `saveWorktreeState()` 持久化会话状态
   - 支持 `--resume` 时恢复 worktree 上下文

### 改进建议

1. **错误处理增强**
   - 当前对 `findCanonicalGitRoot` 失败的处理较简单，可提供更详细的错误信息
   - 建议区分 "不在 Git 仓库" 和 "Git 命令失败" 两种情况

2. **配置验证**
   - 可在工具启用时预检查 WorktreeCreate hooks 配置
   - 提前告知用户可用的 worktree 创建方式

3. **性能优化**
   - 考虑延迟清除缓存，仅在必要时执行
   - 对于频繁进入/退出 worktree 的场景，缓存清除开销较大

4. **监控增强**
   - 可添加 worktree 创建耗时监控
   - 记录 worktree 使用频率和保留时长

5. **文档完善**
   - 提示词中可增加 worktree 与常规分支工作流的对比说明
   - 帮助用户理解何时应该选择 worktree

### 安全考虑

1. **路径遍历防护**
   - `validateWorktreeSlug` 防止 `../` 等路径遍历攻击
   - 每个路径段独立验证，拒绝 `.` 和 `..`

2. **权限检查**
   - 工具本身不直接检查文件权限
   - 依赖底层 `git worktree add` 命令的权限控制

3. **会话隔离**
   - worktree 会话状态与会话 ID 绑定
   - 防止跨会话的 worktree 操作混淆
