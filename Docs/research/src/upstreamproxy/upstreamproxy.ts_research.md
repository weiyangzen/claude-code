# src/upstreamproxy/upstreamproxy.ts 深度研究文档

## 场景与职责

### 部署场景

`upstreamproxy.ts` 是 CCR (Claude Code Remote) 会话容器内的**上游代理初始化与配置中枢**。当 Claude Code CLI 在 CCR 容器中启动时，该模块负责完成从“会话令牌读取”到“子进程环境变量注入”的完整闭环，使容器内的 Agent 及其子进程（Bash、MCP、LSP、Hooks）能够透明地通过组织级上游代理访问外部服务。

| 场景 | 说明 |
|------|------|
| **CCR 远程会话** | `CLAUDE_CODE_REMOTE=1` 时激活，标识当前运行在 CCR 容器环境 |
| **GrowthBook 功能开关** | `CCR_UPSTREAM_PROXY_ENABLED` 由服务端注入，控制是否启用上游代理 |
| **企业级网络出口** | 容器内网受限，需通过 CCR 上游代理进行 MITM 式 TLS 代理和凭证注入 |
| **多子进程环境继承** | 所有子进程统一通过 `subprocessEnv()` 获取代理环境变量 |

### 核心职责

1. **功能门控**：基于环境变量判断是否启用上游代理，任何条件不满足即安全退出（返回 `{ enabled: false }`）。
2. **安全凭证管理**：从 `/run/ccr/session_token` 读取一次性会话令牌，中继启动成功后立即 `unlink()` 删除，使令牌仅存于堆内存。
3. **进程防转储保护**：在 Linux + Bun 环境下调用 `prctl(PR_SET_DUMPABLE, 0)`，阻止同 UID 进程通过 `ptrace` / `gdb` 读取堆内存中的令牌。
4. **CA 证书捆绑**：从 CCR API 下载上游代理 CA 证书，并与系统 CA bundle (`/etc/ssl/certs/ca-certificates.crt`) 拼接，输出到 `~/.ccr/ca-bundle.crt`。
5. **本地中继启动**：调用 `relay.ts` 启动 CONNECT-over-WebSocket 本地中继，并注册清理函数以便优雅关闭。
6. **子进程环境注入**：通过 `getUpstreamProxyEnv()` 向所有子进程暴露 `HTTPS_PROXY`、`SSL_CERT_FILE`、`NODE_EXTRA_CA_CERTS` 等变量。

### 设计原则

> **Fail-open（故障开放）**：初始化链中的任何步骤（读 token、下载 CA、启动 relay）失败都不会抛异常中断 CLI 启动，而是记录 `warn` 级别日志并返回 `{ enabled: false }`，确保正常会话不受影响。
> 
> **延迟加载（Lazy Import）**：`init.ts` 通过动态 `import()` 加载本模块，避免非 CCR 用户承担模块图解析和 `ws` 包加载的启动开销。

---

## 功能点目的

### 功能总览

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        CCR 容器初始化阶段                                    │
│                                                                              │
│  ┌─────────────────┐    ┌─────────────────────────┐    ┌─────────────────┐  │
│  │  环境变量检查    │───▶│  initUpstreamProxy()    │───▶│  读取 session   │  │
│  │  CLAUDE_CODE_   │    │  (upstreamproxy.ts)     │    │  token 文件     │  │
│  │  REMOTE         │    └─────────────────────────┘    └─────────────────┘  │
│  └─────────────────┘              │                        │                 │
│                                   ▼                        ▼                 │
│                          ┌─────────────────┐    ┌─────────────────┐         │
│                          │  启动 relay     │◄───│  下载 CA 证书    │         │
│                          │  (relay.ts)     │    │  & 拼接 bundle   │         │
│                          └─────────────────┘    └─────────────────┘         │
│                                   │                                         │
│                                   ▼                                         │
│                          ┌─────────────────┐                                │
│                          │  注册清理函数    │                                │
│                          │  unlink token   │                                │
│                          └─────────────────┘                                │
│                                   │                                         │
│                                   ▼                                         │
│              子进程环境: HTTPS_PROXY, SSL_CERT_FILE, NODE_EXTRA_CA_CERTS...  │
│                                   ▲                                         │
│                          ┌─────────────────┐                                │
│                          │ subprocessEnv() │                                │
│                          │ (subprocessEnv) │                                │
│                          └─────────────────┘                                │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 各功能点详细说明

#### 1. 环境门控 (`initUpstreamProxy`)

- **`CLAUDE_CODE_REMOTE`**：CCR 环境标识。非 CCR 场景（本地桌面、VSCode 扩展）直接跳过。
- **`CCR_UPSTREAM_PROXY_ENABLED`**：服务端 GrowthBook 功能开关。注释明确指出不能在客户端用 GrowthBook 判断，因为 CCR 每次启动都是全新容器，GB 缓存未预热，客户端判断永远得到默认值 `false`。
- **`CLAUDE_CODE_REMOTE_SESSION_ID`**：会话 ID，用于构造 Basic Auth 的 `sessionId` 部分。缺失则记录警告并禁用代理。

#### 2. 会话令牌管理 (`readToken`)

- **路径**：默认 `/run/ccr/session_token`，测试可通过 `opts.tokenPath` 覆盖。
- **读取逻辑**：`utf8` 读取后 `.trim()`，空字符串视为无效。
- **错误处理**：`ENOENT` 静默返回 `null`；其他错误记录 `warn` 并返回 `null`。
- **删除时机**：仅在 `relay` 成功启动并注册清理后才 `unlink(tokenPath)`。这样若 CA 下载或 `listen()` 失败，supervisor 重启容器时 token 文件仍在，可重试初始化。

#### 3. 进程防转储保护 (`setNonDumpable`)

- **机制**：通过 `bun:ffi` 动态加载 `libc.so.6`，调用 `prctl(PR_SET_DUMPABLE, 4, 0, 0, 0, 0)`。
- **目的**：阻止 prompt-injected 的 `gdb -p $PPID` 或 `/proc/<pid>/mem` 读取堆内存中的 token。
- **限制**：仅 Linux + Bun 有效；Node 环境或 macOS 静默跳过。
- **容错**：`try/catch` 包裹，任何失败仅记录 `warn`，不影响后续初始化。

#### 4. CA 证书下载与捆绑 (`downloadCaBundle`)

- **端点**：`GET ${baseUrl}/v1/code/upstreamproxy/ca-cert`
- **baseUrl 来源**：
  1. `opts.ccrBaseUrl`（测试覆盖）
  2. `process.env.ANTHROPIC_BASE_URL`（由 `StartupContext` 注入，正确值）
  3. 回退 `'https://api.anthropic.com'`
  
  注释特别指出不能使用 `getOauthConfig()`，因为后者基于 `USER_TYPE` 和 `USE_LOCAL/STAGING_OAUTH`，在容器中未设置，永远返回 prod URL，导致 CA 下载 404。

- **超时**：`AbortSignal.timeout(5000)`，防止 Bun 默认无超时的 `fetch` 永久阻塞 CLI 启动。
- **输出**：`opts.caBundlePath ?? join(homedir(), '.ccr', 'ca-bundle.crt')`
- **拼接内容**：`systemCa + '\n' + ccrCa`。先写系统证书再写 CCR 代理证书，确保工具链能同时信任两者。

#### 5. 本地中继启动与清理注册

- **WebSocket URL**：`baseUrl.replace(/^http/, 'ws') + '/v1/code/upstreamproxy/ws'`。例如 `https://api.anthropic.com` → `wss://api.anthropic.com/v1/code/upstreamproxy/ws`。
- **Relay 启动**：调用 `startUpstreamProxyRelay({ wsUrl, sessionId, token })`（来自 `relay.ts`）。
- **清理注册**：`registerCleanup(async () => relay.stop())`，确保 CLI 优雅关闭时释放 TCP 端口。
- **状态更新**：`state = { enabled: true, port: relay.port, caBundlePath }`。

#### 6. 子进程环境变量注入 (`getUpstreamProxyEnv`)

- **代理地址**：`http://127.0.0.1:${state.port}`。注意是 `http://` 而非 `https://`，因为 relay 本身只处理 HTTP CONNECT，TLS 发生在隧道内部。
- **NO_PROXY 列表**：覆盖回环地址、RFC1918 私有网段、IMDS (`169.254.0.0/16`)、Anthropic API（三种格式兼容不同运行时）、GitHub、主流包管理器注册表。避免将这些流量无谓地绕进 relay，也防止 MITM 破坏 Anthropic API 的 Python `httpx`/`certifi` 信任链。
- **CA 证书变量**：
  - `SSL_CERT_FILE`（通用）
  - `NODE_EXTRA_CA_CERTS`（Node.js）
  - `REQUESTS_CA_BUNDLE`（Python requests）
  - `CURL_CA_BUNDLE`（curl）
- **子 CLI 继承逻辑**：若当前进程本身已继承 `HTTPS_PROXY` + `SSL_CERT_FILE`（即子 CLI 无法重新初始化 relay，但父进程的 relay 仍在运行），则透传相关变量，保证嵌套子进程也能走代理。

---

## 具体技术实现

### 关键数据结构

#### `UpstreamProxyState`（模块级状态）

```typescript
// upstreamproxy.ts:65-70
type UpstreamProxyState = {
  enabled: boolean      // 是否成功启用上游代理
  port?: number         // 本地 relay 监听端口
  caBundlePath?: string // CA 捆绑文件绝对路径
}

let state: UpstreamProxyState = { enabled: false }
```

该状态在 `initUpstreamProxy()` 成功后更新，在 `getUpstreamProxyEnv()` 中只读使用。测试可通过 `resetUpstreamProxyForTests()` 重置。

#### `NO_PROXY_LIST`（硬编码免代理列表）

```typescript
// upstreamproxy.ts:37-63
const NO_PROXY_LIST = [
  'localhost', '127.0.0.1', '::1',
  '169.254.0.0/16', '10.0.0.0/8', '172.16.0.0/12', '192.168.0.0/16',
  'anthropic.com', '.anthropic.com', '*.anthropic.com',
  'github.com', 'api.github.com', '*.github.com', '*.githubusercontent.com',
  'registry.npmjs.org', 'pypi.org', 'files.pythonhosted.org',
  'index.crates.io', 'proxy.golang.org',
].join(',')
```

**设计细节**：Anthropic 域名提供了三种形式：
- `*.anthropic.com` — Bun、curl、Go（glob 匹配）
- `.anthropic.com` — Python urllib/httpx（后缀匹配，会去掉前导点）
- `anthropic.com` — 根域名兜底

### 关键流程

#### 1. 初始化流程 (`initUpstreamProxy`)

```
开始
  │
  ▼
CLAUDE_CODE_REMOTE 为真？──否──▶ 返回 {enabled: false}
  │是
  ▼
CCR_UPSTREAM_PROXY_ENABLED 为真？──否──▶ 返回 {enabled: false}
  │是
  ▼
CLAUDE_CODE_REMOTE_SESSION_ID 存在？──否──▶ 记录 warn，返回
  │是
  ▼
读取 token 文件 ──失败/空──▶ 返回 {enabled: false}
  │成功
  ▼
调用 setNonDumpable() ──失败──▶ 记录 warn，继续
  │
  ▼
下载并拼接 CA Bundle ──失败──▶ 返回 {enabled: false}
  │成功
  ▼
构造 wsUrl 并启动 relay
  │
  ├── 失败 ──▶ 记录 warn，返回 {enabled: false}
  │
  ▼ 成功
注册 cleanup(relay.stop)
更新 state = {enabled: true, port, caBundlePath}
记录 info 日志
  │
  ▼
unlink(tokenPath) ──失败──▶ 记录 warn，继续
  │
  ▼
返回 state
```

#### 2. 子进程环境获取流程 (`getUpstreamProxyEnv`)

```
state.enabled 为真且 port/caBundlePath 存在？
  │是
  ▼
返回 {
  HTTPS_PROXY:  http://127.0.0.1:port,
  https_proxy:  http://127.0.0.1:port,
  NO_PROXY:     NO_PROXY_LIST,
  no_proxy:     NO_PROXY_LIST,
  SSL_CERT_FILE: caBundlePath,
  NODE_EXTRA_CA_CERTS: caBundlePath,
  REQUESTS_CA_BUNDLE:  caBundlePath,
  CURL_CA_BUNDLE:      caBundlePath,
}
  │否
  ▼
检查 process.env.HTTPS_PROXY && process.env.SSL_CERT_FILE 是否同时存在
  │是（子 CLI 继承父进程场景）
  ▼
透传已存在的相关变量（HTTPS_PROXY, https_proxy, NO_PROXY, no_proxy,
                     SSL_CERT_FILE, NODE_EXTRA_CA_CERTS,
                     REQUESTS_CA_BUNDLE, CURL_CA_BUNDLE）
  │否
  ▼
返回 {}
```

### FFI 调用细节 (`setNonDumpable`)

```typescript
// upstreamproxy.ts:225-252
const ffi = require('bun:ffi') as typeof import('bun:ffi')
const lib = ffi.dlopen('libc.so.6', {
  prctl: {
    args: ['int', 'u64', 'u64', 'u64', 'u64'],
    returns: 'int',
  },
} as const)
const PR_SET_DUMPABLE = 4
const rc = lib.symbols.prctl(PR_SET_DUMPABLE, 0n, 0n, 0n, 0n)
```

- `prctl` 是 Linux 系统调用，参数 1 为 `option`，参数 2 为 `arg2`（此处 `0` 表示不可 dump）。
- 返回值非零时记录 warn，但不中断初始化。

### CA 下载细节 (`downloadCaBundle`)

```typescript
// upstreamproxy.ts:254-285
const resp = await fetch(`${baseUrl}/v1/code/upstreamproxy/ca-cert`, {
  signal: AbortSignal.timeout(5000),
})
if (!resp.ok) { ... return false }
const ccrCa = await resp.text()
const systemCa = await readFile(systemCaPath, 'utf8').catch(() => '')
await mkdir(join(outPath, '..'), { recursive: true })
await writeFile(outPath, systemCa + '\n' + ccrCa, 'utf8')
return true
```

- 使用原生 `fetch`（Bun/Node 18+ 均支持）。
- `mkdir(..., { recursive: true })` 确保 `~/.ccr` 目录存在。
- 系统证书读取失败时以空字符串兜底，保证至少能写入 CCR CA。

---

## 关键代码路径与文件引用

### 核心函数位置

| 函数 | 文件:行 | 说明 |
|------|---------|------|
| `initUpstreamProxy` | `upstreamproxy.ts:79-153` | 主初始化入口 |
| `getUpstreamProxyEnv` | `upstreamproxy.ts:160-199` | 子进程环境变量生成 |
| `resetUpstreamProxyForTests` | `upstreamproxy.ts:202-204` | 测试状态重置 |
| `readToken` | `upstreamproxy.ts:206-218` | 读取并 trim session token |
| `setNonDumpable` | `upstreamproxy.ts:225-252` | `prctl` FFI 调用 |
| `downloadCaBundle` | `upstreamproxy.ts:254-285` | 下载并拼接 CA bundle |

### 调用方

- **`src/entrypoints/init.ts:167-183`**：动态导入并调用 `initUpstreamProxy()`，同时注册 `getUpstreamProxyEnv` 到 `subprocessEnv.ts`。

### 被调用方 / 依赖方

- **`src/upstreamproxy/relay.ts`**：`startUpstreamProxyRelay()` 启动本地 TCP/WebSocket 中继。
- **`src/utils/subprocessEnv.ts`**：`registerUpstreamProxyEnvFn(getUpstreamProxyEnv)` 将代理环境注入能力注册到全局子进程环境生成器。
- **`src/utils/cleanupRegistry.ts`**：`registerCleanup()` 注册 `relay.stop()` 清理函数。
- **`src/utils/debug.ts`**：`logForDebugging()` 用于所有日志输出。
- **`src/utils/envUtils.ts`**：`isEnvTruthy()` 用于布尔型环境变量解析。
- **`src/utils/errors.ts`**：`isENOENT()` 用于区分 token 文件不存在与其他 IO 错误。

### 子进程环境消费方（间接）

`subprocessEnv()` 的调用方遍布代码库，主要包括：
- Bash 命令执行（`src/utils/Shell.ts`）
- MCP stdio 服务器启动（`src/services/mcp/client.ts`）
- LSP 服务器启动（`src/services/lsp/LSPClient.ts`）
- Shell Hooks 执行（`src/utils/hooks.ts`）
- Shell Snapshot（`src/utils/bash/ShellSnapshot.ts`）

---

## 依赖与外部交互

### 内部依赖

| 模块 | 导入符号 | 用途 |
|------|----------|------|
| `src/upstreamproxy/relay.js` | `startUpstreamProxyRelay` | 启动本地 CONNECT-over-WebSocket 中继 |
| `src/utils/cleanupRegistry.js` | `registerCleanup` | 注册 relay 关闭的清理回调 |
| `src/utils/debug.js` | `logForDebugging` | 全链路日志记录 |
| `src/utils/envUtils.js` | `isEnvTruthy` | 解析环境变量布尔值 |
| `src/utils/errors.js` | `isENOENT` | 区分 ENOENT 与其他文件错误 |

### Node.js 内置模块

| 模块 | 用途 |
|------|------|
| `fs/promises` | `readFile`, `writeFile`, `mkdir`, `unlink` |
| `os` | `homedir()` |
| `path` | `join()` |

### 外部 API 交互

| 端点 | 方法 | 用途 |
|------|------|------|
| `GET /v1/code/upstreamproxy/ca-cert` | HTTP(S) | 下载上游代理 CA 证书 |
| `wss://<baseUrl>/v1/code/upstreamproxy/ws` | WebSocket | relay.ts 建立的隧道端点 |

### 环境变量

#### 输入变量

| 变量 | 说明 |
|------|------|
| `CLAUDE_CODE_REMOTE` | CCR 环境标识，必须为真值才进入初始化 |
| `CCR_UPSTREAM_PROXY_ENABLED` | 服务端 GrowthBook 开关 |
| `CLAUDE_CODE_REMOTE_SESSION_ID` | 会话 ID，用于 Basic Auth |
| `ANTHROPIC_BASE_URL` | API 基础 URL，用于构造 CA 下载地址和 WS URL |

#### 输出变量（子进程环境）

| 变量 | 说明 |
|------|------|
| `HTTPS_PROXY` / `https_proxy` | 本地 relay 地址 `http://127.0.0.1:<port>` |
| `NO_PROXY` / `no_proxy` | 硬编码免代理列表 |
| `SSL_CERT_FILE` | CA 捆绑文件路径 |
| `NODE_EXTRA_CA_CERTS` | Node.js 额外 CA 证书 |
| `REQUESTS_CA_BUNDLE` | Python requests CA 证书 |
| `CURL_CA_BUNDLE` | curl CA 证书 |

---

## 风险、边界与改进建议

### 已知风险

| 风险 | 等级 | 说明 |
|------|------|------|
| **Token 堆内存泄露** | 中 | `prctl` 仅提供有限保护，root 用户或具备 `CAP_SYS_PTRACE` 的进程仍可读取内存。且 Node 运行时完全不支持 `prctl` FFI。 |
| **CA 下载单点阻塞** | 低 | 5 秒超时 + fail-open，失败仅禁用代理，不影响主流程。但若网络抖动频繁，可能导致代理间歇性不可用。 |
| **硬编码 NO_PROXY 过时** | 低 | 随着新包管理器或新 API 端点的引入，免代理列表可能遗漏，导致不必要的 MITM 或连接失败。 |
| **子 CLI 环境继承不完整** | 低 | 子 CLI 进程透传逻辑仅检查 `HTTPS_PROXY && SSL_CERT_FILE`，若父进程只设置了其中之一，子 CLI 可能无法正确信任 CA 或走代理。 |
| **Bun FFI 可移植性** | 低 | `bun:ffi` 是 Bun 专属 API，若未来生产环境迁移到 Node，则 `setNonDumpable` 完全失效。 |

### 边界情况

#### 已处理的边界

1. **Token 文件不存在**：`isENOENT` 判断，静默返回 `null`，不记录错误（因为文件不存在是预期中的禁用条件）。
2. **CA 下载失败**：任何网络错误、非 2xx 状态码、超时均返回 `false`，上游代理被禁用。
3. **Relay 启动失败**：`try/catch` 捕获，`warn` 日志，返回 `{ enabled: false }`。
4. **Token 删除失败**：`unlink` 使用 `.catch()` 兜底，仅记录 `warn`，不中断流程。
5. **系统 CA 文件缺失**：`readFile(systemCaPath).catch(() => '')`，以空字符串兜底，确保至少写入 CCR CA。
6. **子 CLI 嵌套场景**：通过检查 `process.env.HTTPS_PROXY && process.env.SSL_CERT_FILE` 实现变量继承。

#### 未处理或潜在的边界

1. **CA 证书热轮换**：初始化后 `ca-bundle.crt` 不再更新，若服务端 CA 轮换，需重启 CLI 才能生效。
2. **NO_PROXY CIDR 支持不完整**：列表中包含 `10.0.0.0/8` 等 CIDR 写法，但不同工具（curl、Python、Node）对 CIDR 的 NO_PROXY 支持程度不一，部分工具可能仅做字符串前缀匹配。
3. **IPv6 代理地址缺失**：`HTTPS_PROXY` 硬编码 `127.0.0.1`，未提供 `::1` 备选，纯 IPv6 环境子进程可能无法连接。
4. **并发初始化**：`state` 是模块级单例，若 `initUpstreamProxy` 被并发调用（理论上不应发生），可能存在竞态条件。

### 改进建议

#### 高优先级

1. **Node 运行时进程保护**
   - 当前：`setNonDumpable` 仅在 Bun + Linux 下有效。
   - 建议：评估在 Node + Linux 下通过 `node-ffi-napi` 或 `child_process.exec('prctl ...')` 实现同等保护，或至少通过 `process.setuid()` 降低被 dump 风险。

2. **CA 证书过期/轮换监控**
   - 当前：一次性下载，终身使用。
   - 建议：在长时间运行的会话中，增加定时任务（如每 1 小时）重新拉取 CA 证书并比对指纹，变化时自动重写 `ca-bundle.crt`。

3. **NO_PROXY 动态扩展能力**
   - 当前：硬编码 TypeScript 数组。
   - 建议：允许通过环境变量 `CCR_UPSTREAM_PROXY_NO_PROXY_EXTRA` 在运行时追加条目，无需发版即可适配新的内部服务域名。

#### 中优先级

4. **初始化指标与可观测性**
   - 当前：仅 debug/warn 日志。
   - 建议：增加 OpenTelemetry Counter/Histogram，记录 `upstreamproxy.init.attempts`、`upstreamproxy.init.success`、`upstreamproxy.init.duration_ms`、`upstreamproxy.ca_download.duration_ms`。

5. **子 CLI 继承逻辑完善**
   - 当前：仅当 `HTTPS_PROXY && SSL_CERT_FILE` 同时存在时才透传。
   - 建议：降低为单变量存在即透传，或维护一个明确的“代理相关变量集合”，任一存在即全部透传，避免部分变量缺失导致子进程行为异常。

6. **IPv6 双栈支持**
   - 当前：`127.0.0.1` 硬编码。
   - 建议：relay 同时监听 `::1`，`getUpstreamProxyEnv` 根据可用地址返回对应代理 URL，或返回 `http://localhost:port` 让系统自行解析。

#### 低优先级

7. **Token 内存清零**
   - 当前：token 以 JavaScript string 存在于堆中，GC 后仍可能残留于内存碎片。
   - 建议：在 relay 成功启动后，将 token 从所有局部变量中释放，并考虑使用 `Buffer` 并在完成后 `.fill(0)`（虽然 V8/Bun 的 GC 和字符串驻留使彻底擦除困难）。

8. **配置 Schema 校验**
   - 当前：对 `ANTHROPIC_BASE_URL` 等变量无格式校验。
   - 建议：增加 URL 解析校验，防止因拼写错误导致 CA 下载 404 或 WS 连接失败。
