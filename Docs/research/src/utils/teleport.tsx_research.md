# Research Document: src/utils/teleport.tsx

## 1. 场景与职责

`src/utils/teleport.tsx` 是 Claude Code CLI 中实现**远程会话传输（Teleport）**功能的核心模块。该功能允许用户在本地 Claude Code 实例与云端 Claude.ai 远程会话之间进行双向迁移：

### 1.1 核心场景

1. **Teleport Resume（从云端恢复到本地）**
   - 用户通过 `claude --teleport <session-id>` 命令将云端会话拉取到本地继续执行
   - 自动同步会话历史、分支状态和工作目录
   - 适用于跨设备协作或从浏览器版 Claude Code 迁移到 CLI

2. **Teleport to Remote（从本地推送到云端）**
   - 通过 `/remote` 命令或 Agent 工具将当前本地会话迁移到云端执行
   - 创建新的云端会话，上传 Git 仓库状态（通过 GitHub 或 Bundle 模式）
   - 适用于长时间运行的任务、需要云端计算资源的场景

3. **会话轮询与同步**
   - 对已创建的远程会话进行事件轮询（`pollRemoteSessionEvents`）
   - 支持实时获取远程会话的最新状态和消息

### 1.2 架构定位

```
┌─────────────────────────────────────────────────────────────────┐
│                        Claude Code CLI                          │
├─────────────────────────────────────────────────────────────────┤
│  ┌──────────────┐      ┌──────────────┐      ┌──────────────┐  │
│  │   CLI Entry  │──────│  main.tsx    │──────│  teleport.tsx │  │
│  │   (teleport) │      │              │      │  (this file) │  │
│  └──────────────┘      └──────────────┘      └──────┬───────┘  │
│                                                     │           │
│  ┌──────────────────────────────────────────────────┘           │
│  │                                                              │
│  ▼                                                              │
│  ┌──────────────┐      ┌──────────────┐      ┌──────────────┐  │
│  │  Teleport    │      │  Sessions    │      │   GitHub     │  │
│  │  Progress UI │      │  API Client  │      │   App Check  │  │
│  └──────────────┘      └──────────────┘      └──────────────┘  │
│                                                              │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌───────────────────┐
                    │   Claude.ai API   │
                    │  /v1/sessions     │
                    │  /v1/session_ingress│
                    └───────────────────┘
```

---

## 2. 功能点目的

### 2.1 主要导出功能

| 功能 | 函数名 | 用途 |
|------|--------|------|
| 恢复远程会话 | `teleportResumeCodeSession()` | 从云端会话 ID 拉取历史记录并恢复本地会话 |
| 创建远程会话 | `teleportToRemote()` | 创建新的云端会话，支持 GitHub/Git Bundle 两种源码模式 |
| 带错误处理的远程创建 | `teleportToRemoteWithErrorHandling()` | 包装 `teleportToRemote`，添加前置条件检查和 UI 反馈 |
| 带进度 UI 的恢复 | `teleportWithProgress()` | 在 `TeleportProgress.tsx` 中实现，显示恢复进度 |
| 分支检出 | `checkOutTeleportedSessionBranch()` | 自动切换到会话关联的 Git 分支 |
| 消息处理 | `processMessagesForTeleportResume()` | 处理恢复后的消息，添加 Teleport 提示 |
| 仓库验证 | `validateSessionRepository()` | 验证当前仓库与会话目标仓库是否匹配 |
| 会话归档 | `archiveRemoteSession()` | 将远程会话标记为归档状态 |
| 事件轮询 | `pollRemoteSessionEvents()` | 轮询远程会话的新事件 |

### 2.2 前置条件检查

- **Git 状态检查** (`validateGitState`): 确保工作目录干净（无未提交更改）
- **登录状态检查** (`checkNeedsClaudeAiLogin`): 验证用户已登录 Claude.ai
- **GitHub App 安装检查** (`checkGithubAppInstalled`): 验证仓库已安装 GitHub App

---

## 3. 具体技术实现

### 3.1 关键流程

#### 3.1.1 Teleport Resume 流程

```typescript
// 简化流程图
async function teleportResumeCodeSession(sessionId: string, onProgress?) {
  // 1. 权限检查
  if (!isPolicyAllowed('allow_remote_sessions')) throw Error;
  
  // 2. 获取访问令牌
  const accessToken = getClaudeAIOAuthTokens()?.accessToken;
  const orgUUID = await getOrganizationUUID();
  
  // 3. 验证仓库匹配
  onProgress?.('validating');
  const sessionData = await fetchSession(sessionId);
  const repoValidation = await validateSessionRepository(sessionData);
  // 处理各种验证结果：match/mismatch/not_in_repo/no_repo_required/error
  
  // 4. 获取会话日志
  return await teleportFromSessionsAPI(sessionId, orgUUID, accessToken, onProgress, sessionData);
}
```

#### 3.1.2 Teleport to Remote 流程

```typescript
async function teleportToRemote(options) {
  // 1. 认证检查
  await checkAndRefreshOAuthTokenIfNeeded();
  
  // 2. 源码选择策略（Source Selection Ladder）
  // Priority 1: GitHub Clone (if preflight passes)
  // Priority 2: Git Bundle (if CCR_ENABLE_BUNDLE gate is on)
  // Priority 3: Empty Sandbox (no repo)
  
  // 3. 生成会话标题和分支名（使用 Claude Haiku）
  const { title, branchName } = await generateTitleAndBranch(description, signal);
  
  // 4. GitHub 预检（避免 50% 用户安装后无法克隆的问题）
  const ghViable = await checkGithubAppInstalled(repoInfo.owner, repoInfo.name);
  
  // 5. 如预检失败，尝试 Bundle 模式
  if (!ghViable && bundleSeedGateOn) {
    const bundle = await createAndUploadGitBundle({...});
    seedBundleFileId = bundle.fileId;
  }
  
  // 6. 获取环境列表并选择
  const environments = await fetchEnvironments();
  const selectedEnvironment = selectEnvironment(environments, settings);
  
  // 7. 创建会话 API 调用
  const response = await axios.post(`${BASE_API_URL}/v1/sessions`, {
    title,
    events: initialMessage ? [userEvent] : [],
    session_context: {
      sources: gitSource ? [gitSource] : [],
      seed_bundle_file_id: seedBundleFileId,
      outcomes: gitOutcome ? [gitOutcome] : [],
    },
    environment_id: selectedEnvironment.environment_id,
  });
  
  return { id: sessionData.id, title: sessionData.title };
}
```

### 3.2 数据结构

#### 3.2.1 核心类型定义

```typescript
// 远程会话响应
export type TeleportRemoteResponse = {
  log: Message[];
  branch?: string;
};

// Teleport 结果
export type TeleportResult = {
  messages: Message[];
  branchName: string;
};

// 进度步骤
export type TeleportProgressStep = 
  | 'validating' 
  | 'fetching_logs' 
  | 'fetching_branch' 
  | 'checking_out' 
  | 'done';

// 仓库验证结果
export type RepoValidationResult = {
  status: 'match' | 'mismatch' | 'not_in_repo' | 'no_repo_required' | 'error';
  sessionRepo?: string;
  currentRepo?: string | null;
  sessionHost?: string;
  currentHost?: string;
  errorMessage?: string;
};
```

#### 3.2.2 Sessions API 类型（来自 api.ts）

```typescript
export type SessionResource = {
  type: 'session';
  id: string;
  title: string | null;
  session_status: SessionStatus;
  environment_id: string;
  created_at: string;
  updated_at: string;
  session_context: SessionContext;
};

export type GitSource = {
  type: 'git_repository';
  url: string;
  revision?: string | null;
  allow_unrestricted_git_push?: boolean;
};

export type GitRepositoryOutcome = {
  type: 'git_repository';
  git_info: {
    type: 'github';
    repo: string;
    branches: string[];
  };
};
```

### 3.3 协议与 API

#### 3.3.1 使用的 API 端点

| 端点 | 方法 | 用途 |
|------|------|------|
| `/v1/sessions` | GET | 获取会话列表 |
| `/v1/sessions` | POST | 创建新会话 |
| `/v1/sessions/{id}` | GET | 获取单个会话 |
| `/v1/sessions/{id}` | PATCH | 更新会话标题 |
| `/v1/sessions/{id}/events` | GET/POST | 获取/发送会话事件 |
| `/v1/sessions/{id}/archive` | POST | 归档会话 |
| `/v1/code/sessions/{id}/teleport-events` | GET | 获取 Teleport 事件（v2 API） |
| `/v1/session_ingress/session/{id}` | GET | 获取会话日志（旧版） |
| `/v1/environment_providers` | GET | 获取可用环境列表 |
| `/api/oauth/organizations/{org}/code/repos/{owner}/{repo}` | GET | 检查 GitHub App 安装状态 |

#### 3.3.2 请求头规范

```typescript
const headers = {
  'Authorization': `Bearer ${accessToken}`,
  'Content-Type': 'application/json',
  'anthropic-version': '2023-06-01',
  'anthropic-beta': 'ccr-byoc-2025-07-29',  // BYOC (Bring Your Own Code) Beta
  'x-organization-uuid': orgUUID,
};
```

### 3.4 Git Bundle 技术细节

Bundle 模式用于在不依赖 GitHub 的情况下上传本地仓库状态：

```
┌─────────────────────────────────────────────────────────────┐
│                    Git Bundle Flow                          │
├─────────────────────────────────────────────────────────────┤
│  1. git stash create → refs/seed/stash (保存 WIP)           │
│  2. git bundle create --all (打包所有 refs)                 │
│     └── 如果太大 → 降级到 HEAD → 降级到 squashed-root       │
│  3. Upload to /v1/files → 获取 file_id                      │
│  4. Cleanup refs/seed/stash                                 │
│  5. 在 SessionContext 中设置 seed_bundle_file_id            │
└─────────────────────────────────────────────────────────────┘
```

Bundle 大小限制：默认 100MB（可通过 `tengu_ccr_bundle_max_bytes` GrowthBook 特性调整）

---

## 4. 关键代码路径与文件引用

### 4.1 调用关系图

```
teleport.tsx
├── 被调用方（Callers）
│   ├── main.tsx                    # CLI 入口，处理 --teleport 参数
│   ├── cli/print.ts                # 非交互式 CLI 打印
│   ├── commands/review/reviewRemote.ts  # 远程代码审查
│   ├── commands/ultraplan.tsx      # Ultrareview 功能
│   ├── tools/AgentTool/AgentTool.tsx   # Agent 远程执行
│   ├── components/TeleportProgress.tsx   # 带 UI 的恢复
│   ├── hooks/useTeleportResume.tsx     # React Hook 封装
│   └── components/tasks/RemoteSessionDetailDialog.tsx
│
├── 被调用方（Callees）
│   ├── utils/teleport/api.ts       # Sessions API 客户端
│   ├── utils/teleport/environments.ts  # 环境管理
│   ├── utils/teleport/gitBundle.ts     # Git Bundle 创建
│   ├── services/api/sessionIngress.ts  # 会话日志获取
│   ├── utils/background/remote/preconditions.ts  # 前置条件
│   ├── utils/conversationRecovery.ts   # 消息反序列化
│   ├── utils/detectRepository.ts       # 仓库检测
│   ├── utils/git.ts                    # Git 操作
│   └── services/api/claude.ts          # Claude Haiku 调用
│
└── 相关 UI 组件
    ├── components/TeleportError.tsx      # 错误处理 UI
    ├── components/TeleportStash.tsx      # Git Stash UI
    ├── components/TeleportProgress.tsx   # 进度显示
    ├── components/TeleportRepoMismatchDialog.tsx
    └── components/TeleportResumeWrapper.tsx
```

### 4.2 关键文件路径

| 文件 | 职责 |
|------|------|
| `src/utils/teleport.tsx` | 核心 Teleport 逻辑（本文件） |
| `src/utils/teleport/api.ts` | Sessions API 客户端，含重试逻辑 |
| `src/utils/teleport/environments.ts` | 远程环境列表获取 |
| `src/utils/teleport/gitBundle.ts` | Git Bundle 创建与上传 |
| `src/services/api/sessionIngress.ts` | 会话日志获取（v1/v2 API） |
| `src/utils/background/remote/preconditions.ts` | 前置条件检查 |
| `src/utils/conversationRecovery.ts` | 会话恢复消息处理 |
| `src/components/TeleportError.tsx` | Teleport 错误 UI |
| `src/components/TeleportProgress.tsx` | Teleport 进度 UI |
| `src/components/TeleportStash.tsx` | Git Stash 交互 UI |
| `src/hooks/useTeleportResume.tsx` | Teleport Resume React Hook |

---

## 5. 依赖与外部交互

### 5.1 外部依赖

```typescript
// 核心依赖
import axios from 'axios';                    // HTTP 客户端
import chalk from 'chalk';                    // 终端颜色
import { randomUUID } from 'crypto';          // UUID 生成
import React from 'react';                    // UI 渲染
import { z } from 'zod/v4';                   // 运行时类型校验

// 内部模块
import { queryHaiku } from '../services/api/claude.js';           // 标题生成
import { getSessionLogsViaOAuth, getTeleportEvents } from '../services/api/sessionIngress.js';
import { checkGithubAppInstalled } from './background/remote/preconditions.js';
import { createAndUploadGitBundle } from './teleport/gitBundle.js';
import { fetchSession, getOAuthHeaders } from './teleport/api.js';
import { fetchEnvironments } from './teleport/environments.js';
```

### 5.2 服务依赖

| 服务 | 用途 |
|------|------|
| Claude.ai OAuth | 身份验证和访问令牌管理 |
| Claude.ai Sessions API | 远程会话 CRUD 操作 |
| Claude.ai Files API | Git Bundle 上传 |
| GitHub API | 仓库检测和 App 安装状态检查 |
| GrowthBook | 特性开关（Bundle 模式、大小限制等） |

### 5.3 环境变量

| 变量 | 用途 |
|------|------|
| `CCR_FORCE_BUNDLE=1` | 强制使用 Bundle 模式（跳过 GitHub 预检） |
| `CCR_ENABLE_BUNDLE=1` | 启用 Bundle 模式作为回退 |
| `CLAUDE_AFTER_LAST_COMPACT` | 控制会话日志获取范围 |

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 网络与可靠性风险

1. **API 依赖风险**
   - Sessions API 404 时可能回退到 session-ingress，但后者即将下线
   - 网络中断可能导致 Bundle 上传失败（已添加重试逻辑）

2. **Git 状态风险**
   - 工作目录不干净时 Teleport 会失败（已提供 Stash UI 解决）
   - 分支切换冲突可能导致恢复失败

#### 6.1.2 安全风险

1. **Bundle 模式安全**
   - Bundle 包含完整的 Git 历史（包括未提交的 WIP）
   - 上传到 Claude.ai Files API，需要确保传输加密和存储安全

2. **仓库验证绕过风险**
   - 当前仓库与会话目标仓库不匹配时可能误操作
   - 已实现 `validateSessionRepository` 进行严格校验

### 6.2 边界条件

| 边界条件 | 处理方式 |
|----------|----------|
| 空仓库（无提交） | Bundle 创建失败，返回 `empty_repo` 错误 |
| 超大仓库（>100MB） | Bundle 降级链：--all → HEAD → squashed-root |
| 无 GitHub App 安装 | 预检失败，自动回退到 Bundle 模式 |
| 无可用环境 | 返回 null，记录错误日志 |
| 会话不存在（404） | 抛出 `TeleportOperationError`，提示用户检查 /status |
| 认证过期（401） | 提示用户运行 /login |

### 6.3 改进建议

#### 6.3.1 短期改进

1. **增强错误提示**
   - 当前 Bundle 失败时提示 "Please setup GitHub"，可添加更具体的诊断链接
   - 建议添加 `claude doctor --teleport` 诊断命令

2. **优化 Bundle 大小限制**
   - 当前 100MB 限制可能不适用于大型 monorepo
   - 建议支持分层 Bundle（只上传变更的文件）

3. **改进进度反馈**
   - Bundle 创建和上传过程可能耗时较长，当前进度步骤较粗
   - 建议添加 "uploading_bundle" 和 "creating_session" 子步骤

#### 6.3.2 中长期改进

1. **增量同步支持**
   - 当前 Teleport Resume 拉取完整会话历史，对于长会话效率低
   - 建议支持增量同步，只拉取上次同步后的新事件

2. **多仓库支持**
   - 当前只支持单仓库会话，多仓库工作区（如 monorepo 子包）支持有限
   - 建议扩展 `SessionContext.sources` 支持多仓库

3. **离线模式支持**
   - 当前完全依赖网络，建议支持离线缓存和延迟同步

4. **代码结构优化**
   - 当前 `teleport.tsx` 超过 1200 行，职责较多
   - 建议拆分为：
     - `teleport/resume.ts` - 恢复逻辑
     - `teleport/create.ts` - 创建远程会话
     - `teleport/validation.ts` - 验证逻辑
     - `teleport/polling.ts` - 事件轮询

### 6.4 测试建议

1. **单元测试覆盖**
   - `validateSessionRepository` 的各种边界情况
   - `generateTitleAndBranch` 的降级逻辑
   - Bundle 降级链（--all → HEAD → squashed）

2. **集成测试**
   - 完整的 Teleport Resume 流程（需要 mock Sessions API）
   - Bundle 创建和上传流程
   - GitHub App 预检失败后的回退逻辑

3. **E2E 测试**
   - 跨设备会话迁移
   - 大仓库（接近 100MB）的 Bundle 模式
   - 网络中断后的重试行为

---

## 7. 附录

### 7.1 术语表

| 术语 | 解释 |
|------|------|
| Teleport | Claude Code 的远程会话传输功能 |
| BYOC | Bring Your Own Code，用户自带代码仓库 |
| CCR | Claude Code Remote，云端执行环境 |
| Bundle | Git bundle 文件，用于打包仓库状态 |
| Session Ingress | 旧版会话日志 API（将被 v2 替代） |
| GrowthBook | 特性开关和实验平台 |

### 7.2 相关文档

- `src/utils/teleport/api.ts` - API 客户端实现
- `src/utils/teleport/gitBundle.ts` - Bundle 创建细节
- `src/services/api/sessionIngress.ts` - 会话日志获取
- `src/components/TeleportError.tsx` - 错误处理 UI
- `src/components/TeleportProgress.tsx` - 进度 UI

---

*文档生成时间: 2026-04-01*  
*研究范围: src/utils/teleport.tsx 及其直接依赖*  
*版本: 基于当前代码库 HEAD*
