# src/utils/caCerts.ts 深入研究

## 场景与职责

`caCerts.ts` 负责为 Claude Code 的 TLS 连接加载并组装自定义 CA 证书。由于 Node.js/Bun 在设置 `ca` 选项时会**完全替换**默认证书存储，因此该模块必须保证：
- 当用户未配置任何自定义 CA 时，直接返回 `undefined`，让运行时使用原生默认行为。
- 当用户配置了 `NODE_EXTRA_CA_CERTS` 或 `--use-system-ca` 时，将自定义证书与系统/内置 Mozilla CA **合并返回**，避免仅信任自定义证书而导致连接失败。

该模块被 `proxy.ts`、`mtls.ts` 和 `telemetry/instrumentation.ts` 等网络层模块调用，是 HTTPS、WebSocket、mTLS 的证书基础。

## 功能点目的

| 功能 | 目的 |
|------|------|
| `getCACertificates()` | 按需加载并缓存 CA 证书列表；返回 `string[] \| undefined` |
| `clearCACertsCache()` | 在环境变量或配置变更后清除 memoize 缓存 |

## 具体技术实现

### 配置来源
- `NODE_EXTRA_CA_CERTS`：环境变量，指向额外的 PEM 证书文件。
- `--use-system-ca` / `--use-openssl-ca`：Node/Bun 启动参数，指示使用操作系统 CA 存储而非内置 Mozilla CA。

### 行为矩阵
| `useSystemCA` | `extraCertsPath` | 返回内容 |
|---------------|------------------|----------|
| false | 未设置 | `undefined`（使用运行时默认） |
| false | 已设置 | 内置 Mozilla CAs + extra cert 文件内容 |
| true | 未设置 | 系统 CAs（Bun API）或内置 CAs（Node fallback） |
| true | 已设置 | 系统 CAs + extra cert 文件内容 |

### 延迟加载优化
```ts
const tls = require('tls') as typeof import('tls')
```
- Bun 的 `node:tls` 模块在导入时会 eagerly 实例化约 150 个 Mozilla 根证书（~750KB 堆内存）。
- 由于大多数用户不会配置自定义 CA，模块在 `getCACertificates` 内部才 `require('tls')`，使常见路径零额外开销。

### 系统 CA 加载（Bun 专属）
```ts
const systemCAs = (tls as any).getCACertificates?.('system')
```
- Bun 提供了 `tls.getCACertificates('system')` 扩展 API 来读取操作系统证书存储。
- 在纯 Node.js 环境下该 API 不存在，此时若用户仅设置了 `--use-system-ca` 且无 extra certs，模块返回 `undefined`，让 Node.js 原生处理 `--use-system-ca`。

### 缓存
- 使用 `lodash-es/memoize` 对 `getCACertificates` 做无参缓存。
- 缓存生命周期为进程级；配置热更新后需调用 `clearCACertsCache()`。

## 关键代码路径与文件引用

```
src/utils/proxy.ts
  └── getCACertificates()
      [HTTPS 代理连接时注入自定义 CA]

src/utils/mtls.ts
  └── getCACertificates()
      [mTLS Agent 与 WebSocket TLS 选项组装]

src/utils/telemetry/instrumentation.ts
  └── getCACertificates()
      [遥测上报的 TLS 配置]

src/utils/managedEnv.ts
  └── clearCACertsCache()
      [环境变量变更后刷新证书缓存]
```

### 依赖模块
- `src/utils/debug.js` — `logForDebugging`
- `src/utils/envUtils.js` — `hasNodeOption`
- `src/utils/fsOperations.js` — `getFsImplementation().readFileSync`
- `node:tls` — 延迟 require

## 依赖与外部交互

| 外部实体 | 交互方式 | 说明 |
|---------|---------|------|
| 操作系统 CA 存储 | `tls.getCACertificates('system')` (Bun) | 读取系统信任根证书 |
| Mozilla 内置 CA | `tls.rootCertificates` | Node.js/Bun 提供的内置根证书列表 |
| 用户自定义证书文件 | `fs.readFileSync(process.env.NODE_EXTRA_CA_CERTS)` | 追加到 CA 列表 |
| 环境变量 | `process.env.NODE_EXTRA_CA_CERTS` / `NODE_OPTIONS` | 通过 `hasNodeOption` 解析启动参数 |

## 风险、边界与改进建议

### 风险
1. **Bun 扩展 API 的 Node.js 不可移植性**：`tls.getCACertificates` 是 Bun 独有，若未来需要纯 Node.js 发行版，系统 CA 加载路径需重写。
2. **`readFileSync` 阻塞事件循环**：虽然证书加载通常只在连接初始化时发生，但大证书文件的同步读取在极端场景下可能产生短暂卡顿。
3. **证书格式无校验**：模块仅做原始文本读取，不验证 PEM 格式正确性；若用户提供了损坏的证书文件，错误会延迟到 TLS 握手时才暴露。
4. **memoize 缓存无依赖追踪**：`getCACertificates` 的缓存不感知 `NODE_EXTRA_CA_CERTS` 文件内容的磁盘变更，仅能通过显式 `clearCACertsCache()` 刷新。

### 边界
- 该模块**不读取任何配置文件**（如 `settings.json`），只依赖 `process.env.NODE_EXTRA_CA_CERTS`。配置到环境变量的映射由 `caCertsConfig.ts` 在 CLI 初始化时完成。
- 返回 `undefined` 时，调用方应**不设置**任何 `ca` 选项，以保留运行时的默认证书行为。
- `getCACertificates` 返回的是字符串数组（PEM 文本），而非 `Buffer` 或 `crypto` 对象；调用方（如 `mtls.ts`）直接将其拼入 `AgentOptions.ca`。

### 改进建议
1. **证书格式预校验**：在读取后使用简单的正则 `/-----BEGIN CERTIFICATE-----/` 检查文件内容是否包含至少一个有效 PEM 块，提前报错并给出清晰的文件路径提示。
2. **异步加载变体**：提供 `getCACertificatesAsync()` 使用 `readFile` 替代 `readFileSync`，供非阻塞敏感路径（如 UI 初始化）选用。
3. **文件变更监听（可选）**：在 `clearCACertsCache` 之外，考虑通过 `fs.watchFile` 或配置变更事件自动刷新缓存。
4. **Node.js 系统 CA 回退增强**：当前 Node.js 环境下 `--use-system-ca` 直接返回 `undefined` 依赖 Node 原生处理；若需要显式控制系统 CA 列表（如与代理一起使用时），可引入 `node-system-ca` 等库做跨平台系统 CA 读取。
