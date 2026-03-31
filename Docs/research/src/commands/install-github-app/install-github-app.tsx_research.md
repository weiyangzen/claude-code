# install-github-app.tsx 深度研究文档

## 文件基本信息

- **路径**: `src/commands/install-github-app/install-github-app.tsx`
- **大小**: ~87KB (587 行)
- **类型**: React JSX 组件 + CLI 命令入口
- **职责**: GitHub App 安装流程的交互式向导主组件

---

## 1. 场景与职责

### 1.1 核心场景

该文件实现了一个**交互式 CLI 向导**，用于帮助用户在 GitHub 仓库中安装和配置 Claude Code GitHub Actions 工作流。这是 `/install-github-app` 命令的主要实现。

### 1.2 用户旅程

```
用户运行 /install-github-app
    ↓
检查 GitHub CLI (gh) 安装和认证状态
    ↓
选择目标仓库（当前仓库或手动输入）
    ↓
验证仓库权限和访问
    ↓
引导安装 Claude GitHub App
    ↓
选择要安装的工作流类型
    ↓
处理 API Key（现有/OAuth/新输入）
    ↓
创建分支、工作流文件、设置 Secrets
    ↓
打开浏览器创建 PR
```

### 1.3 主要职责

1. **状态管理**: 使用 React useState 管理多步骤向导的状态机
2. **GitHub CLI 集成**: 检测 `gh` CLI 的安装、认证和权限状态
3. **仓库验证**: 验证用户输入的仓库格式、权限和可访问性
4. **工作流选择**: 支持选择多种 Claude 工作流类型
5. **认证处理**: 支持 API Key 和 OAuth Token 两种认证方式
6. **错误处理**: 提供详细的错误信息和修复指导

---

## 2. 功能点目的

### 2.1 步骤状态机 (Step State Machine)

| 步骤 | 标识符 | 目的 |
|------|--------|------|
| 检查 GitHub CLI | `check-gh` | 验证 gh CLI 安装和认证状态 |
| 警告展示 | `warnings` | 显示检测到的警告信息（如 gh 未安装） |
| 选择仓库 | `choose-repo` | 让用户选择或输入目标仓库 |
| 安装 App | `install-app` | 引导用户打开浏览器安装 Claude GitHub App |
| 检查现有工作流 | `check-existing-workflow` | 检测是否已存在 Claude 工作流文件 |
| 选择工作流 | `select-workflows` | 让用户选择要安装的工作流类型 |
| 检查现有 Secret | `check-existing-secret` | 检测是否已存在 ANTHROPIC_API_KEY secret |
| API Key 输入 | `api-key` | 输入或选择 API Key 来源 |
| OAuth 流程 | `oauth-flow` | 处理 OAuth 认证流程 |
| 创建中 | `creating` | 执行 GitHub Actions 设置 |
| 成功 | `success` | 显示成功信息 |
| 错误 | `error` | 显示错误信息和修复建议 |

### 2.2 支持的认证方式

1. **现有 API Key**: 使用本地已配置的 Claude Code API Key
2. **OAuth Token**: 通过 Claude 订阅创建长期有效的 OAuth Token
3. **新 API Key**: 用户手动输入新的 Anthropic API Key

### 2.3 支持的工作流类型

- **`claude`**: 基础 Claude PR Assistant 工作流（通过 `@claude` 触发）
- **`claude-review`**: 自动代码审查工作流（PR 创建时自动触发）

---

## 3. 具体技术实现

### 3.1 核心数据结构

```typescript
// 状态定义 (从 types.js 导入)
interface State {
  step: Step;                          // 当前步骤
  selectedRepoName: string;            // 选中的仓库名 (owner/repo)
  currentRepo: string;                 // 当前 git 仓库
  useCurrentRepo: boolean;             // 是否使用当前仓库
  apiKeyOrOAuthToken: string;          // API Key 或 OAuth Token
  useExistingKey: boolean;             // 是否使用现有 Key
  currentWorkflowInstallStep: number;  // 当前工作流安装进度
  warnings: Warning[];                 // 警告信息列表
  secretExists: boolean;               // 是否已存在 secret
  secretName: string;                  // Secret 名称
  useExistingSecret: boolean;          // 是否使用现有 secret
  workflowExists: boolean;             // 是否已存在工作流文件
  selectedWorkflows: Workflow[];       // 选中的工作流
  selectedApiKeyOption: 'existing' | 'new' | 'oauth';
  authType: 'api_key' | 'oauth_token';
  workflowAction?: 'update' | 'skip' | 'exit';  // 现有工作流处理动作
  error?: string;                      // 错误信息
  errorReason?: string;                // 错误原因
  errorInstructions?: string[];        // 错误修复指导
}

interface Warning {
  title: string;                       // 警告标题
  message: string;                     // 警告描述
  instructions: string[];              // 修复指导
}

type Workflow = 'claude' | 'claude-review';
type Step = 'check-gh' | 'warnings' | 'choose-repo' | 'install-app' | 
            'check-existing-workflow' | 'select-workflows' | 'check-existing-secret' | 
            'api-key' | 'oauth-flow' | 'creating' | 'success' | 'error';
```

### 3.2 初始化状态

```typescript
const INITIAL_STATE: State = {
  step: 'check-gh',
  selectedRepoName: '',
  currentRepo: '',
  useCurrentRepo: false,
  apiKeyOrOAuthToken: '',
  useExistingKey: true,
  currentWorkflowInstallStep: 0,
  warnings: [],
  secretExists: false,
  secretName: 'ANTHROPIC_API_KEY',
  useExistingSecret: true,
  workflowExists: false,
  selectedWorkflows: ['claude', 'claude-review'],
  selectedApiKeyOption: 'existing',
  authType: 'api_key'
};
```

### 3.3 关键流程实现

#### 3.3.1 GitHub CLI 检查 (`checkGitHubCLI`)

```typescript
const checkGitHubCLI = useCallback(async () => {
  const warnings: Warning[] = [];
  
  // 1. 检查 gh 是否安装
  const ghVersionResult = await execa('gh --version', { shell: true, reject: false });
  if (ghVersionResult.exitCode !== 0) {
    warnings.push({
      title: 'GitHub CLI not found',
      message: 'GitHub CLI (gh) does not appear to be installed or accessible.',
      instructions: ['Install GitHub CLI from https://cli.github.com/', ...]
    });
  }
  
  // 2. 检查认证状态
  const authResult = await execa('gh auth status -a', { shell: true, reject: false });
  if (authResult.exitCode !== 0) {
    warnings.push({...});  // 未认证警告
  } else {
    // 3. 检查必要权限 (repo, workflow)
    const tokenScopesMatch = authResult.stdout.match(/Token scopes:.*$/m);
    if (tokenScopesMatch) {
      const scopes = tokenScopesMatch[0];
      const missingScopes: string[] = [];
      if (!scopes.includes('repo')) missingScopes.push('repo');
      if (!scopes.includes('workflow')) missingScopes.push('workflow');
      if (missingScopes.length > 0) {
        // 直接跳转到错误步骤
        setState(prev => ({ ...prev, step: 'error', error: ..., errorInstructions: [...] }));
        return;
      }
    }
  }
  
  // 4. 获取当前 git 仓库
  const currentRepo = (await getGithubRepo()) ?? '';
  
  // 更新状态
  setState(prev => ({
    ...prev,
    warnings,
    currentRepo,
    selectedRepoName: currentRepo,
    useCurrentRepo: !!currentRepo,
    step: warnings.length > 0 ? 'warnings' : 'choose-repo'
  }));
}, []);
```

#### 3.3.2 仓库权限检查 (`checkRepositoryPermissions`)

```typescript
async function checkRepositoryPermissions(repoName: string): Promise<{
  hasAccess: boolean;
  error?: string;
}> {
  try {
    // 使用 gh api 检查仓库权限
    const result = await execFileNoThrow('gh', [
      'api', 
      `repos/${repoName}`, 
      '--jq', '.permissions.admin'
    ]);
    
    if (result.code === 0) {
      const hasAdmin = result.stdout.trim() === 'true';
      return { hasAccess: hasAdmin };
    }
    
    if (result.stderr.includes('404') || result.stderr.includes('Not Found')) {
      return { hasAccess: false, error: 'repository_not_found' };
    }
    
    return { hasAccess: false };
  } catch {
    return { hasAccess: false };
  }
}
```

#### 3.3.3 提交处理 (`handleSubmit`)

这是一个大型条件分支函数，根据当前步骤执行不同操作：

```typescript
const handleSubmit = async () => {
  if (state.step === 'warnings') {
    // 记录分析事件，进入安装步骤
    logEvent('tengu_install_github_app_step_completed', { step: 'warnings' });
    setState(prev => ({ ...prev, step: 'install-app' }));
    setTimeout(openGitHubAppInstallation, 0);
  } 
  else if (state.step === 'choose-repo') {
    // 验证仓库格式和权限
    // 解析 GitHub URL 格式
    // 检查管理员权限
    // 检查现有工作流文件
    // 进入下一步
  }
  else if (state.step === 'install-app') {
    // 根据是否存在工作流文件决定下一步
  }
  // ... 更多步骤处理
};
```

#### 3.3.4 GitHub Actions 设置执行

```typescript
const runSetupGitHubActions = useCallback(async (
  apiKeyOrOAuthToken: string | null, 
  secretName: string
) => {
  setState(prev => ({ ...prev, step: 'creating', currentWorkflowInstallStep: 0 }));
  
  try {
    await setupGitHubActions(
      state.selectedRepoName,
      apiKeyOrOAuthToken,
      secretName,
      () => { /* 进度更新回调 */ },
      state.workflowAction === 'skip',
      state.selectedWorkflows,
      state.authType,
      { useCurrentRepo: state.useCurrentRepo, workflowExists: state.workflowExists, secretExists: state.secretExists }
    );
    
    logEvent('tengu_install_github_app_step_completed', { step: 'creating' });
    setState(prev => ({ ...prev, step: 'success' }));
  } catch (error) {
    // 错误处理，区分工作流文件已存在的情况
    const errorMessage = error instanceof Error ? error.message : 'Failed to set up GitHub Actions';
    if (errorMessage.includes('workflow file already exists')) {
      // 特定错误处理
    } else {
      // 通用错误处理
    }
  }
}, [/* deps */]);
```

### 3.4 组件渲染

根据 `state.step` 使用 switch 语句渲染不同步骤组件：

```typescript
switch (state.step) {
  case 'check-gh':
    return <CheckGitHubStep />;
  case 'warnings':
    return <WarningsStep warnings={state.warnings} onContinue={handleSubmit} />;
  case 'choose-repo':
    return <ChooseRepoStep currentRepo={state.currentRepo} ... />;
  case 'install-app':
    return <InstallAppStep repoUrl={state.selectedRepoName} onSubmit={handleSubmit} />;
  case 'check-existing-workflow':
    return <ExistingWorkflowStep repoName={state.selectedRepoName} onSelectAction={handleWorkflowAction} />;
  case 'check-existing-secret':
    return <CheckExistingSecretStep ... />;
  case 'api-key':
    return <ApiKeyStep existingApiKey={existingApiKey} ... />;
  case 'creating':
    return <CreatingStep ... />;
  case 'success':
    return <Box onKeyDown={handleDismissKeyDown}><SuccessStep ... /></Box>;
  case 'error':
    return <Box onKeyDown={handleDismissKeyDown}><ErrorStep ... /></Box>;
  case 'select-workflows':
    return <WorkflowMultiselectDialog ... />;
  case 'oauth-flow':
    return <OAuthFlowStep onSuccess={handleOAuthSuccess} onCancel={handleOAuthCancel} />;
}
```

---

## 4. 关键代码路径与文件引用

### 4.1 导入依赖

```typescript
// 外部库
import { execa } from 'execa';
import React, { useCallback, useState } from 'react';

// 内部组件
import { WorkflowMultiselectDialog } from '../../components/WorkflowMultiselectDialog.js';
import { ApiKeyStep } from './ApiKeyStep.js';
import { CheckExistingSecretStep } from './CheckExistingSecretStep.js';
import { CheckGitHubStep } from './CheckGitHubStep.js';
import { ChooseRepoStep } from './ChooseRepoStep.js';
import { CreatingStep } from './CreatingStep.js';
import { ErrorStep } from './ErrorStep.js';
import { ExistingWorkflowStep } from './ExistingWorkflowStep.js';
import { InstallAppStep } from './InstallAppStep.js';
import { OAuthFlowStep } from './OAuthFlowStep.js';
import { SuccessStep } from './SuccessStep.js';
import { WarningsStep } from './WarningsStep.js';

// 工具函数
import { getAnthropicApiKey, isAnthropicAuthEnabled } from '../../utils/auth.js';
import { openBrowser } from '../../utils/browser.js';
import { execFileNoThrow } from '../../utils/execFileNoThrow.js';
import { getGithubRepo } from '../../utils/git.js';
import { plural } from '../../utils/stringUtils.js';

// 其他
import { setupGitHubActions } from './setupGitHubActions.js';
import type { State, Warning, Workflow } from './types.js';
```

### 4.2 相关文件路径

| 文件 | 路径 | 用途 |
|------|------|------|
| 主组件 | `src/commands/install-github-app/install-github-app.tsx` | 向导主逻辑 |
| 设置逻辑 | `src/commands/install-github-app/setupGitHubActions.ts` | GitHub Actions 实际设置 |
| 类型定义 | `src/commands/install-github-app/types.js` (运行时) | State/Warning/Workflow 类型 |
| 命令入口 | `src/commands/install-github-app/index.ts` | 命令注册 |
| 工作流选择 | `src/components/WorkflowMultiselectDialog.tsx` | 工作流多选对话框 |
| 步骤组件 | `src/commands/install-github-app/*Step.tsx` | 各步骤 UI 组件 |
| 常量 | `src/constants/github-app.ts` | 工作流内容、PR 模板 |
| 认证工具 | `src/utils/auth.ts` | API Key/OAuth 获取 |
| 浏览器 | `src/utils/browser.ts` | 打开浏览器 |
| Git 工具 | `src/utils/git.ts` | 获取 GitHub 仓库信息 |
| 执行工具 | `src/utils/execFileNoThrow.ts` | 安全执行 shell 命令 |
| 字符串 | `src/utils/stringUtils.ts` | plural 等工具函数 |

---

## 5. 依赖与外部交互

### 5.1 外部依赖

| 依赖 | 用途 |
|------|------|
| `execa` | 执行 shell 命令，跨平台兼容 |
| `react` | UI 组件框架 |

### 5.2 系统依赖

| 工具 | 用途 | 检查方式 |
|------|------|----------|
| GitHub CLI (`gh`) | GitHub API 调用、认证管理 | `gh --version` |
| Git | 获取当前仓库信息 | 通过 `src/utils/git.ts` |
| 浏览器 | 打开 GitHub App 安装页、创建 PR | `open`/`xdg-open`/`rundll32` |

### 5.3 GitHub CLI 命令使用

```bash
# 检查版本
gh --version

# 检查认证状态
gh auth status -a

# 检查仓库权限
gh api repos/{owner}/{repo} --jq '.permissions.admin'

# 检查文件是否存在
gh api repos/{owner}/{repo}/contents/.github/workflows/claude.yml --jq '.sha'

# 列出 secrets
gh secret list --app actions --repo {owner}/{repo}

# 设置 secret
gh secret set {name} --body {value} --repo {owner}/{repo}

# 创建分支
gh api --method POST repos/{owner}/{repo}/git/refs -f ref=refs/heads/{branch} -f sha={sha}

# 创建/更新文件
gh api --method PUT repos/{owner}/{repo}/contents/{path} -f message={msg} -f content={base64} -f branch={branch} -f sha={sha}
```

### 5.4 分析事件

| 事件名 | 触发时机 |
|--------|----------|
| `tengu_install_github_app_started` | 向导启动 |
| `tengu_install_github_app_step_completed` | 各步骤完成 |
| `tengu_install_github_app_error` | 发生错误 |
| `tengu_install_github_app_completed` | 向导完成 |

---

## 6. 风险、边界与改进建议

### 6.1 潜在风险

#### 6.1.1 安全风险

1. **Secret 泄露风险**
   - API Key 在内存中明文存储
   - 通过 `gh secret set` 传递时可能在进程列表中暴露
   - **建议**: 使用更安全的方式传递敏感信息

2. **权限检查不足**
   - 仅检查 `admin` 权限，但某些操作可能需要更细粒度的权限
   - **建议**: 增加更详细的权限检查

3. **URL 验证**
   - `openBrowser` 调用前 URL 验证有限
   - **建议**: 加强 URL 白名单验证

#### 6.1.2 功能风险

1. **GitHub CLI 依赖**
   - 强依赖 `gh` CLI，用户可能未安装或版本不兼容
   - **缓解**: 有检测逻辑和安装指导

2. **网络依赖**
   - 所有 GitHub 操作都需要网络连接
   - 缺乏离线模式或重试机制

3. **并发问题**
   - 快速连续操作可能导致状态不一致
   - `setState` 异步更新可能导致竞态条件

### 6.2 边界情况

| 场景 | 当前行为 | 潜在问题 |
|------|----------|----------|
| 用户无 gh CLI | 显示警告，继续流程 | 后续步骤会失败 |
| gh 认证过期 | 显示认证错误 | 错误信息可能不够明确 |
| 仓库不存在 | 显示 404 错误 | 错误恢复流程不够友好 |
| 无管理员权限 | 显示权限警告 | 用户可能不理解为什么需要 |
| 工作流文件已存在 | 显示冲突错误 | 没有自动合并/更新选项 |
| 网络中断 | 命令失败 | 没有断点续传或重试 |
| OAuth 取消 | 返回 api-key 步骤 | 状态重置可能不完整 |

### 6.3 改进建议

#### 6.3.1 功能增强

1. **自动重试机制**
   ```typescript
   // 建议添加指数退避重试
   async function retryWithBackoff<T>(
     fn: () => Promise<T>,
     maxRetries: number = 3
   ): Promise<T> { ... }
   ```

2. **更好的错误恢复**
   - 保存用户进度，支持从中断点恢复
   - 提供 "重试" 选项

3. **预览模式**
   - 在实际执行前显示将要执行的操作预览
   - 让用户确认后再创建分支/PR

4. **工作流模板选择**
   - 支持更多预设工作流模板
   - 支持自定义工作流配置

#### 6.3.2 代码改进

1. **状态管理优化**
   - 考虑使用状态机库（如 XState）替代手动 switch
   - 添加状态转换验证

2. **测试覆盖**
   - 当前缺乏单元测试
   - 建议添加各步骤的单元测试和集成测试

3. **类型安全**
   - `types.js` 文件在编译时可能不存在
   - 建议将类型定义移到 `.ts` 文件

#### 6.3.3 用户体验

1. **进度指示**
   - 添加更详细的进度指示器
   - 显示预计剩余时间

2. **帮助文档**
   - 在每个步骤添加内联帮助
   - 提供常见问题的快速解答

3. **撤销功能**
   - 支持撤销已创建的 branch/PR
   - 清理已设置的 secrets

---

## 7. 附录

### 7.1 调用链

```
用户输入 /install-github-app
    ↓
commands/index.ts 路由到 install-github-app
    ↓
install-github-app/index.ts 加载命令
    ↓
install-github-app.tsx call() 函数
    ↓
InstallGitHubApp 组件渲染
    ↓
各 Step 组件根据状态渲染
    ↓
setupGitHubActions.ts 执行实际设置
```

### 7.2 环境变量依赖

| 变量 | 用途 |
|------|------|
| `DISABLE_INSTALL_GITHUB_APP_COMMAND` | 禁用此命令 |
| `GITHUB_ACTION_SETUP_DOCS_URL` | 文档链接（来自 constants） |

### 7.3 相关文档

- GitHub Actions 文档: https://github.com/anthropics/claude-code-action
- 设置指南: https://github.com/anthropics/claude-code-action/blob/main/docs/setup.md
