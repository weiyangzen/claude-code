# 研究文档: src/commands/feedback/feedback.tsx

## 场景与职责

本文件实现 Claude Code 的交互式反馈/bug 报告功能。当用户输入 `/feedback` 或 `/bug` 命令时，系统会渲染一个基于 Ink (React for CLI) 的交互式界面，引导用户完成反馈提交流程。

### 使用场景
1. **Bug 报告**: 用户遇到问题，通过 `/bug 问题描述` 提交详细报告
2. **功能反馈**: 用户通过 `/feedback 建议内容` 提交产品改进建议
3. **GitHub Issue 创建**: 提交后自动生成 GitHub Issue 草稿链接

### 核心职责
1. **交互式 UI 渲染**: 提供多步骤的终端交互界面
2. **数据收集**: 收集用户描述、环境信息、会话记录等
3. **敏感信息脱敏**: 自动清理 API 密钥等敏感信息
4. **反馈提交**: 通过 API 将反馈发送到 Anthropic 服务器
5. **GitHub Issue 集成**: 生成 GitHub Issue 链接便于公开追踪

---

## 功能点目的

### 1. 多步骤交互流程 (Step)

```typescript
type Step = 'userInput' | 'consent' | 'submitting' | 'done';
```

| 步骤 | 说明 |
|------|------|
| `userInput` | 用户输入反馈描述 |
| `consent` | 展示将要提交的数据，请求确认 |
| `submitting` | 提交中状态显示 |
| `done` | 完成状态，显示反馈 ID 和 GitHub 链接选项 |

### 2. 反馈数据结构 (FeedbackData)

```typescript
type FeedbackData = {
  latestAssistantMessageId: string | null;
  message_count: number;
  datetime: string;
  description: string;
  platform: string;
  gitRepo: boolean;
  version: string | null;
  transcript: Message[];
  subagentTranscripts?: { [agentId: string]: Message[] };
  rawTranscriptJsonl?: string;
};
```

收集的信息包括：
- **会话上下文**: 最新消息 ID、消息数量、完整会话记录
- **环境信息**: 平台、终端类型、版本号、Git 状态
- **子代理记录**: 如果有子代理任务，收集其会话记录
- **原始记录**: 会话的原始 JSONL 格式记录

### 3. 敏感信息脱敏 (redactSensitiveInfo)

自动识别并替换以下敏感信息：
- Anthropic API Keys (`sk-ant...`)
- AWS Keys (`AKIA...`, `AWS...`)
- Google Cloud Keys (`AIza...`)
- GCP Service Account 邮箱
- API Key 请求头
- Authorization/Bearer Tokens
- AWS/GCP 环境变量值

---

## 具体技术实现

### 1. 组件入口函数

#### `renderFeedbackComponent`
共享函数，用于渲染 Feedback 组件，被主调用函数和可能的测试代码复用：

```typescript
export function renderFeedbackComponent(
  onDone: (result?: string, options?: { display?: CommandResultDisplay }) => void,
  abortSignal: AbortSignal,
  messages: Message[],
  initialDescription: string = '',
  backgroundTasks: { [taskId: string]: {...} } = {}
): React.ReactNode
```

#### `call`
命令系统的标准调用入口：

```typescript
export async function call(
  onDone: LocalJSXCommandOnDone,
  context: LocalJSXCommandContext,
  args?: string
): Promise<React.ReactNode>
```

### 2. 主组件 Feedback

#### Props 定义
```typescript
type Props = {
  abortSignal: AbortSignal;        // 用于取消提交请求
  messages: Message[];             // 当前会话消息列表
  initialDescription?: string;     // 初始描述（从命令参数传入）
  onDone(result: string, options?): void;  // 完成回调
  backgroundTasks?: {...};         // 后台任务信息
};
```

#### State 管理
```typescript
const [step, setStep] = useState<Step>('userInput');
const [cursorOffset, setCursorOffset] = useState(0);
const [description, setDescription] = useState(initialDescription ?? '');
const [feedbackId, setFeedbackId] = useState<string | null>(null);
const [error, setError] = useState<string | null>(null);
const [envInfo, setEnvInfo] = useState({ isGit: false, gitState: null });
const [title, setTitle] = useState<string | null>(null);
```

### 3. 提交流程 (submitReport)

```typescript
const submitReport = useCallback(async () => {
  setStep('submitting');
  
  // 1. 收集错误日志并脱敏
  const sanitizedErrors = getSanitizedErrorLogs();
  
  // 2. 获取最后一条助手消息 ID
  const lastAssistantMessage = getLastAssistantMessage(messages);
  const lastAssistantMessageId = lastAssistantMessage?.requestId ?? null;
  
  // 3. 并行加载子代理记录和原始记录
  const [diskTranscripts, rawTranscriptJsonl] = await Promise.all([
    loadAllSubagentTranscriptsFromDisk(),
    loadRawTranscriptJsonl()
  ]);
  
  // 4. 合并子代理记录
  const teammateTranscripts = extractTeammateTranscriptsFromTasks(backgroundTasks);
  const subagentTranscripts = { ...diskTranscripts, ...teammateTranscripts };
  
  // 5. 构建报告数据
  const reportData = {
    latestAssistantMessageId: lastAssistantMessageId,
    message_count: messages.length,
    datetime: new Date().toISOString(),
    description,
    platform: env.platform,
    gitRepo: envInfo.isGit,
    terminal: env.terminal,
    version: MACRO.VERSION,
    transcript: normalizeMessagesForAPI(messages),
    errors: sanitizedErrors,
    lastApiRequest: getLastAPIRequest(),
    ...(Object.keys(subagentTranscripts).length > 0 && { subagentTranscripts }),
    ...(rawTranscriptJsonl && { rawTranscriptJsonl })
  };
  
  // 6. 并行提交反馈和生成标题
  const [result, t] = await Promise.all([
    submitFeedback(reportData, abortSignal),
    generateTitle(description, abortSignal)
  ]);
  
  // 7. 处理结果
  if (result.success) {
    setFeedbackId(result.feedbackId);
    logEvent('tengu_bug_report_submitted', {...});
    logEventTo1P('tengu_bug_report_description', {...});
    setStep('done');
  } else {
    setError(result.isZdrOrg ? '...' : 'Could not submit feedback...');
    setStep('userInput');  // 允许重试
  }
}, [description, envInfo.isGit, messages]);
```

### 4. 标题生成 (generateTitle)

使用 Haiku 模型自动生成 GitHub Issue 标题：

```typescript
async function generateTitle(description: string, abortSignal: AbortSignal): Promise<string>
```

系统提示词要求：
- 生成简洁的技术 Issue 标题（最多 80 字符）
- 必须以 `[Bug]` 或 `[Feature Request]` 开头
- 使用恰当的技术术语
- 如果是错误消息，提取关键错误而非完整消息

降级策略：
- 如果 API 调用失败或返回 API 错误前缀，使用 `createFallbackTitle`
- Fallback 使用描述的第一行，截断到 60 字符

### 5. GitHub Issue URL 生成

```typescript
export function createGitHubIssueUrl(
  feedbackId: string,
  title: string,
  description: string,
  errors: Array<{ error?: string; timestamp?: string }>
): string
```

URL 长度限制处理：
- GitHub URL 限制: 7250 字节（实验测定）
- 如果内容超出限制，优先保留描述，截断错误日志
- 使用智能截断避免切断 URL 编码序列 (`%XX`)

目标仓库：
- 内部构建 (`'ant'`): `https://github.com/anthropics/claude-cli-internal/issues`
- 外部构建: `https://github.com/anthropics/claude-code/issues`

### 6. 反馈提交 API

```typescript
async function submitFeedback(
  data: FeedbackData,
  signal?: AbortSignal
): Promise<{ success: boolean; feedbackId?: string; isZdrOrg?: boolean }>
```

API 端点: `POST https://api.anthropic.com/api/claude_cli_feedback`

认证方式：
1. 先调用 `checkAndRefreshOAuthTokenIfNeeded()` 刷新 OAuth Token
2. 使用 `getAuthHeaders()` 获取认证头

特殊处理：
- 403 错误且包含 "Custom data retention settings" 时，返回 `isZdrOrg: true`
- 请求超时: 30 秒
- 取消信号处理: 使用 `axios.isCancel` 检测

---

## 关键代码路径与文件引用

### 本文件位置
```
src/commands/feedback/feedback.tsx
```

### 导入依赖

#### 类型定义
| 导入 | 来源 | 用途 |
|------|------|------|
| `CommandResultDisplay`, `LocalJSXCommandContext` | `../../commands.js` | 命令系统类型 |
| `LocalJSXCommandOnDone` | `../../types/command.js` | 完成回调类型 |
| `Message` | `../../types/message.js` | 消息类型 |

#### UI 组件
| 导入 | 来源 | 用途 |
|------|------|------|
| `Feedback` (组件) | `../../components/Feedback.js` | 实际 UI 组件实现 |

### 被调用方

| 文件 | 调用方式 | 用途 |
|------|----------|------|
| `src/commands/feedback/index.ts` | `load: () => import('./feedback.js')` | 懒加载入口 |

### Feedback 组件的依赖 (从 components/Feedback.tsx 分析)

| 类别 | 依赖文件 | 用途 |
|------|----------|------|
| HTTP 客户端 | `axios` | API 请求 |
| 文件系统 | `fs/promises` | 读取会话记录 |
| React | `react`, `ink` | UI 渲染 |
| 状态/分析 | `src/bootstrap/state.js` | 获取最后 API 请求 |
| 分析服务 | `src/services/analytics/` | 事件日志记录 |
| 消息工具 | `src/utils/messages.js` | 消息处理 |
| 终端尺寸 | `src/hooks/useTerminalSize.js` | 自适应输入框 |
| 快捷键 | `src/keybindings/useKeybinding.js` | 键盘交互 |
| API 服务 | `src/services/api/claude.js` | 调用 Haiku 生成标题 |
| 错误处理 | `src/services/api/errors.js` | API 错误检测 |
| 认证 | `src/utils/auth.js` | OAuth Token 刷新 |
| 浏览器 | `src/utils/browser.js` | 打开 GitHub Issue |
| 调试 | `src/utils/debug.js` | 调试日志 |
| 环境 | `src/utils/env.js` | 平台信息 |
| Git | `src/utils/git.js` | Git 状态 |
| HTTP | `src/utils/http.js` | 认证头、User-Agent |
| 日志 | `src/utils/log.js` | 错误日志获取 |
| 隐私 | `src/utils/privacyLevel.js` | 隐私级别检查 |
| 会话存储 | `src/utils/sessionStorage.js` | 读取子代理记录 |
| 系统提示 | `src/utils/systemPromptType.js` | 系统提示类型 |

---

## 依赖与外部交互

### 外部 API 调用

1. **Anthropic Feedback API**
   - 端点: `POST /api/claude_cli_feedback`
   - 认证: OAuth Bearer Token 或 API Key
   - 数据: JSON 格式的反馈数据
   - 响应: `{ feedback_id: string }`

2. **Haiku API** (用于标题生成)
   - 通过 `queryHaiku()` 调用
   - 系统提示词指导标题生成
   - 用户提示词为反馈描述

### 文件系统交互

1. **会话记录读取**
   - `getTranscriptPath()`: 获取记录文件路径
   - `loadAllSubagentTranscriptsFromDisk()`: 加载子代理记录
   - 大小限制: `MAX_TRANSCRIPT_READ_BYTES` (防止过大文件)

2. **Git 状态获取**
   - `getIsGit()`: 检查是否在 Git 仓库
   - `getGitState()`: 获取分支、commit、remote 信息

### 环境信息收集

```typescript
// 来自 env 对象
env.platform     // 操作系统平台
env.terminal     // 终端类型
MACRO.VERSION    // 编译时版本号
```

---

## 风险、边界与改进建议

### 潜在风险

1. **敏感信息脱敏不完整**
   - 正则表达式可能遗漏新型 API Key 格式
   - 建议: 定期审计脱敏规则，添加更多测试用例

2. **会话记录过大**
   - 虽然有过滤，但大量子代理记录仍可能导致请求体过大
   - 建议: 添加总体大小限制和分片上传机制

3. **API 失败处理**
   - 当前失败时仅显示通用错误消息
   - 建议: 区分网络错误、认证错误、服务器错误，给出不同提示

4. **GitHub URL 长度限制**
   - 7250 字节是实验值，可能随 GitHub 更新而变化
   - 建议: 添加运行时检测和警告

5. **标题生成失败**
   - Haiku API 失败时使用 Fallback，但可能不够描述性
   - 建议: 使用本地简单算法提取关键词作为备选

### 边界情况

1. **空描述**
   - 用户直接按 Enter 提交空描述
   - 当前: 允许提交，但可能无意义
   - 建议: 添加最小长度验证

2. **极长描述**
   - 用户粘贴大量内容
   - 当前: 无长度限制
   - 建议: 添加合理长度限制（如 10000 字符）

3. **无网络连接**
   - 提交时网络断开
   - 当前: 显示错误，允许重试
   - 建议: 添加本地缓存，网络恢复后自动重试

4. **隐私模式**
   - `isEssentialTrafficOnly()` 返回 true
   - 当前: 在 `index.ts` 中禁用命令
   - 如果绕过，提交会失败

5. **ZDR (Zero Data Retention) 组织**
   - 服务器返回 403 和特定错误消息
   - 当前: 显示友好提示
   - 建议: 在 UI 中提前检测并禁用功能

### 改进建议

1. **增强验证**
   ```typescript
   // 添加描述长度验证
   if (description.trim().length < 10) {
     setError('Please provide a more detailed description (at least 10 characters)');
     return;
   }
   ```

2. **附件支持**
   - 允许用户附加截图或日志文件
   - 需要上传机制和安全扫描

3. **反馈分类**
   - 添加问题类型选择（Bug/Feature/Performance/Other）
   - 帮助团队更好地路由反馈

4. **历史记录**
   - 本地保存已提交的反馈记录
   - 允许查看提交状态和历史

5. **离线支持**
   - 缓存反馈到本地存储
   - 网络恢复时自动同步

6. **国际化**
   - 当前仅支持英文界面
   - 考虑多语言支持

### 测试建议

1. **单元测试**
   - `redactSensitiveInfo` 的各种输入
   - `createGitHubIssueUrl` 的 URL 长度处理
   - `generateTitle` 的降级行为

2. **集成测试**
   - 完整提交流程（模拟 API）
   - 网络失败重试
   - 取消信号处理

3. **E2E 测试**
   - 实际提交到测试环境
   - 验证 GitHub Issue 链接可正常打开
