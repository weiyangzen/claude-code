# setupGitHubActions.ts 深度研究文档

## 文件基本信息

- **路径**: `src/commands/install-github-app/setupGitHubActions.ts`
- **大小**: ~10KB (325 行)
- **类型**: TypeScript 模块
- **职责**: GitHub Actions 工作流和 Secret 的实际创建与配置

---

## 1. 场景与职责

### 1.1 核心场景

该文件是 GitHub App 安装流程的**后端执行引擎**，负责实际与 GitHub API 交互完成以下操作：

1. 在指定仓库创建 Git 分支
2. 创建工作流 YAML 文件
3. 设置 GitHub Secrets（API Key 或 OAuth Token）
4. 打开浏览器引导用户创建 Pull Request

### 1.2 调用场景

```
install-github-app.tsx 向导完成用户交互
    ↓
runSetupGitHubActions callback 被调用
    ↓
setupGitHubActions() 执行实际设置
    ↓
GitHub API 调用 (通过 gh CLI)
    ↓
浏览器打开 PR 创建页面
```

### 1.3 主要职责

1. **仓库验证**: 确认目标仓库存在且可访问
2. **分支管理**: 基于默认分支创建新分支
3. **工作流创建**: 根据选择创建一种或多种工作流文件
4. **Secret 管理**: 安全地设置 API Key 或 OAuth Token
5. **PR 引导**: 生成 PR URL 并打开浏览器
6. **配置持久化**: 更新全局配置记录安装次数

---

## 2. 功能点目的

### 2.1 工作流类型支持

| 工作流 | 文件名 | 触发条件 | 用途 |
|--------|--------|----------|------|
| Claude PR Assistant | `.github/workflows/claude.yml` | `@claude` 提及 | 交互式 PR/Issue 助手 |
| Claude Code Review | `.github/workflows/claude-code-review.yml` | PR 创建/更新 | 自动代码审查 |

### 2.2 Secret 配置策略

| 认证类型 | Secret 名称 | 工作流参数 |
|----------|-------------|------------|
| API Key (默认) | `ANTHROPIC_API_KEY` | `anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}` |
| API Key (自定义) | 用户指定 | `anthropic_api_key: ${{ secrets.{自定义} }}` |
| OAuth Token | `CLAUDE_CODE_OAUTH_TOKEN` | `claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}` |

### 2.3 错误分类处理

- **仓库不存在**: `repo_not_found`
- **获取默认分支失败**: `failed_to_get_default_branch`
- **获取分支 SHA 失败**: `failed_to_get_branch_sha`
- **创建分支失败**: `failed_to_create_branch`
- **创建工作流文件失败**: `failed_to_create_workflow_file`
- **设置 Secret 失败**: `failed_to_set_api_key_secret`
- **意外错误**: `unexpected_error`

---

## 3. 具体技术实现

### 3.1 核心函数签名

```typescript
export async function setupGitHubActions(
  repoName: string,                    // 仓库名 (owner/repo)
  apiKeyOrOAuthToken: string | null,   // API Key 或 OAuth Token
  secretName: string,                  // Secret 名称
  updateProgress: () => void,          // 进度更新回调
  skipWorkflow = false,               // 是否跳过工作流创建
  selectedWorkflows: Workflow[],       // 选中的工作流类型
  authType: 'api_key' | 'oauth_token', // 认证类型
  context?: {                          // 上下文信息（用于分析）
    useCurrentRepo?: boolean
    workflowExists?: boolean
    secretExists?: boolean
  },
): Promise<void>
```

### 3.2 内部函数: createWorkflowFile

```typescript
async function createWorkflowFile(
  repoName: string,
  branchName: string,
  workflowPath: string,        // 如: .github/workflows/claude.yml
  workflowContent: string,     // YAML 内容模板
  secretName: string,          // 用于替换模板中的 secret 引用
  message: string,             // Git commit message
  context?: {...},             // 分析上下文
): Promise<void>
```

#### 3.2.1 文件存在性检查

```typescript
// 检查文件是否已存在
const checkFileResult = await execFileNoThrow('gh', [
  'api',
  `repos/${repoName}/contents/${workflowPath}`,
  '--jq',
  '.sha',
]);

let fileSha: string | null = null;
if (checkFileResult.code === 0) {
  fileSha = checkFileResult.stdout.trim();  // 存在则获取 SHA 用于更新
}
```

#### 3.2.2 Secret 名称替换逻辑

```typescript
let content = workflowContent;

if (secretName === 'CLAUDE_CODE_OAUTH_TOKEN') {
  // OAuth Token: 替换参数名和 secret 引用
  content = workflowContent.replace(
    /anthropic_api_key: \$\{\{ secrets\.ANTHROPIC_API_KEY \}\}/g,
    `claude_code_oauth_token: \${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}`,
  );
} else if (secretName !== 'ANTHROPIC_API_KEY') {
  // 自定义 Secret 名称: 只替换 secret 引用
  content = workflowContent.replace(
    /anthropic_api_key: \$\{\{ secrets\.ANTHROPIC_API_KEY \}\}/g,
    `anthropic_api_key: \${{ secrets.${secretName} }}`,
  );
}

const base64Content = Buffer.from(content).toString('base64');
```

#### 3.2.3 GitHub API 调用

```typescript
const apiParams = [
  'api',
  '--method', 'PUT',
  `repos/${repoName}/contents/${workflowPath}`,
  '-f', `message=${fileSha ? `"Update ${message}"` : `"${message}"`}`,
  '-f', `content=${base64Content}`,
  '-f', `branch=${branchName}`,
];

if (fileSha) {
  apiParams.push('-f', `sha=${fileSha}`);  // 更新现有文件需要 SHA
}

const createFileResult = await execFileNoThrow('gh', apiParams);
```

#### 3.2.4 错误处理

```typescript
if (createFileResult.code !== 0) {
  // 422 + sha 错误通常表示文件已存在但 SHA 不匹配
  if (createFileResult.stderr.includes('422') && 
      createFileResult.stderr.includes('sha')) {
    throw new Error(
      `Failed to create workflow file ${workflowPath}: ` +
      `A Claude workflow file already exists in this repository. ` +
      `Please remove it first or update it manually.`
    );
  }
  
  // 其他错误附带帮助信息
  const helpText =
    '\n\nNeed help? Common issues:\n' +
    '· Permission denied → Run: gh auth refresh -h github.com -s repo,workflow\n' +
    '· Not authorized → Ensure you have admin access to the repository\n' +
    '· For manual setup → Visit: https://github.com/anthropics/claude-code-action';
  
  throw new Error(`Failed to create workflow file ${workflowPath}: ${createFileResult.stderr}${helpText}`);
}
```

### 3.3 主流程实现

#### 3.3.1 仓库验证

```typescript
// 检查仓库是否存在
const repoCheckResult = await execFileNoThrow('gh', [
  'api', `repos/${repoName}`, '--jq', '.id'
]);
if (repoCheckResult.code !== 0) {
  logEvent('tengu_setup_github_actions_failed', {
    reason: 'repo_not_found',
    exit_code: repoCheckResult.code,
    ...context,
  });
  throw new Error(`Failed to access repository ${repoName}: ${repoCheckResult.stderr}`);
}
```

#### 3.3.2 获取默认分支

```typescript
const defaultBranchResult = await execFileNoThrow('gh', [
  'api', `repos/${repoName}`, '--jq', '.default_branch'
]);
if (defaultBranchResult.code !== 0) {
  logEvent('tengu_setup_github_actions_failed', {
    reason: 'failed_to_get_default_branch',
    exit_code: defaultBranchResult.code,
    ...context,
  });
  throw new Error(`Failed to get default branch: ${defaultBranchResult.stderr}`);
}
const defaultBranch = defaultBranchResult.stdout.trim();
```

#### 3.3.3 获取分支 SHA

```typescript
const shaResult = await execFileNoThrow('gh', [
  'api',
  `repos/${repoName}/git/ref/heads/${defaultBranch}`,
  '--jq', '.object.sha'
]);
if (shaResult.code !== 0) {
  logEvent('tengu_setup_github_actions_failed', {
    reason: 'failed_to_get_branch_sha',
    exit_code: shaResult.code,
    ...context,
  });
  throw new Error(`Failed to get branch SHA: ${shaResult.stderr}`);
}
const sha = shaResult.stdout.trim();
```

#### 3.3.4 创建分支

```typescript
if (!skipWorkflow) {
  updateProgress();
  
  // 使用时间戳生成唯一分支名
  branchName = `add-claude-github-actions-${Date.now()}`;
  
  const createBranchResult = await execFileNoThrow('gh', [
    'api',
    '--method', 'POST',
    `repos/${repoName}/git/refs`,
    '-f', `ref=refs/heads/${branchName}`,
    '-f', `sha=${sha}`
  ]);
  
  if (createBranchResult.code !== 0) {
    logEvent('tengu_setup_github_actions_failed', {
      reason: 'failed_to_create_branch',
      exit_code: createBranchResult.code,
      ...context,
    });
    throw new Error(`Failed to create branch: ${createBranchResult.stderr}`);
  }
}
```

#### 3.3.5 创建工作流文件

```typescript
updateProgress();
const workflows = [];

if (selectedWorkflows.includes('claude')) {
  workflows.push({
    path: '.github/workflows/claude.yml',
    content: WORKFLOW_CONTENT,
    message: 'Claude PR Assistant workflow',
  });
}

if (selectedWorkflows.includes('claude-review')) {
  workflows.push({
    path: '.github/workflows/claude-code-review.yml',
    content: CODE_REVIEW_PLUGIN_WORKFLOW_CONTENT,
    message: 'Claude Code Review workflow',
  });
}

for (const workflow of workflows) {
  await createWorkflowFile(
    repoName,
    branchName,
    workflow.path,
    workflow.content,
    secretName,
    workflow.message,
    context,
  );
}
```

#### 3.3.6 设置 Secret

```typescript
updateProgress();
if (apiKeyOrOAuthToken) {
  const setSecretResult = await execFileNoThrow('gh', [
    'secret', 'set', secretName,
    '--body', apiKeyOrOAuthToken,
    '--repo', repoName,
  ]);
  
  if (setSecretResult.code !== 0) {
    logEvent('tengu_setup_github_actions_failed', {
      reason: 'failed_to_set_api_key_secret',
      exit_code: setSecretResult.code,
      ...context,
    });
    
    const helpText = '...';  // 帮助信息
    throw new Error(`Failed to set API key secret: ${setSecretResult.stderr || 'Unknown error'}${helpText}`);
  }
}
```

#### 3.3.7 打开 PR 创建页面

```typescript
if (!skipWorkflow && branchName) {
  updateProgress();
  
  // 使用 GitHub 的快速 PR 创建 URL 参数
  const compareUrl = `https://github.com/${repoName}/compare/${defaultBranch}...${branchName}?` +
    `quick_pull=1&` +
    `title=${encodeURIComponent(PR_TITLE)}&` +
    `body=${encodeURIComponent(PR_BODY)}`;
  
  await openBrowser(compareUrl);
}
```

#### 3.3.8 更新全局配置

```typescript
logEvent('tengu_setup_github_actions_completed', {
  skip_workflow: skipWorkflow,
  has_api_key: !!apiKeyOrOAuthToken,
  auth_type: authType,
  using_default_secret_name: secretName === 'ANTHROPIC_API_KEY',
  selected_claude_workflow: selectedWorkflows.includes('claude'),
  selected_claude_review_workflow: selectedWorkflows.includes('claude-review'),
  ...context,
});

// 记录安装次数
saveGlobalConfig(current => ({
  ...current,
  githubActionSetupCount: (current.githubActionSetupCount ?? 0) + 1,
}));
```

---

## 4. 关键代码路径与文件引用

### 4.1 导入依赖

```typescript
// 分析服务
import {
  type AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS,
  logEvent,
} from 'src/services/analytics/index.js';

// 配置管理
import { saveGlobalConfig } from 'src/utils/config.js';

// 工作流内容模板
import {
  CODE_REVIEW_PLUGIN_WORKFLOW_CONTENT,
  PR_BODY,
  PR_TITLE,
  WORKFLOW_CONTENT,
} from '../../constants/github-app.js';

// 工具函数
import { openBrowser } from '../../utils/browser.js';
import { execFileNoThrow } from '../../utils/execFileNoThrow.js';
import { logError } from '../../utils/log.js';

// 类型
import type { Workflow } from './types.js';
```

### 4.2 相关文件路径

| 文件 | 路径 | 用途 |
|------|------|------|
| 主执行模块 | `src/commands/install-github-app/setupGitHubActions.ts` | 本文件 |
| 工作流模板 | `src/constants/github-app.ts` | YAML 内容、PR 模板 |
| 类型定义 | `src/commands/install-github-app/types.js` | Workflow 类型 |
| 浏览器工具 | `src/utils/browser.ts` | 打开浏览器 |
| 执行工具 | `src/utils/execFileNoThrow.ts` | 安全执行 gh CLI |
| 配置工具 | `src/utils/config.ts` | 保存全局配置 |
| 日志工具 | `src/utils/log.ts` | 错误日志 |
| 分析服务 | `src/services/analytics/index.ts` | 事件追踪 |

### 4.3 工作流模板内容

#### Claude PR Assistant 工作流

```yaml
name: Claude Code

on:
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]
  issues:
    types: [opened, assigned]
  pull_request_review:
    types: [submitted]

jobs:
  claude:
    if: |
      (github.event_name == 'issue_comment' && contains(github.event.comment.body, '@claude')) ||
      (github.event_name == 'pull_request_review_comment' && contains(github.event.comment.body, '@claude')) ||
      (github.event_name == 'pull_request_review' && contains(github.event.review.body, '@claude')) ||
      (github.event_name == 'issues' && (contains(github.event.issue.body, '@claude') || contains(github.event.issue.title, '@claude')))
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: read
      issues: read
      id-token: write
      actions: read
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 1

      - name: Run Claude Code
        id: claude
        uses: anthropics/claude-code-action@v1
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
          additional_permissions: |
            actions: read
```

#### Claude Code Review 工作流

```yaml
name: Claude Code Review

on:
  pull_request:
    types: [opened, synchronize, ready_for_review, reopened]

jobs:
  claude-review:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: read
      issues: read
      id-token: write

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 1

      - name: Run Claude Code Review
        id: claude-review
        uses: anthropics/claude-code-action@v1
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
          plugin_marketplaces: 'https://github.com/anthropics/claude-code.git'
          plugins: 'code-review@claude-code-plugins'
          prompt: '/code-review:code-review ${{ github.repository }}/pull/${{ github.event.pull_request.number }}'
```

---

## 5. 依赖与外部交互

### 5.1 外部依赖

| 依赖 | 用途 |
|------|------|
| `gh` (GitHub CLI) | 所有 GitHub API 调用 |
| 浏览器 | 打开 PR 创建页面 |

### 5.2 GitHub CLI 命令详解

```bash
# 1. 验证仓库存在
gh api repos/{owner}/{repo} --jq '.id'

# 2. 获取默认分支
gh api repos/{owner}/{repo} --jq '.default_branch'

# 3. 获取分支 SHA
gh api repos/{owner}/{repo}/git/ref/heads/{branch} --jq '.object.sha'

# 4. 创建新分支
gh api --method POST repos/{owner}/{repo}/git/refs \
  -f ref=refs/heads/{new-branch} \
  -f sha={sha}

# 5. 创建/更新文件
gh api --method PUT repos/{owner}/{repo}/contents/{path} \
  -f message="Commit message" \
  -f content={base64-content} \
  -f branch={branch} \
  -f sha={sha}  # 更新时需要

# 6. 设置 Secret
gh secret set {name} \
  --body {value} \
  --repo {owner}/{repo}
```

### 5.3 GitHub API 端点

| 端点 | 方法 | 用途 |
|------|------|------|
| `/repos/{owner}/{repo}` | GET | 验证仓库存在 |
| `/repos/{owner}/{repo}/git/ref/heads/{branch}` | GET | 获取分支引用 |
| `/repos/{owner}/{repo}/git/refs` | POST | 创建新分支 |
| `/repos/{owner}/{repo}/contents/{path}` | PUT | 创建/更新文件 |

### 5.4 分析事件

| 事件名 | 触发时机 | 包含数据 |
|--------|----------|----------|
| `tengu_setup_github_actions_started` | 开始设置 | skip_workflow, has_api_key, using_default_secret_name, selected_workflows |
| `tengu_setup_github_actions_failed` | 设置失败 | reason, exit_code, context |
| `tengu_setup_github_actions_completed` | 设置完成 | skip_workflow, has_api_key, auth_type, selected_workflows |

---

## 6. 风险、边界与改进建议

### 6.1 潜在风险

#### 6.1.1 安全风险

1. **Secret 在命令行暴露**
   ```typescript
   // 当前实现: Secret 作为命令行参数传递
   gh secret set {name} --body {value} --repo {repo}
   ```
   - 风险: 在进程列表中可能暴露
   - **建议**: 使用 stdin 传递敏感信息
   ```typescript
   // 更安全的做法
   execFileNoThrow('gh', ['secret', 'set', name, '--repo', repo], {
     input: value  // 通过 stdin 传递
   });
   ```

2. **Base64 编码依赖**
   - 使用 `Buffer.from(content).toString('base64')` 编码文件内容
   - 对于大文件可能内存占用高
   - **当前场景**: 工作流文件较小，风险可控

3. **URL 构造注入风险**
   ```typescript
   const compareUrl = `https://github.com/${repoName}/compare/...`;
   ```
   - 如果 `repoName` 包含特殊字符可能导致 URL 解析问题
   - **建议**: 使用 URL 构造器
   ```typescript
   const url = new URL(`/compare/...`, `https://github.com/${repoName}`);
   ```

#### 6.1.2 功能风险

1. **分支名冲突**
   ```typescript
   branchName = `add-claude-github-actions-${Date.now()}`;
   ```
   - 使用时间戳理论上不会冲突，但极端情况下可能
   - **建议**: 添加随机字符串增强唯一性

2. **SHA 竞态条件**
   - 获取 SHA 和创建文件之间，如果默认分支被更新，SHA 会失效
   - **当前处理**: 422 错误会提示文件已存在
   - **改进**: 添加重试逻辑

3. **网络超时**
   - 大文件或慢网络可能导致操作超时
   - **当前**: 依赖 `execFileNoThrow` 的默认超时（10分钟）

### 6.2 边界情况

| 场景 | 当前行为 | 潜在问题 |
|------|----------|----------|
| 仓库被删除（执行中） | 后续 API 调用失败 | 错误信息可能不够友好 |
| 分支已存在 | 创建失败 | 没有自动清理或复用逻辑 |
| Secret 已存在 | 覆盖更新 | 用户可能不知情 |
| 文件已存在（SHA 匹配） | 更新文件 | 可能覆盖用户自定义内容 |
| 用户无浏览器 | `openBrowser` 返回 false | 没有提供手动 URL |
| 网络中断 | 命令失败 | 没有断点续传 |
| OAuth Token 过期 | 工作流运行失败 | 安装时无法验证 Token 有效性 |

### 6.3 改进建议

#### 6.3.1 功能增强

1. **事务性回滚**
   ```typescript
   // 建议: 添加 cleanup 函数
   async function cleanupOnFailure(
     repoName: string,
     branchName: string | null,
     createdFiles: string[]
   ): Promise<void> {
     // 删除已创建的分支和文件
   }
   ```

2. **幂等性支持**
   - 支持重复运行，检测已存在的配置
   - 提供 "更新" 而非 "创建" 模式

3. **预览模式**
   ```typescript
   interface SetupPreview {
     branchName: string;
     files: Array<{ path: string; content: string }>;
     secretName: string;
     prUrl: string;
   }
   
   export async function previewSetup(...): Promise<SetupPreview>;
   ```

4. **更好的错误恢复**
   - 区分可恢复和不可恢复错误
   - 提供具体的修复命令

#### 6.3.2 代码改进

1. **类型安全**
   ```typescript
   // 建议: 定义更严格的类型
type RepoName = `${string}/${string}`;
  
   function parseRepoName(input: string): RepoName | null {
     const match = input.match(/^([^/]+)\/([^/]+)$/);
     return match ? `${match[1]}/${match[2]}` as RepoName : null;
   }
   ```

2. **常量提取**
   ```typescript
   // 建议: 提取魔法字符串
   const GITHUB_API = {
     REPO: (name: string) => `repos/${name}`,
     REF: (name: string, branch: string) => `repos/${name}/git/ref/heads/${branch}`,
     REFS: (name: string) => `repos/${name}/git/refs`,
     CONTENTS: (name: string, path: string) => `repos/${name}/contents/${path}`,
   };
   ```

3. **测试覆盖**
   - 当前缺乏单元测试
   - 建议添加:
     - `createWorkflowFile` 的 mock 测试
     - 错误处理路径测试
     - Secret 名称替换逻辑测试

#### 6.3.3 用户体验

1. **进度细化**
   - 当前使用简单的 `updateProgress()` 回调
   - 建议添加详细进度信息
   ```typescript
   interface ProgressUpdate {
     step: 'validating' | 'creating-branch' | 'creating-files' | 'setting-secrets' | 'opening-pr';
     detail?: string;
     percent: number;
   }
   ```

2. **PR 模板自定义**
   - 支持用户自定义 PR 标题和描述
   - 支持添加额外的上下文信息

3. **安装后验证**
   - 验证 Secret 是否正确设置
   - 验证工作流文件语法
   - 提供测试触发方式

---

## 7. 附录

### 7.1 调用链

```
install-github-app.tsx
    ↓ 调用
setupGitHubActions(repoName, apiKey, secretName, updateProgress, skipWorkflow, selectedWorkflows, authType, context)
    ↓
├─> execFileNoThrow('gh', ['api', `repos/${repoName}`])
├─> execFileNoThrow('gh', ['api', `repos/${repoName}`, '--jq', '.default_branch'])
├─> execFileNoThrow('gh', ['api', `repos/${repoName}/git/ref/heads/${defaultBranch}`])
├─> execFileNoThrow('gh', ['api', '--method', 'POST', `repos/${repoName}/git/refs`, ...])
├─> createWorkflowFile() [每个选中的工作流]
│   ├─> execFileNoThrow('gh', ['api', `repos/${repoName}/contents/${path}`])
│   └─> execFileNoThrow('gh', ['api', '--method', 'PUT', `repos/${repoName}/contents/${path}`, ...])
├─> execFileNoThrow('gh', ['secret', 'set', secretName, '--body', apiKeyOrOAuthToken, '--repo', repoName])
├─> openBrowser(compareUrl)
├─> logEvent('tengu_setup_github_actions_completed', ...)
└─> saveGlobalConfig(...)
```

### 7.2 配置字段

```typescript
// GlobalConfig 中的相关字段
interface GlobalConfig {
  githubActionSetupCount?: number;  // 安装次数统计
  // ... 其他字段
}
```

### 7.3 相关文档

- GitHub Actions 工作流语法: https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions
- GitHub REST API - Contents: https://docs.github.com/en/rest/repos/contents#create-or-update-file-contents
- GitHub REST API - Git References: https://docs.github.com/en/rest/git/refs#create-a-reference
- GitHub CLI 文档: https://cli.github.com/manual/
- Claude Code Action: https://github.com/anthropics/claude-code-action
