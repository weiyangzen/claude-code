# ghPrStatus.ts 深度研究文档

## 场景与职责

`ghPrStatus.ts` 提供 GitHub CLI (`gh`) 集成的 PR 状态获取功能，用于：

1. **PR 状态显示**：在状态栏显示当前分支的 PR 状态
2. **代码审查工作流**：支持审查状态（approved、changes_requested、pending 等）
3. **GitHub 集成**：通过 `gh pr view` 命令获取 PR 信息

该模块是 GitHub 工作流支持的一部分，与 `usePrStatus` hook 和 `PrBadge` 组件配合使用。

## 功能点目的

### 1. PR 状态获取
- 使用 `gh pr view --json` 获取当前分支的 PR 信息
- 解析 PR 编号、URL、审查状态、草稿状态

### 2. 状态派生
- 从 GitHub API 返回的 `reviewDecision` 派生内部状态
- 处理草稿 PR 的特殊逻辑

### 3. 智能过滤
- 跳过默认分支（main/master）上的 PR
- 跳过已合并或已关闭的 PR
- 跳过从默认分支创建的 PR

## 具体技术实现

### 关键数据结构

```typescript
export type PrReviewState =
  | 'approved'           // 已批准
  | 'pending'            // 等待审查
  | 'changes_requested'  // 需要修改
  | 'draft'              // 草稿
  | 'merged'             // 已合并（内部使用）
  | 'closed'             // 已关闭（内部使用）

export type PrStatus = {
  number: number        // PR 编号
  url: string           // PR URL
  reviewState: PrReviewState  // 审查状态
}
```

### 关键流程

#### PR 状态获取流程
```typescript
export async function fetchPrStatus(): Promise<PrStatus | null> {
  // 1. 检查是否在 Git 仓库
  const isGit = await getIsGit()
  if (!isGit) return null

  // 2. 跳过默认分支
  const [branch, defaultBranch] = await Promise.all([
    getBranch(),
    getDefaultBranch(),
  ])
  if (branch === defaultBranch) return null

  // 3. 调用 gh pr view
  const { stdout, code } = await execFileNoThrow(
    'gh',
    [
      'pr',
      'view',
      '--json',
      'number,url,reviewDecision,isDraft,headRefName,state',
    ],
    { timeout: GH_TIMEOUT_MS, preserveOutputOnError: false },
  )

  if (code !== 0 || !stdout.trim()) return null

  // 4. 解析并过滤
  try {
    const data = jsonParse(stdout) as { ... }

    // 跳过从默认分支创建的 PR
    if (data.headRefName === defaultBranch ||
        data.headRefName === 'main' ||
        data.headRefName === 'master') {
      return null
    }

    // 跳过已合并/关闭的 PR
    if (data.state === 'MERGED' || data.state === 'CLOSED') {
      return null
    }

    return {
      number: data.number,
      url: data.url,
      reviewState: deriveReviewState(data.isDraft, data.reviewDecision),
    }
  } catch {
    return null
  }
}
```

#### 状态派生逻辑
```typescript
export function deriveReviewState(
  isDraft: boolean,
  reviewDecision: string,
): PrReviewState {
  if (isDraft) return 'draft'
  switch (reviewDecision) {
    case 'APPROVED':
      return 'approved'
    case 'CHANGES_REQUESTED':
      return 'changes_requested'
    default:
      return 'pending'
  }
}
```

### GitHub API 字段映射

| GitHub API | 内部状态 |
|-----------|---------|
| `isDraft: true` | `draft` |
| `reviewDecision: 'APPROVED'` | `approved` |
| `reviewDecision: 'CHANGES_REQUESTED'` | `changes_requested` |
| `reviewDecision: 'REVIEW_REQUIRED'` 或空 | `pending` |
| `state: 'MERGED'` | 过滤掉（返回 null） |
| `state: 'CLOSED'` | 过滤掉（返回 null） |

## 关键代码路径与文件引用

### 核心导出
- `PrReviewState` - PR 审查状态类型
- `PrStatus` - PR 状态对象类型
- `deriveReviewState(isDraft, reviewDecision)` - 状态派生函数
- `fetchPrStatus()` - 获取 PR 状态

### 依赖关系

**被以下模块导入**：
- `src/hooks/usePrStatus.ts` - PR 状态 Hook
- `src/components/PrBadge.tsx` - PR 徽章组件

**依赖的模块**：
- `src/utils/execFileNoThrow.ts` - `execFileNoThrow`
- `src/utils/git.ts` - `getBranch`, `getDefaultBranch`, `getIsGit`
- `src/utils/slowOperations.ts` - `jsonParse`

### 文件位置
- 源码：`src/utils/ghPrStatus.ts` (106 行)

## 依赖与外部交互

### Node.js 内置模块
- 无

### 项目内部依赖
- `src/utils/execFileNoThrow.ts` - 安全的命令执行
- `src/utils/git.ts` - Git 状态获取
- `src/utils/slowOperations.ts` - JSON 解析

### 外部依赖
- GitHub CLI (`gh`) - 需要在 PATH 中且已认证

## 风险、边界与改进建议

### 已知风险

1. **gh CLI 依赖**：需要用户安装并配置 GitHub CLI
2. **认证状态**：`gh` 需要登录才能访问 PR 信息
3. **超时处理**：5 秒超时可能不足以处理慢速网络
4. **API 变更**：GitHub API 字段变更可能导致解析失败

### 边界情况

1. **非 Git 目录**：返回 null
2. **默认分支**：返回 null（避免显示最近合并的 PR）
3. **无 PR 的分支**：返回 null
4. **从 main/master 创建的 PR**：返回 null（避免误报）
5. **已合并/关闭的 PR**：返回 null
6. **fork 仓库**：行为取决于 `gh` 的处理

### 改进建议

1. **错误暴露**：考虑添加调试日志帮助诊断问题
2. **缓存机制**：添加短期缓存避免重复调用 gh
3. **重试逻辑**：对网络错误添加重试
4. **配置超时**：允许通过环境变量配置超时时间
5. **测试覆盖**：当前没有专门的测试文件，建议添加单元测试
6. **GHE 支持**：考虑添加 GitHub Enterprise 支持
7. **回退机制**：当 gh 不可用时，考虑使用 git remote 信息构造 PR URL

### 相关组件

该模块与以下组件配合工作：
- `usePrStatus` hook：定期轮询 PR 状态
- `PrBadge` 组件：在 UI 中显示 PR 状态和链接
