# 研究文档：src/utils/hooks/ssrfGuard.ts

> 生成时间：2026-04-01  
> 研究范围：代码、调用方、网络协议、安全策略、边界条件  
> 文件大小：约 8.7 KB

---

## 1. 场景与职责

`ssrfGuard.ts` 是 Claude Code HTTP Hook 执行链路中的**服务器端请求伪造（SSRF）防护层**。其职责是：在 HTTP Hook 通过 axios 发起出站请求时，拦截对**私有地址、链路本地地址及其他不可路由地址**的访问，防止恶意或误配置的 HTTP Hook 触及云元数据端点（如 AWS/Azure/GCP 的 `169.254.169.254`）或企业内部基础设施。

关键设计取舍：
- **允许回环（Loopback）**：`127.0.0.0/8` 与 `::1` 被显式放行，因为本地开发策略服务器（local dev policy servers）是 HTTP Hook 的主要使用场景之一。
- **代理感知绕过**：当系统配置了全局代理（`HTTP_PROXY`/`HTTPS_PROXY`）或启用了沙箱网络代理时，SSRF Guard 自动失效。原因是代理服务器会代为执行 DNS 解析，此时 Guard 验证的将是代理服务器的 IP 而非目标主机，可能误杀合法的代理连接。

---

## 2. 功能点目的

| 功能点 | 目的 |
|--------|------|
| `isBlockedAddress` | 判断一个 IP 字符串是否属于被禁止的地址范围，支持 IPv4 与 IPv6。 |
| `isBlockedV4` | IPv4 专用的范围检查：私有网段、链路本地、共享地址空间（CGNAT）等。 |
| `isBlockedV6` | IPv6 专用的范围检查：未指定地址、唯一本地地址、链路本地、IPv4 映射地址。 |
| `expandIPv6Groups` | 将任意合法 IPv6 表示法（含 `::` 压缩与尾部点分十进制）展开为 8 个 16 位十六进制数组。 |
| `extractMappedIPv4` | 从 IPv4-mapped IPv6 地址（如 `::ffff:169.254.169.254`）中提取内嵌的 IPv4 地址。 |
| `ssrfGuardedLookup` | 提供 axios 兼容的 `lookup` 函数，在 DNS 解析后或 IP 字面量直接验证地址，阻止非法连接。 |
| `ssrfError` | 构造带自定义错误码 `ERR_HTTP_HOOK_BLOCKED_ADDRESS` 的异常对象，供上游日志与错误处理使用。 |

---

## 3. 具体技术实现

### 3.1 被阻止的 IPv4 范围

| CIDR | 说明 |
|------|------|
| `0.0.0.0/8` | "this" network（未指定/当前网络） |
| `10.0.0.0/8` | RFC 1918 私有地址 |
| `100.64.0.0/10` | RFC 6598 共享地址空间 / CGNAT；注释特别指出阿里云等云厂商在此范围放置元数据端点（如 `100.100.100.200`） |
| `169.254.0.0/16` | 链路本地（Link-Local），包含著名的 `169.254.169.254` 云元数据端点 |
| `172.16.0.0/12` | RFC 1918 私有地址 |
| `192.168.0.0/16` | RFC 1918 私有地址 |

**显式放行**：`127.0.0.0/8`（`a === 127` 直接返回 `false`）。

实现方式采用**手动 octet 判断**而非引入 CIDR 库：
```ts
function isBlockedV4(address: string): boolean {
  const parts = address.split('.').map(Number)
  const [a, b] = parts
  // ... 合法性校验后逐个范围判断
  if (a === 127) return false
  if (a === 0) return true
  if (a === 10) return true
  if (a === 169 && b === 254) return true
  if (a === 172 && b >= 16 && b <= 31) return true
  if (a === 100 && b >= 64 && b <= 127) return true
  if (a === 192 && b === 168) return true
  return false
}
```

### 3.2 被阻止的 IPv6 范围

| 地址/前缀 | 说明 |
|-----------|------|
| `::` | 未指定地址 |
| `fc00::/7` | 唯一本地地址（ULA） |
| `fe80::/10` | 链路本地地址 |
| `::ffff:<blocked_v4>` | IPv4 映射地址，若内嵌 IPv4 被阻止则整体被阻止 |

**显式放行**：`::1`（回环）。

### 3.3 IPv4-mapped IPv6 的提取与防护

这是 SSRF 防护中极易被忽视的绕过点。攻击者可能使用：
- `::ffff:169.254.169.254`
- `::ffff:a9fe:a9fe`
- `0:0:0:0:0:ffff:169.254.169.254`

`extractMappedIPv4` 的实现流程：
1. 调用 `expandIPv6Groups(addr)` 将地址规范化为 8 个 16 位整数数组；
2. 检查前 6 个元素是否为 `0, 0, 0, 0, 0, 0xffff`；
3. 若是，将第 7、8 个元素拆分为 4 个十进制 octet，返回点分十进制 IPv4 字符串；
4. `isBlockedV6` 在检测到 mapped IPv4 后，委托 `isBlockedV4` 判断。

### 3.4 IPv6 规范化：`expandIPv6Groups`

该函数处理所有标准 IPv6 表示法：
1. **尾部点分十进制**：若地址包含 `.`，则定位最后一个 `:`，将尾部作为 IPv4 解析为两个 16 位 hextet，并从原字符串中移除尾部；
2. **`::` 展开**：找到唯一的 `::`，计算需要填充的零组数量 `fill = 8 - tailHextets.length - head.length - tail.length`；
3. **合法性校验**：所有 hextet 必须在 `[0, 0xffff]` 范围内，且最终长度为 8。

### 3.5 axios 集成：`ssrfGuardedLookup`

签名严格匹配 axios 的 `lookup` 配置选项（注意：与 Node.js 原生的 `dns.lookup` 签名不同）：

```ts
export function ssrfGuardedLookup(
  hostname: string,
  options: object,
  callback: (err, address, family?) => void,
): void
```

执行逻辑：
1. **检测 `options.all`**：决定返回单个地址还是地址数组；
2. **IP 字面量短路**：若 `isIP(hostname) !== 0`，直接调用 `isBlockedAddress(hostname)` 判断，无需 DNS；
3. **DNS 解析**：调用 `dnsLookup(hostname, { all: true }, ...)` 获取全部解析结果；
4. **逐地址校验**：遍历 `addresses`，任一地址被阻止则立即调用 `callback(ssrfError(...), '')` 并返回；
5. **无结果处理**：若 `addresses` 为空，构造 `ENOTFOUND` 错误返回；
6. **正常返回**：按 `options.all` 返回数组或单个地址，family 统一规范化为 `4` 或 `6`。

> 关键点：该函数作为 axios 的 `lookup` 选项传入，意味着**验证后的 IP 就是 socket 实际连接的 IP**，不存在 DNS Rebinding 的时间窗口（validation 与 connection 之间没有窗口期）。

### 3.6 代理场景下的失效策略

在 `execHttpHook.ts` 中：
```ts
const envProxyActive =
  !sandboxProxy &&
  getProxyUrl() !== undefined &&
  !shouldBypassProxy(hook.url)

lookup: sandboxProxy || envProxyActive ? undefined : ssrfGuardedLookup,
```

- **沙箱代理**：`SandboxManager` 提供独立的网络代理，其自身维护域名白名单，返回 403 阻止非法域名；
- **环境变量代理**：企业内网常见代理位于私有 IP（如 `10.0.0.1:3128`），若启用 Guard 会误将代理 IP 判定为 blocked；因此直接跳过 Guard，信任代理后的目标访问控制。

---

## 4. 关键代码路径与文件引用

### 4.1 本文件导出的符号

| 符号 | 类型 | 说明 |
|------|------|------|
| `isBlockedAddress` | function | 入口函数，按 IP 版本分发到 v4/v6 检查 |
| `ssrfGuardedLookup` | function | axios 兼容的 lookup 替代函数 |

### 4.2 上游调用方

| 文件 | 调用符号 | 用途 |
|------|----------|------|
| `src/utils/hooks/execHttpHook.ts` | `ssrfGuardedLookup` | 作为 axios `lookup` 配置项传入，执行 HTTP Hook 请求前的 IP 校验。 |

### 4.3 无直接下游消费方

`ssrfGuardedLookup` 和 `isBlockedAddress` 均为纯工具函数，除了 `execHttpHook.ts` 外，当前仓库中无其他直接调用点。但 `isBlockedAddress` 作为导出函数，理论上可被任何需要 IP 范围检查的新模块复用。

---

## 5. 依赖与外部交互

### 5.1 直接依赖

| 模块 | 用途 |
|------|------|
| `axios` | 类型导入 `AddressFamily` 与 `LookupAddress`，用于签名兼容 |
| `dns` | Node.js 内置模块，提供 `dnsLookup` 解析主机名 |
| `net` | Node.js 内置模块，提供 `isIP` 判断字符串是否为合法 IP 字面量 |

### 5.2 运行时外部交互

- **操作系统 DNS**：`dns.lookup` 实际调用操作系统层面的 getaddrinfo；
- **axios 内部网络栈**：`ssrfGuardedLookup` 作为回调被 axios 在建立 TCP 连接前调用，其返回值直接影响 axios 使用的目标 IP；
- **沙箱代理 / 系统代理**：当代理生效时，Guard 被跳过，网络访问控制由代理层承担。

---

## 6. 风险、边界与改进建议

### 6.1 风险

1. **代理绕过 = 完全失效**
   - 一旦用户环境配置了 `HTTP_PROXY` 或沙箱代理启用，SSRF Guard 不再执行任何 IP 校验。若代理层本身配置不当（如未启用域名白名单、白名单过于宽松），则攻击者可自由访问内网元数据端点。

2. **IPv6 边缘表示法的覆盖度**
   - `expandIPv6Groups` 和 `extractMappedIPv4` 已覆盖标准压缩、展开、尾部点分十进制形式，但 IPv6 存在多种非标准或平台相关的表示（如兼容地址 `::192.168.1.1` 而非映射地址 `::ffff:192.168.1.1`）。当前实现仅检查 `0...0xffff` 的 mapped 形式，兼容地址（RFC 4291 已废弃）未被处理。

3. **缺少对 0.0.0.0 的明确处理**
   - `isBlockedV4` 将 `0.0.0.0/8` 全部阻止（`a === 0`），但某些系统上 `0.0.0.0` 可能被解析为本地监听地址，虽然从 SSRF 角度应阻止，但注释中未特别说明该边界。

4. **无日志记录被阻止的请求**
   - `ssrfGuardedLookup` 在发现 blocked 地址时仅通过 `callback` 返回错误，未调用任何日志函数。在排查用户“HTTP Hook 无法连接”问题时，缺乏审计日志会增加调试难度。

5. **CIDR 范围硬编码且无配置入口**
   - 所有阻止范围都是代码中的硬编码常量。若未来需要支持新的云厂商元数据段（如某厂商使用 `198.18.0.0/15` 作为内部服务网），必须修改源码并重新发布。

### 6.2 边界

- **仅作用于 HTTP Hook**：`ssrfGuardedLookup` 只在 `execHttpHook.ts` 中使用，Claude Code 的其他 HTTP 出口（如 MCP 服务器调用、API 请求、WebBrowser Tool）不受此 Guard 约束。
- **仅阻止 IP 层访问**：Guard 不检查域名本身，一个指向 `http://evil.example.com` 的 Hook，只要其解析到公网 IP，就不会被阻止；DNS Rebinding 攻击在 Guard 生效时无效，但在代理模式下仍可能通过代理的 DNS 缓存窗口实现。
- **IPv4/IPv6 字面量与 DNS A/AAAA 记录**：Guard 校验的是 DNS 解析后的结果，若主机名只有 A 记录（IPv4）或只有 AAAA 记录（IPv6），则分别进入对应分支；双栈主机名需全部地址合法才放行。

### 6.3 改进建议

1. **增加被阻止请求的审计日志**
   - 在 `ssrfGuardedLookup` 的 blocked 分支中加入 `logForDebugging` 或 `logEvent`，记录 `hostname`、`resolvedAddress`、`hookUrl` 等信息，便于安全审计与用户排障。

2. **考虑引入轻量级 CIDR 库或统一规则表**
   - 当前手动 octet/hextet 判断虽然零依赖，但维护成本高。可考虑使用一个经过安全审计的轻量级 CIDR 匹配库（如 `ip-address` 或自研的小型 trie），将阻止规则表达为可配置的 CIDR 列表，便于后续扩展。

3. **补充 IPv6 兼容地址的检查**
   - 虽然 IPv4-mapped IPv6 已覆盖，但可进一步检查 `::<ipv4>` 形式的 IPv4-compatible IPv6 地址（前 96 位为 0，第 5、6 组不为 `ffff`），防止历史客户端或特殊网络栈产生的绕过。

4. **为代理模式增加二次校验（域名白名单或 DNS-over-Proxy 验证）**
   - 当代理生效时，Guard 被跳过是因为无法直接解析目标 IP。可考虑在 `execHttpHook.ts` 中增加一层**域名白名单**或**解析后验证**（如通过代理的 CONNECT 隧道在建立后获取远端 IP），但这会显著增加复杂度，需权衡收益。

5. **补充单元测试**
   - 建议建立专门的 `ssrfGuard.test.ts`，覆盖以下场景：
     - 各 IPv4 边界地址（`10.0.0.1`、`172.16.0.1`、`192.168.1.1`、`127.0.0.1`、`0.0.0.0`、`169.254.169.254`、`100.100.100.200`）的 allow/block 判定；
     - 各 IPv6 边界地址（`::1`、`::`、`fe80::1`、`fc00::1`、`fd00::1`、`::ffff:192.168.1.1`、`::ffff:169.254.169.254`、`0:0:0:0:0:ffff:169.254.169.254`）的判定；
     - `expandIPv6Groups` 对压缩、展开、尾部点分十进制的正确性；
     - `ssrfGuardedLookup` 的 callback 行为：IP 字面量直接 block、DNS 全合法放行、DNS 部分非法 block、`options.all` 的返回格式、`ENOTFOUND` 处理。
