# src/utils/preflightChecks.tsx 研究文档

## 场景与职责

`preflightChecks.tsx` 是 Claude Code 首次启动（Onboarding）时的网络连通性检查组件。当用户通过 OAuth 流程登录前，系统需要确认能够正常访问 Anthropic 的 API 端点和 OAuth 服务。若检查失败，会给出 SSL 错误提示、网络配置文档链接或支持国家列表链接，并在严重错误时退出进程。

## 功能点目的

1. **端点连通性检查（`checkEndpoints`）**
   - 并发检查两个端点：
     - `${BASE_API_URL}/api/hello`
     - `${TOKEN_URL.origin}/v1/oauth/hello`
   - 使用 `axios.get` 发送请求，携带 `User-Agent`。
   - 任一端点非 200 或抛错即视为失败，返回包含 hostname、状态码/错误信息、`sslHint`（若适用）的结果。
   - 失败时通过 `logEvent` 上报 `tengu_preflight_check_failed` 事件，标记是否为连接错误、是否有错误消息、是否为 SSL 错误。

2. **React 组件状态管理（`PreflightStep`）**
   - 使用 React state 跟踪 `result`（检查结果）和 `isChecking`（是否仍在检查）。
   - `useTimeout(1000)` 控制 spinner 的显示时机：检查超过 1 秒才显示 "Checking connectivity..."，避免闪屏。
   - 两个 `useEffect`：
     - 第一个在挂载时异步执行 `checkEndpoints`。
     - 第二个在 `result` 变化时处理后续：成功则调用 `onSuccess()`；失败则设置 100ms 定时器后 `process.exit(1)`。

3. **错误提示渲染**
   - 失败时显示红色错误文本：`Unable to connect to Anthropic services` + 具体错误。
   - 若 `sslHint` 存在，显示 SSL 提示 + 网络配置文档链接。
   - 若无 SSL 提示，显示通用网络检查建议 + 支持国家列表链接。

## 具体技术实现

- 文件为 `.tsx`，但内容已被 React Compiler 编译（可见 `_c(12)` 等缓存数组操作和 `t0`/`t1` 等临时变量）。
- `axios` 直接导入，无代理配置复用；不过 Onboarding 发生在 `configureGlobalAgents()` 之前，代理可能尚未配置。
- `getSSLErrorHint` 从 `src/services/api/errorUtils.js` 导入，负责把底层 TLS/证书错误翻译为用户可读的提示。
- `logError` 用于捕获 `checkEndpoints` 内部意外抛错。
- `process.exit(1)` 被包装在 `_temp()` 中，通过 `setTimeout` 延迟 100ms 执行，给 React 一帧时间完成错误渲染。

## 关键代码路径与文件引用

- **本文件**：`src/utils/preflightChecks.tsx`
- **调用方**：
  - `src/components/Onboarding.tsx` — Onboarding 流程的第一步（当 `oauthEnabled` 为 true 时）。
- **依赖**：
  - `axios` — HTTP 请求。
  - `react` — `useEffect`、`useState`。
  - `src/services/analytics/index.js` — `logEvent`。
  - `src/components/Spinner.js` — 加载动画。
  - `src/constants/oauth.js` — `getOauthConfig`。
  - `src/hooks/useTimeout.js` — 延迟显示 spinner。
  - `src/ink.js` — `Box`、`Text`。
  - `src/services/api/errorUtils.js` — `getSSLErrorHint`。
  - `src/utils/http.js` — `getUserAgent`。
  - `src/utils/log.js` — `logError`。

## 依赖与外部交互

- 向 Anthropic API 和 OAuth 服务端点发送 HTTP GET 请求。
- 通过 Statsig（`logEvent`）上报预检失败事件。
- 引用外部文档链接：`https://code.claude.com/docs/en/network-config`、`https://anthropic.com/supported-countries`。

## 风险、边界与改进建议

1. **进程直接退出**：`process.exit(1)` 是硬退出，不会触发 Claude Code 正常的 graceful shutdown 流程（如保存会话、关闭 MCP 连接等）。虽然 Onboarding 发生在会话正式开始前，风险较低，但如果未来预检被复用到其他场景，可能造成数据丢失。
2. **代理未配置**：Onboarding 的预检请求使用裸 `axios.get`，没有显式应用 `proxy.ts` 中的代理配置。若用户处于必须走代理才能访问外网的环境，预检会失败，而后续正常流程（已配置代理）可能成功，导致误报。
3. **React Compiler 编译后代码的可读性**：当前文件是编译产物，原始 TypeScript 源码可能在 `src/` 的其他位置（或构建时内联）。编译后的缓存数组逻辑（`_c`）使得调试和代码审查困难，错误堆栈中的行号也可能与源码不对应。
4. **定时器清理**：失败时的 `setTimeout(_temp, 100)` 在 `useEffect` 的 cleanup 函数中被 `clearTimeout` 处理，但如果组件在 100ms 内 unmount（理论上 Onboarding 流程中不太可能），定时器仍可能触发 `process.exit(1)`。
5. **改进建议**：
   - 在 `checkEndpoints` 中复用 `createAxiosInstance()`（来自 `proxy.ts`）或至少读取 `HTTPS_PROXY` 环境变量，确保代理环境也能通过预检。
   - 将 `process.exit(1)` 替换为向父组件报告失败状态，由 `Onboarding.tsx` 决定是否退出或允许用户重试，提升用户体验。
   - 若保留编译产物在仓库中，建议同时保留 source map 并确保构建流程可复现，方便问题定位。
