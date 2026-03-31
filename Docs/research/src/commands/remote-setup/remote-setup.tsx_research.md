# Research: src/commands/remote-setup/remote-setup.tsx

## 场景与职责

本文件是 `/web-setup` 命令的交互式 UI 实现，使用 React + Ink 构建终端用户界面。核心场景是引导用户将本地 GitHub CLI 认证同步到 Claude.ai Web 平台，实现以下用户旅程：

1. **首次使用 Web 平台的用户**：快速建立 GitHub 连接，无需在 Web 端重新认证
2. **GitHub CLI 已安装用户**：复用现有凭证，避免重复登录
3. **GitHub CLI 未安装用户**：引导至 Web 端替代认证流程

整体流程：检查状态 → 确认授权 → 导入 Token → 创建环境 → 打开浏览器

## 功能点目的

### 1. 登录状态检查 (checkLoginState)

**目的**：确定当前系统的认证状态，决定后续流程分支。

**状态机**：
```
Claude 登录? 
  ├─ 未登录 → 'not_signed_in'
  └─ 已登录 → GitHub CLI 安装?
                 ├─ 未安装 → 'gh_not_installed'
                 └─ 已安装 → GitHub CLI 认证?
                                ├─ 未认证 → 'gh_not_authenticated'
                                └─ 已认证 → 'has_gh_token' + token
```

**安全设计**：
- 首次调用 `getGhAuthStatus()` 使用 `stdout: 'ignore'`，防止 Token 进入进程内存
- 仅在确认已认证后，第二次调用 `gh auth token` 读取 Token
- Token 立即包装为 `RedactedGithubToken`，防止日志泄露

### 2. 错误消息映射 (errorMessage)

**目的**：将内部错误类型转换为友好的用户提示。

| 错误类型 | 用户消息 | 后续动作 |
|----------|----------|----------|
| `not_signed_in` | "Login failed. Please visit ${codeUrl}..." | 引导至 Web 端登录 |
| `invalid_token` | "GitHub rejected that token..." | 建议重新运行 `gh auth login` |
| `server` | "Server error (${status})..." | 建议稍后重试 |
| `network` | "Couldn't reach the server..." | 建议检查网络连接 |

### 3. 三步式 UI 流程

**Step 1: checking**
- 显示 "Checking login status…" 加载状态
- 异步执行 `checkLoginState()`
- 根据结果决定下一步

**Step 2: confirm**
- 显示确认对话框："Connect Claude on the web to GitHub?"
- 说明："Claude on the web requires connecting to your GitHub account to clone and push code on your behalf"
- 提供 "Continue" / "Cancel" 选项

**Step 3: uploading**
- 显示 "Connecting GitHub to Claude…" 加载状态
- 调用 `importGithubToken()` 导入 Token
- 成功后调用 `createDefaultEnvironment()` 创建默认环境
- 打开浏览器跳转至 claude.ai/code

### 4. 遥测事件

| 事件名 | 触发时机 | 元数据 |
|--------|----------|--------|
| `tengu_remote_setup_started` | 组件挂载时 | - |
| `tengu_remote_setup_result` | 流程结束时 | `result`: success / not_signed_in / gh_not_installed / gh_not_authenticated / import_failed / cancelled |
| `tengu_remote_setup_result` (import_failed) | 导入失败时 | 额外包含 `error_kind` |

## 具体技术实现

### 组件架构

```typescript
// 步骤状态联合类型
type Step =
  | { name: 'checking' }
  | { name: 'confirm'; token: RedactedGithubToken }
  | { name: 'uploading' }

// 检查结果联合类型
type CheckResult =
  | { status: 'not_signed_in' }
  | { status: 'has_gh_token'; token: RedactedGithubToken }
  | { status: 'gh_not_installed' }
  | { status: 'gh_not_authenticated' }
```

### 核心流程代码

```typescript
// 状态检查与分支处理
useEffect(() => {
  logEvent('tengu_remote_setup_started', {})
  void checkLoginState().then(async result => {
    switch (result.status) {
      case 'not_signed_in':
        // 未登录 Claude，提示先运行 /login
        onDone('Not signed in to Claude. Run /login first.')
        return
      case 'gh_not_installed':
      case 'gh_not_authenticated':
        // 打开浏览器到替代认证页面
        await openBrowser(`${getCodeWebUrl()}/onboarding?step=alt-auth`)
        onDone(/* 相应提示 */)
        return
      case 'has_gh_token':
        // 进入确认步骤
        setStep({ name: 'confirm', token: result.token })
    }
  })
}, [])
```

### 确认处理流程

```typescript
const handleConfirm = async (token: RedactedGithubToken) => {
  setStep({ name: 'uploading' })
  const result = await importGithubToken(token)
  if (!result.ok) {
    // 记录失败事件并显示错误
    logEvent('tengu_remote_setup_result', { result: 'import_failed', error_kind })
    onDone(errorMessage(result.error, getCodeWebUrl()))
    return
  }
  
  // Best-effort 环境创建
  await createDefaultEnvironment()
  
  // 打开 Web 端
  await openBrowser(getCodeWebUrl())
  logEvent('tengu_remote_setup_result', { result: 'success' })
  onDone(`Connected as ${result.result.github_username}. Opened ${url}`)
}
```

## 关键代码路径与文件引用

### 依赖文件

| 文件 | 用途 |
|------|------|
| `src/components/CustomSelect/index.js` | `Select` 组件 - 选项选择 UI |
| `src/components/design-system/Dialog.js` | `Dialog` 组件 - 确认对话框 |
| `src/components/design-system/LoadingState.js` | `LoadingState` 组件 - 加载状态 |
| `src/ink.js` | `Box`, `Text` - Ink 基础组件 |
| `src/services/analytics/index.js` | `logEvent` - 遥测事件 |
| `src/types/command.js` | `LocalJSXCommandOnDone` 类型 |
| `src/utils/browser.js` | `openBrowser()` - 浏览器打开 |
| `src/utils/github/ghAuthStatus.js` | `getGhAuthStatus()` - GitHub CLI 状态检查 |
| `./api.js` | `createDefaultEnvironment`, `getCodeWebUrl`, `importGithubToken`, `isSignedIn`, `RedactedGithubToken` |

### 被调用方

| 文件 | 用途 |
|------|------|
| `src/commands/remote-setup/index.ts` | 懒加载本模块 |

### 调用链

```
remote-setup.tsx
├── checkLoginState()
│   ├── isSignedIn() → api.ts
│   ├── getGhAuthStatus() → ghAuthStatus.ts
│   └── execa('gh', ['auth', 'token'])
├── importGithubToken() → api.ts
├── createDefaultEnvironment() → api.ts
└── openBrowser() → browser.ts
```

## 依赖与外部交互

### 外部系统

1. **GitHub CLI (`gh`)**
   - `gh auth status`：检查认证状态
   - `gh auth token`：获取访问令牌
   - 要求用户已安装并登录 GitHub CLI

2. **系统浏览器**
   - 通过 `openBrowser()` 调用系统默认浏览器
   - 支持 macOS (`open`)、Linux (`xdg-open`)、Windows (`rundll32`/`explorer`)

### 内部依赖图

```
remote-setup.tsx
├── execa (子进程执行)
├── react (UI 框架)
├── ../../components/CustomSelect/index.js
├── ../../components/design-system/Dialog.js
├── ../../components/design-system/LoadingState.js
├── ../../ink.js
├── ../../services/analytics/index.js
├── ../../types/command.js
├── ../../utils/browser.js
├── ../../utils/github/ghAuthStatus.js
└── ./api.js
```

## 风险、边界与改进建议

### 安全风险

1. **Token 内存暴露**
   - **现状**：`execa('gh', ['auth', 'token'])` 会将 Token 读入内存
   - **缓解**：立即包装为 `RedactedGithubToken`，限制暴露范围
   - **建议**：考虑使用文件描述符或临时文件传递，避免命令行参数

2. **浏览器 URL 注入**
   - **现状**：`openBrowser(url)` 直接传递 URL
   - **缓解**：`browser.ts` 中已验证 URL 协议（仅 http/https）
   - **建议**：额外验证 URL 域名白名单

### 功能边界

1. **竞态条件**
   - `useEffect` 依赖数组有意省略 `onDone`（注释说明其跨渲染稳定）
   - ESLint 禁用规则：`react-hooks/exhaustive-deps`
   - 风险：如果 `onDone` 实现变化，可能导致内存泄漏或重复调用

2. **Best-effort 环境创建**
   - `createDefaultEnvironment()` 失败不阻塞流程
   - 用户可能在 Web 端看到环境配置向导
   - 这是设计决策，但可能导致用户体验不一致

3. **浏览器打开失败**
   - `openBrowser()` 返回 boolean 但不检查
   - 如果浏览器打开失败，用户可能困惑
   - 建议：添加失败提示，提供手动 URL

4. **GitHub CLI 依赖**
   - 严格要求 `gh` 命令可用
   - 无备用方案（如直接读取 ~/.gh 配置文件）
   - 建议：考虑支持直接 Token 粘贴作为 fallback

### 用户体验改进

1. **进度指示**
   - 当前 "uploading" 步骤可能持续数秒
   - 建议：添加更细粒度的进度（如 "验证 Token..."、"创建环境..."）

2. **取消机制**
   - 导入过程中无法取消
   - 建议：添加超时或取消按钮

3. **成功确认**
   - 仅显示文本消息 "Connected as ${username}"
   - 建议：显示 GitHub 头像或更多信息增强确认感

4. **错误恢复**
   - 某些错误（如 `invalid_token`）需要用户重新运行 `gh auth login`
   - 建议：提供一键执行该命令的快捷方式

### 代码质量改进

1. **类型安全**
   - `SafeString` 类型强制转换较多
   - 建议：定义更精确的遥测元数据类型

2. **测试覆盖**
   - 需要 Mock `execa`、`openBrowser`、API 调用
   - 建议：添加单元测试覆盖各分支

3. **国际化**
   - 当前全英文界面
   - 建议：支持多语言（如用户系统语言）

### 监控与运维

1. **成功率监控**
   - 通过 `tengu_remote_setup_result` 事件可计算成功率
   - 建议：添加失败原因分布图表

2. **性能监控**
   - 各步骤耗时未知
   - 建议：添加步骤耗时指标

3. **漏斗分析**
   - started → confirm → success 的转化漏斗
   - 建议：在数据分析平台建立漏斗报表
