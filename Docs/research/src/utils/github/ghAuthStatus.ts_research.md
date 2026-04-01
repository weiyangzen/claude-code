# 研究文档：src/utils/github/ghAuthStatus.ts

> 研究对象：`src/utils/github/ghAuthStatus.ts` 及其完整上下文依赖  
> 执行器：kimi (k2p5)  
> 生成日期：2026-04-01

---

## 1. 场景与职责

### 1.1 模块定位

`src/utils/github/ghAuthStatus.ts` 是 Claude Code CLI 中负责 **GitHub CLI（`gh`）安装与认证状态探测** 的专用工具模块。它在整个代码库中处于基础设施层（utility layer），不直接面向用户交互，而是为上层业务（启动遥测、Web 设置流程）提供原子化的状态查询能力。

该模块的核心职责可归纳为四点：

1. **探测 `gh` 是否已安装**：通过命令查找机制（`which`/`where.exe`）判断目标系统是否存在可执行的 `gh` 二进制文件。
2. **探测 `gh` 是否已完成认证**：在确认 `gh` 已安装的前提下，调用 `gh auth token` 读取本地存储的 GitHub Token，以退出码推断认证状态。
3. **安全隔离敏感凭证**：在状态探测阶段显式设置 `stdout: 'ignore'`，确保 GitHub Token 不会进入当前 Node/Bun 进程的内存空间。
4. **为遥测与业务决策提供输入**：将三态结果（`'authenticated'` / `'not_authenticated'` / `'not_installed'`）返回给调用方，用于匿名化统计或 UI 流程分支。

### 1.2 使用场景

| 场景 | 调用方 | 用途 |
|------|--------|------|
| **启动遥测收集** | `src/main.tsx::logStartupTelemetry()` | 在应用启动时并行收集 `gh_auth_status`，随 `tengu_startup_telemetry` 事件上报 |
| **Web 设置流程（`/web-setup`）** | `src/commands/remote-setup/remote-setup.tsx::checkLoginState()` | 判断用户是否具备通过本地 `gh` 凭证向 Claude Web 导入 GitHub Token 的条件 |

---

## 2. 功能点目的

### 2.1 核心功能：`getGhAuthStatus()`

**函数签名**：

```typescript
export type GhAuthStatus =
  | 'authenticated'
  | 'not_authenticated'
  | 'not_installed'

export async function getGhAuthStatus(): Promise<GhAuthStatus>
```

**设计目的与决策**：

1. **优先使用 `gh auth token` 而非 `gh auth status`**  
   `gh auth status` 会向 `api.github.com` 发起网络请求以验证凭证有效性，而 `gh auth token` 仅读取本地配置文件或系统钥匙串（keyring）。后者在网络隔离、慢网或高隐私要求场景下更可靠，且执行更快。

2. **隐私保护：状态探测不读取 Token 内容**  
   函数内部调用 `execa('gh', ['auth', token'], { stdout: 'ignore', ... })`。`stdout: 'ignore'` 会将子进程标准输出重定向到 `process.stdout` 的忽略模式（或等效的空设备），从而保证即使 `gh` 成功返回了 Token 字符串，该字符串也**不会**被父进程捕获。这是专门为遥测场景设计的安全边界。

3. **性能优化：零子进程的 `which()` 探测（Bun 环境）**  
   在 Bun 运行时，`which()` 直接映射到 `Bun.which()`，无需派生子进程；在 Node 运行时则回退到平台原生的 `which` / `where.exe` 子进程调用。

4. **超时与容错**  
   `gh auth token` 设置了 5 秒超时（`timeout: 5000`），并启用 `reject: false`，确保任何非零退出码或超时都不会抛出未捕获异常，而是被收敛为 `'not_authenticated'`。

### 2.2 状态机

```
          ┌─────────────┐
          │ 调用开始    │
          └──────┬──────┘
                 ▼
          ┌─────────────┐
          │ which('gh') │◄── 检测 gh CLI 是否存在
          └──────┬──────┘
                 │
         ┌───────┴───────┐
         ▼               ▼
    ┌─────────┐    ┌──────────┐
    │  null   │    │ 有路径    │
    └────┬────┘    └────┬─────┘
         │              ▼
         │       ┌─────────────────┐
         │       │ gh auth token   │◄── 检测认证状态（stdout 被忽略）
         │       │ (stdout:ignore) │
         │       └────────┬────────┘
         │                │
         │         ┌──────┴──────┐
         │         ▼             ▼
         │   ┌─────────┐    ┌──────────┐
         │   │exitCode │    │ exitCode │
         │   │   0     │    │  非 0    │
         │   └───┬─────┘    └────┬─────┘
         │       │               │
         ▼       ▼               ▼
   ┌──────────┐ ┌──────────┐ ┌─────────────────┐
   │not_inst. │ │authenticated│ │not_authenticated│
   └──────────┘ └──────────┘ └─────────────────┘
```

---

## 3. 具体技术实现

### 3.1 关键流程

#### 流程 1：GitHub CLI 存在性检测

**文件**：`src/utils/github/ghAuthStatus.ts`（第 18–21 行）

```typescript
const ghPath = await which('gh')
if (!ghPath) {
  return 'not_installed'
}
```

- 依赖 `src/utils/which.ts` 提供的 `which()` 异步函数。
- `which.ts` 的实现逻辑：
  - 若全局存在 `Bun` 且 `Bun.which` 为函数，则直接调用 `Bun.which(command)`（零子进程、基于 Bun 内部 PATH 解析）。
  - 否则调用 `whichNodeAsync(command)`：
    - Windows 平台执行 `where.exe ${command}`（`shell: true`），取返回结果的第一行。
    - POSIX 平台执行 `which ${command}`（`shell: true`），取标准输出并 `trim()`。
- `whichNodeAsync` 同样使用 `execa`，但 `stderr: 'ignore'`，`reject: false`，失败时返回 `null`。

#### 流程 2：认证状态检测

**文件**：`src/utils/github/ghAuthStatus.ts`（第 22–28 行）

```typescript
const { exitCode } = await execa('gh', ['auth', 'token'], {
  stdout: 'ignore',
  stderr: 'ignore',
  timeout: 5000,
  reject: false,
})
return exitCode === 0 ? 'authenticated' : 'not_authenticated'
```

**安全与实现细节**：

- `stdout: 'ignore'` 在 `execa` 内部通常等价于将子进程 `stdio[1]` 设置为 `'ignore'`（绑定到 `process.stdout` 的忽略流或 `null`），确保 `gh` 输出的 Token 不会被缓冲到父进程内存。
- `reject: false` 是 `execa` 的选项，表示即使子进程以非零退出码结束，返回的 Promise 也** resolve **而不是 reject。这样可以通过检查 `exitCode` 安全地判断状态。
- 超时机制由 `execa` 内置实现：超过 5000ms 后自动向子进程发送终止信号，此时 `exitCode` 通常为 `undefined`（或由于信号终止导致的非零值），因此会被归类为 `'not_authenticated'`。

### 3.2 数据结构

#### 类型定义

```typescript
// src/utils/github/ghAuthStatus.ts
export type GhAuthStatus =
  | 'authenticated'      // gh 已安装且已登录
  | 'not_authenticated'  // gh 已安装但未登录（或超时、钥匙串锁定）
  | 'not_installed'      // gh 未安装（或不在 PATH 中）
```

#### 返回值语义与业务映射

| 返回值 | 业务含义 | `main.tsx` 处理 | `remote-setup.tsx` 处理 |
|--------|----------|-----------------|-------------------------|
| `'authenticated'` | 用户具备可用的 GitHub CLI 凭证 | 作为 `gh_auth_status` 字段上报遥测 | 进入 Token 读取与导入流程 |
| `'not_authenticated'` | `gh` 存在但无有效凭证 | 作为 `gh_auth_status` 字段上报遥测 | 引导用户运行 `gh auth login` 或转 Web OAuth |
| `'not_installed'` | `gh` 不在 PATH 中 | 作为 `gh_auth_status` 字段上报遥测 | 引导用户安装 GitHub CLI 或转 Web OAuth |

### 3.3 协议与命令

#### 外部命令调用表

| 命令 | 参数 | 用途 | 超时 | 输出处理 |
|------|------|------|------|----------|
| `gh` | `['auth', 'token']` | 读取本地存储的 GitHub Token | 5000ms | `stdout: 'ignore'`（状态探测） |
| `which` / `where.exe` | `'gh'` | 检测 `gh` 可执行文件位置 | 无显式超时（依赖 `execa` 默认） | `stdout: 'pipe'`（在 `which.ts` 中） |

#### 依赖包

| 包名 | 用途 | 备注 |
|------|------|------|
| `execa` | 跨平台子进程执行 | 项目中多处使用（`src/utils/execFileNoThrow.ts`、`src/utils/which.ts` 等），统一处理 Windows 的 `.bat`/`.cmd` 兼容性与 shell 转义 |

---

## 4. 关键代码路径与文件引用

### 4.1 核心文件

```
src/utils/github/
└── ghAuthStatus.ts          # 被研究对象（29 行）
```

### 4.2 直接依赖

```
src/utils/github/ghAuthStatus.ts
├── import { execa } from 'execa'              # 子进程执行库（外部依赖）
└── import { which } from '../which.js'        # 命令查找工具（内部模块）
    ├── Bun.which()  或
    └── whichNodeAsync() / whichNodeSync()
        └── execa('which gh') / execSync_DEPRECATED('where.exe gh')
            └── src/utils/execSyncWrapper.ts::execSync_DEPRECATED()
                └── node:child_process::execSync()
```

### 4.3 调用方文件

#### 调用方 1：启动遥测

**文件**：`src/main.tsx`

```typescript
// 第 114 行
import { getGhAuthStatus } from './utils/github/ghAuthStatus.js'

// 第 307–321 行
async function logStartupTelemetry(): Promise<void> {
  if (isAnalyticsDisabled()) return
  const [isGit, worktreeCount, ghAuthStatus] = await Promise.all([
    getIsGit(),
    getWorktreeCount(),
    getGhAuthStatus(),
  ])
  logEvent('tengu_startup_telemetry', {
    is_git: isGit,
    worktree_count: worktreeCount,
    gh_auth_status: ghAuthStatus as AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS,
    // ... 其他字段
  })
}
```

- `getGhAuthStatus()` 与 `getIsGit()`、`getWorktreeCount()` **并行执行**（`Promise.all`），以最小化启动阻塞时间。
- 结果通过 `logEvent()` 作为 `tengu_startup_telemetry` 事件的一部分上报，字段名为 `gh_auth_status`。
- 使用 `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` 类型别名进行编译时安全标记，防止误将文件路径或代码字符串传入遥测。

#### 调用方 2：Web 设置流程

**文件**：`src/commands/remote-setup/remote-setup.tsx`

```typescript
// 第 11 行
import { getGhAuthStatus } from '../../utils/github/ghAuthStatus.js'

// 第 23–61 行
async function checkLoginState(): Promise<CheckResult> {
  if (!(await isSignedIn())) {
    return { status: 'not_signed_in' }
  }
  const ghStatus = await getGhAuthStatus()
  if (ghStatus === 'not_installed') {
    return { status: 'gh_not_installed' }
  }
  if (ghStatus === 'not_authenticated') {
    return { status: 'gh_not_authenticated' }
  }

  // ghStatus === 'authenticated'. getGhAuthStatus spawns with stdout:'ignore'
  // (telemetry-safe); spawn once more with stdout:'pipe' to read the token.
  const { stdout } = await execa('gh', ['auth', 'token'], {
    stdout: 'pipe',
    stderr: 'ignore',
    timeout: 5000,
    reject: false,
  })
  const trimmed = stdout.trim()
  if (!trimmed) {
    return { status: 'gh_not_authenticated' }
  }
  return {
    status: 'has_gh_token',
    token: new RedactedGithubToken(trimmed),
  }
}
```

- 该调用方体现了 `getGhAuthStatus()` 的**两阶段设计**：
  1. 第一阶段调用 `getGhAuthStatus()` 做安全的状态探测（Token 不进入内存）。
  2. 仅在确认 `'authenticated'` 后，第二阶段才直接调用 `execa('gh', ['auth', 'token'], { stdout: 'pipe' })` 读取实际 Token。
- 读取到的原始 Token 被立即包装进 `RedactedGithubToken` 类（定义于 `src/commands/remote-setup/api.ts`），该类重写了 `toString()`、`toJSON()` 和 `util.inspect.custom`，确保 Token 不会通过日志、错误信息或 JSON 序列化意外泄露。

### 4.4 完整调用链

#### 遥测链路

```
src/main.tsx::logStartupTelemetry()
├── getIsGit()                         # 并行分支 1
├── getWorktreeCount()                 # 并行分支 2
└── getGhAuthStatus()                  # 并行分支 3
    ├── src/utils/which.ts::which('gh')
    │   ├── Bun.which('gh')            # Bun 环境（零子进程）
    │   └── whichNodeAsync('gh')       # Node 环境
    │       └── execa('which gh', { shell: true, ... })
    └── execa('gh', ['auth', 'token'], { stdout: 'ignore', timeout: 5000, reject: false })
        └── exitCode === 0 ? 'authenticated' : 'not_authenticated'
```

#### Web 设置链路

```
src/commands/remote-setup/remote-setup.tsx::checkLoginState()
├── isSignedIn()                       # 检查 Claude 登录状态
└── getGhAuthStatus()                  # 安全状态探测
    ├── which('gh')
    └── execa('gh', ['auth', 'token'], { stdout: 'ignore', ... })

# 若状态为 authenticated，继续读取 Token：
execa('gh', ['auth', 'token'], { stdout: 'pipe', ... })
└── stdout (raw GitHub Token)
    └── new RedactedGithubToken(trimmed)
        └── importGithubToken(token)   # POST 到 Claude 后端
```

---

## 5. 依赖与外部交互

### 5.1 内部依赖

| 依赖模块 | 路径 | 用途 | 关键实现 |
|----------|------|------|----------|
| `which` | `src/utils/which.ts` | 检测 `gh` 命令是否存在 | `Bun.which()` 优先，Node 回退到 `which`/`where.exe` |
| `execSyncWrapper` | `src/utils/execSyncWrapper.ts` | 为 `which.ts` 的同步路径提供包装后的 `execSync` | 基于 `node:child_process`，附加慢操作日志 |
| `execFileNoThrow` | `src/utils/execFileNoThrow.ts` | 项目中另一套 `execa` 封装（**未被 `ghAuthStatus.ts` 直接使用**，但属于同一子进程工具家族） | 统一处理 `reject: false`、错误信息提取、跨平台兼容 |

#### `which.ts` 实现细节补充

**文件**：`src/utils/which.ts`（82 行）

```typescript
const bunWhich =
  typeof Bun !== 'undefined' && typeof Bun.which === 'function'
    ? Bun.which
    : null

export const which: (command: string) => Promise<string | null> = bunWhich
  ? async command => bunWhich(command)
  : whichNodeAsync
```

- **Bun 路径**：`Bun.which()` 是 Bun 运行时提供的原生 API，性能最优，不产生额外子进程。
- **Node 异步路径**：`whichNodeAsync()` 使用 `execa(command, { shell: true, stderr: 'ignore', reject: false })`，区分 Windows（`where.exe`）与 POSIX（`which`）。
- **Node 同步路径**：`whichSync()` 使用 `execSync_DEPRECATED()`（包装后的 `child_process.execSync`），供同步场景调用。

### 5.2 外部依赖

| 依赖包 | 用途 | 在本文件中的使用方式 |
|--------|------|----------------------|
| `execa` | 跨平台子进程执行 | `execa('gh', ['auth', 'token'], { ... })` |

**备注**：本仓库工作目录中**未发现 `package.json` 或 `tsconfig.json`**，因此无法直接确认 `execa` 的具体版本约束。但从代码中使用的 API（如 `reject: false`、`timeout`）来看，符合 `execa` v5+ 的接口规范。

### 5.3 外部系统交互

| 外部系统 | 交互方式 | 数据流向 | 网络行为 |
|----------|----------|----------|----------|
| GitHub CLI (`gh`) | 本地子进程调用（`execa`） | 读取本地 Token（或探测其存在性） | **无网络请求**（`auth token` 为纯本地操作） |
| 操作系统 PATH / 可执行文件查找 | `which` / `where.exe` | 获取 `gh` 可执行文件的绝对路径 | 无 |
| 本地密钥存储（macOS Keychain、Windows Credential Manager、Linux secret-service 等） | 间接通过 `gh auth token` 访问 | `gh` 从系统钥匙串中读取已保存的 OAuth Token | 无 |

**重要说明**：`getGhAuthStatus()` 本身**不发起任何网络请求**。这与 `gh auth status` 有本质区别——后者会调用 GitHub API 验证 Token 有效性，可能在离线环境或企业内网（GitHub Enterprise Server）场景下表现不同。

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 风险 1：超时导致的误判

**问题描述**：`gh auth token` 在读取系统钥匙串时，若用户首次解锁后钥匙串响应较慢，或系统处于高负载状态，可能超过 5 秒超时。超时后 `execa` 会终止子进程，返回非零退出码（或 `exitCode` 为 `undefined`），从而被误判为 `'not_authenticated'`。

**代码位置**：`src/utils/github/ghAuthStatus.ts:25`

**影响范围**：
- 启动遥测：导致 `gh_auth_status` 统计偏低（将实际已认证用户记为未认证）。
- Web 设置流程：可能引导已认证用户重复执行 `gh auth login` 或转向 Web OAuth，降低用户体验。

**缓解措施**：
- 当前代码层面无重试逻辑。调用方（`remote-setup.tsx`）可通过 UI 提示用户重试，或提供“使用 Web 方式连接”的 fallback。

#### 风险 2：Token 在调用链中的二次暴露

**问题描述**：虽然 `getGhAuthStatus()` 使用 `stdout: 'ignore'` 做到了状态探测阶段的 Token 隔离，但 `remote-setup.tsx` 在确认 `'authenticated'` 后会**再次**执行 `gh auth token` 并读取 Token 内容。此时 Token 以明文形式短暂存在于 Node 进程的 `stdout` 字符串中，随后被传入 `RedactedGithubToken`。

**代码位置**：`src/commands/remote-setup/remote-setup.tsx:43–50`

**现有缓解措施**：
- `RedactedGithubToken` 类（`src/commands/remote-setup/api.ts:16–33`）通过私有字段 `#value` 存储原始 Token，并重写 `toString()`、`toJSON()`、`util.inspect.custom`，有效防止日志、异常、调试输出中的意外泄露。
- Token 仅在 `importGithubToken()` 的 HTTP POST 请求体中通过 `.reveal()` 暴露。

**残余风险**：
- 在 `trimmed` 变量存活期间，若进程发生核心转储（core dump）或被外部内存扫描工具读取，Token 仍有短暂暴露窗口。

#### 风险 3：并发子进程负载

**问题描述**：在 `main.tsx` 的 `logStartupTelemetry()` 中，`getGhAuthStatus()` 与 `getIsGit()`、`getWorktreeCount()` 并行执行。每个函数都可能派生子进程（`git` 命令或 `which` / `gh` 命令），在启动瞬间增加了系统负载。

**代码位置**：`src/main.tsx:309`

**影响评估**：对于现代桌面系统，3 个并发子进程的影响可忽略；但在资源受限的 CI 容器或远程开发环境中，可能略微增加启动延迟。

### 6.2 边界情况

| 边界情况 | 实际行为 | 备注 |
|----------|----------|------|
| `gh` 未安装 | 返回 `'not_installed'` | 由 `which('gh')` 返回 `null` 触发 |
| `gh` 已安装但未执行 `gh auth login` | 返回 `'not_authenticated'` | `gh auth token` 退出码非零 |
| `gh` 已安装且已登录 | 返回 `'authenticated'` | `gh auth token` 退出码为 0 |
| `gh` 命令执行时间超过 5 秒 | 返回 `'not_authenticated'` | `execa` 超时后子进程被终止 |
| 系统钥匙串被锁定或需要用户交互 | 可能超时或返回非零退出码 | 取决于操作系统钥匙串的提示行为；无头环境（headless）下常见 |
| `gh` 指向一个非官方脚本/别名 | 行为不可预测 | `which()` 仅检查 PATH 中是否存在名为 `gh` 的可执行文件，不验证其签名或版本 |

### 6.3 改进建议

#### 建议 1：引入会话级缓存

**现状**：每次调用 `getGhAuthStatus()` 都会重新执行 `which('gh')` 和 `gh auth token`。在启动遥测与后续功能检查（如 `/web-setup`）连续发生时，存在冗余子进程开销。

**改进方案**：

```typescript
let cachedStatus: GhAuthStatus | null = null

export async function getGhAuthStatus(): Promise<GhAuthStatus> {
  if (cachedStatus !== null) {
    return cachedStatus
  }
  // ... 原有逻辑 ...
  cachedStatus = result
  return result
}
```

**适用场景**：单次 CLI 会话内，GitHub CLI 的安装与认证状态通常不会发生变化，缓存可以安全地减少子进程调用。

#### 建议 2：细化 `'not_authenticated'` 的原因分类

**现状**：`'not_authenticated'` 是一个粗粒度状态，无法区分“未登录”、“命令超时”、“子进程执行错误（如 `gh` 损坏）”。

**改进方案**：在不破坏现有三态接口的前提下，可新增一个详细版本供调试使用：

```typescript
export type GhAuthStatusDetailed =
  | { status: 'authenticated' }
  | { status: 'not_authenticated'; reason: 'not_logged_in' | 'timeout' | 'error' }
  | { status: 'not_installed' }

export async function getGhAuthStatusDetailed(): Promise<GhAuthStatusDetailed> { ... }

export async function getGhAuthStatus(): Promise<GhAuthStatus> {
  const detailed = await getGhAuthStatusDetailed()
  return detailed.status
}
```

**收益**：便于在遥测中区分“真正的未认证用户”与“环境异常用户”，优化产品决策。

#### 建议 3：增加环境变量覆盖（测试/调试友好）

**现状**：在 CI 或自动化测试环境中，难以稳定模拟 `gh` 的三种状态，因为状态完全取决于外部系统环境。

**改进方案**：

```typescript
export async function getGhAuthStatus(): Promise<GhAuthStatus> {
  const forced = process.env.CLAUDE_DEBUG_GH_STATUS
  if (
    forced === 'authenticated' ||
    forced === 'not_authenticated' ||
    forced === 'not_installed'
  ) {
    return forced
  }
  // ... 原有逻辑 ...
}
```

**收益**：便于编写单元测试和进行故障注入调试，且不会影响生产环境（未设置环境变量时行为不变）。

#### 建议 4：统一 GitHub 相关工具目录

**现状**：GitHub 相关功能分散在多个位置：
- `src/utils/github/ghAuthStatus.ts` — 认证状态
- `src/utils/ghPrStatus.ts` — PR 状态查询
- `src/utils/githubRepoPathMapping.ts` — 仓库路径映射

**改进方案**：考虑将 `ghPrStatus.ts` 和 `githubRepoPathMapping.ts` 迁移到 `src/utils/github/` 目录下，并增加 `index.ts` 统一导出：

```
src/utils/github/
├── index.ts
├── authStatus.ts         # 由 ghAuthStatus.ts 重命名
├── prStatus.ts           # 由 ghPrStatus.ts 迁移
└── repoPathMapping.ts    # 由 githubRepoPathMapping.ts 迁移
```

**收益**：提升模块内聚性，降低新开发者理解 GitHub 工具集的认知负担。

### 6.4 测试建议

当前仓库中**未发现任何针对 `getGhAuthStatus()` 的单元测试或集成测试**（`find` 未检索到 `*.test.ts` / `*.spec.ts` 文件）。建议补充以下测试：

1. **Mock 单元测试**：使用 `vi.mock('execa')`（若使用 Vitest）或 `jest.mock('execa')` 模拟 `gh auth token` 的不同退出码，验证三态返回逻辑。
2. **`which()` Mock 测试**：模拟 `which` 模块返回 `null` 或具体路径，验证 `'not_installed'` 分支。
3. **超时行为测试**：模拟 `execa` 超时（如通过 `vi.useFakeTimers()`），验证函数返回 `'not_authenticated'` 且不抛出异常。
4. **集成测试**：在已安装 `gh` 的容器环境中运行真实调用，验证与 GitHub CLI 的实际交互行为。

---

## 7. 附录

### 7.1 相关文件清单

| 文件路径 | 类型 | 关系 |
|----------|------|------|
| `src/utils/github/ghAuthStatus.ts` | 核心实现 | **被研究对象** |
| `src/utils/which.ts` | 直接依赖 | 命令查找工具 |
| `src/utils/execSyncWrapper.ts` | 间接依赖 | 为 `which.ts` 同步路径提供 `execSync` 包装 |
| `src/utils/execFileNoThrow.ts` | 相关工具 | 项目中另一套 `execa` 封装 |
| `src/main.tsx` | 调用方 | 启动遥测收集 |
| `src/commands/remote-setup/remote-setup.tsx` | 调用方 | Web 设置流程（`/web-setup`） |
| `src/commands/remote-setup/api.ts` | 相关 | Token 导入 API 与 `RedactedGithubToken` 实现 |
| `src/utils/ghPrStatus.ts` | 相关 | PR 状态查询（使用 `gh pr view`） |
| `src/utils/githubRepoPathMapping.ts` | 相关 | 本地仓库路径映射管理 |

### 7.2 代码统计

- **核心代码行数**：29 行（`ghAuthStatus.ts`）
- **直接依赖文件**：`src/utils/which.ts`（82 行）
- **调用点数量**：2 处（`main.tsx`、`remote-setup.tsx`）
- **测试覆盖**：0（本仓库未检索到测试文件）

---

*文档结束*
