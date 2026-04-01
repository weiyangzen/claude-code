# 研究文档：src/utils/userAgent.ts

## 场景与职责

本模块是 Claude Code 的 **User-Agent 字符串生成器**。它被设计为**零依赖**的叶子模块，以便 SDK 桥接代码（`bridge`）、CLI transport 层、以及各种 API 客户端都能安全导入，而不会连带拉入庞大的认证/配置依赖树（如 `auth.ts` → `oauth` → `keychain` 等）。

User-Agent 用于所有对外 HTTP 请求（Anthropic API、内部服务、遥测端点等），帮助服务端识别客户端版本、进行流量分析和兼容性决策。

## 功能点目的

| 导出符号 | 目的 |
|---------|------|
| `getClaudeCodeUserAgent()` | 返回格式为 `claude-code/${MACRO.VERSION}` 的 User-Agent 字符串。 |

## 具体技术实现

- 单函数，10 行代码，无任何 import。
- `MACRO.VERSION` 是在构建时通过 bundler（Bun）的 `--define` 注入的全局常量，对应 `package.json` 中的版本号。
- 返回值固定格式：`claude-code/x.y.z`（例如 `claude-code/0.2.45`）。

## 关键代码路径与文件引用

- **主实现**：`src/utils/userAgent.ts`（10 行）
- **API 调用层**：
  - `src/services/api/bootstrap.ts`
  - `src/services/api/firstTokenDate.ts`
  - `src/services/api/grove.ts`
  - `src/services/api/metricsOptOut.ts`
  - `src/services/api/usage.ts`
- **内部服务**：
  - `src/services/teamMemorySync/index.ts`
  - `src/services/remoteManagedSettings/index.ts`
  - `src/services/policyLimits/index.ts`
  - `src/services/settingsSync/index.ts`
- **Analytics**：
  - `src/services/analytics/firstPartyEventLoggingExporter.ts`
  - `src/utils/telemetry/bigqueryExporter.ts`
- **CLI Transport**：
  - `src/cli/transports/SSETransport.ts`
  - `src/cli/transports/ccrClient.ts`
- **HTTP 工具**：
  - `src/utils/http.ts`

## 依赖与外部交互

- 无内部模块依赖。
- 无外部 npm 依赖。
- 依赖构建时注入的全局常量 `MACRO.VERSION`。

## 风险、边界与改进建议

### 风险

1. **构建注入缺失导致运行时崩溃**：若构建脚本未正确注入 `MACRO.VERSION`（如本地开发直接运行未打包的源码、或 bundler 配置变更），访问 `MACRO.VERSION` 会抛出 `MACRO is not defined` 的 `ReferenceError`，导致所有依赖该模块的 HTTP 请求初始化失败。
2. **版本信息过于单一**：当前仅包含应用名称和版本号，缺少平台（macOS/Windows/Linux）、架构（x64/arm64）等上下文。服务端在排查特定平台问题时，需要结合其他请求头或 payload 字段才能定位。
3. **无运行时版本校验**：函数不检查 `MACRO.VERSION` 的格式是否合法（如是否为语义化版本字符串），可能将 `'unknown'` 或空字符串发送到服务端。

### 边界

- ** intentionally 极简**：为了保持零依赖，模块拒绝引入任何平台检测或配置读取逻辑。这些信息的补充由调用方通过其他请求头（如 `X-Claude-Code-Platform`）负责。
- **无缓存**：虽然函数是纯函数且计算成本极低，但部分高频调用方（如 `http.ts`）可能会在每次请求时都重新调用。这本身没有性能问题，但在极端高频场景下（如批量 telemetry 上报）可考虑由调用方自行缓存。

### 改进建议

1. **增加运行时 fallback**：将函数实现改为：
   ```ts
   export function getClaudeCodeUserAgent(): string {
     const version = typeof MACRO !== 'undefined' ? MACRO.VERSION : 'unknown'
     return `claude-code/${version}`
   }
   ```
   这样即使构建注入缺失，也不会抛异常，而是降级为 `claude-code/unknown`。
2. **扩展 User-Agent 格式**：在保持本模块零依赖的前提下，可考虑通过构建时额外注入 `MACRO.PLATFORM` 和 `MACRO.ARCH`，将格式扩展为 `claude-code/0.2.45 (darwin; arm64)`，从而在不增加运行时依赖的情况下传递更多信息。
3. **版本格式校验（开发模式）**：在 `process.env.NODE_ENV === 'development'` 时增加断言，确保 `MACRO.VERSION` 符合 semver 格式，提前发现构建配置错误。
4. **调用方缓存建议**：在 `src/utils/http.ts` 等高频调用点将 `getClaudeCodeUserAgent()` 的结果缓存到模块级常量，避免重复字符串拼接（虽然现代 JS 引擎对此优化已很好，但仍是良好实践）。
