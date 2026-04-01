# parseDeepLink.ts 研究文档

> 文件路径：`src/utils/deepLink/parseDeepLink.ts`  
> 研究时间：2026-04-01  
> 执行器：kimi (k2p5)

---

## 场景与职责

`parseDeepLink.ts` 是 Claude Code **Deep Link URI 的解析与构造器**。它定义了自定义协议 `claude-cli://` 的语法规范，负责：
1. **解析**外部传入的 URI（如 `claude-cli://open?q=hello+world&repo=owner/repo`）；
2. **校验**各参数的安全性（防止命令注入、路径遍历、Unicode 隐藏字符攻击）；
3. **构造**符合规范的 deep link URL（`buildDeepLink`）。

该模块是 deep link 安全边界的第一道闸门，所有用户可控输入在此被清洗、限长、格式校验。

---

## 功能点目的

| 功能 | 目的 |
|------|------|
| `parseDeepLink(uri)` | 将 URI 字符串解析为结构化的 `DeepLinkAction`，任何异常均抛出 `Error`。 |
| `buildDeepLink(action)` | 根据 `DeepLinkAction` 反向构造标准 `claude-cli://open` URL。 |
| `containsControlChars(s)` | 检测 ASCII 控制字符（0x00-0x1F, 0x7F），防止换行符等被用作命令分隔符。 |

### 支持的查询参数
- `q` — 预填充 prompt（不自动提交）
- `cwd` — 工作目录（绝对路径）
- `repo` — GitHub `owner/name` slug，后续由 `githubRepoPaths` MRU 配置解析

### 安全策略
- **URL 解码**：通过 `URL` 标准 API 自动解码。
- **Unicode 清洗**：调用 `partiallySanitizeUnicode(rawQuery.trim())` 去除 Tag 字符、方向控制符、零宽字符等（防御 ASCII Smuggling / Hidden Prompt Injection）。
- **控制字符拦截**：对 `cwd` 和 `query` 均执行 `containsControlChars` 检查，拦截换行、回车等潜在 shell 分隔符。
- **单引号转义**：注释说明最终转义发生在 `terminalLauncher.ts`（shell-quoting 是注入边界）。

---

## 具体技术实现

### 关键流程

#### `parseDeepLink(uri: string): DeepLinkAction`
1. **协议规范化**：
   - 接受 `claude-cli://...` 或 `claude-cli:...`（后者自动补全为 `://`）。
   - 不匹配则抛出错误。
2. **URL 解析**：使用原生 `new URL(normalized)`。
3. **Action 校验**：
   - `hostname` 必须是 `open`，否则报 "Unknown deep link action"。
4. **cwd 校验**：
   - 必须是绝对路径（`/` 开头或 Windows 盘符 `a-zA-Z]:[/\]`）。
   - 禁止控制字符。
   - 长度上限 `MAX_CWD_LENGTH = 4096`。
5. **repo 校验**：
   - 必须匹配 `REPO_SLUG_PATTERN = /^[\w.-]+\/[\w.-]+$/`。
   - 仅做格式校验，**不做文件系统访问**（保持 parser 纯函数）。
6. **query 校验**：
   - 非空且 trim 后长度 > 0 才生效。
   - 先 `partiallySanitizeUnicode` 清洗。
   - 再检查控制字符。
   - 长度上限 `MAX_QUERY_LENGTH = 5000`。

#### `buildDeepLink(action: DeepLinkAction): string`
- 以 `claude-cli://open` 为 base，依次 `searchParams.set('q'|'cwd'|'repo', ...)`，最后 `url.toString()`。

### 数据结构

```ts
export const DEEP_LINK_PROTOCOL = 'claude-cli'

export type DeepLinkAction = {
  query?: string
  cwd?: string
  repo?: string
}
```

### 安全常量

| 常量 | 值 | 设计 rationale |
|------|-----|----------------|
| `REPO_SLUG_PATTERN` | `/^[\w.-]+\/[\w.-]+$/` | 限制为 GitHub 合法 slug，防止路径遍历。 |
| `MAX_QUERY_LENGTH` | `5000` | 超过此长度 prompt 已难以一眼扫描；同时给 Windows `cmd.exe` 的 8191 字符命令行上限留足余量。 |
| `MAX_CWD_LENGTH` | `4096` | Linux PATH_MAX / Windows MAX_PATH 的量级，超出即视为异常。 |

---

## 关键代码路径与文件引用

### 本文件导出
- `parseDeepLink(uri)` → `src/utils/deepLink/protocolHandler.ts` ~L41
- `buildDeepLink(action)` → 未在本批次直接发现调用点，推测用于生成分享链接或测试构造 URL。
- `DEEP_LINK_PROTOCOL` → `src/utils/deepLink/registerProtocol.ts` ~L31（注册协议时使用）

### 被调用方详情

| 导入来源 | 符号 | 用途 |
|----------|------|------|
| `../sanitization.js` | `partiallySanitizeUnicode` | 清洗 query 中的隐藏 Unicode 字符 |

### 调用方详情

**`src/utils/deepLink/protocolHandler.ts`**
- `handleDeepLinkUri` 在 try-catch 中调用 `parseDeepLink(uri)`，解析失败时向 stderr 输出错误并返回 exit code `1`。

---

## 依赖与外部交互

- **零外部 I/O**：parser 刻意保持纯函数，不读取配置、不访问文件系统、不发起网络请求。
- **Unicode 清洗下沉**：将复杂的 Unicode 安全处理委托给 `../sanitization.js`（该模块基于 NFKC 归一化 + 危险 Unicode 类别剔除）。

---

## 风险、边界与改进建议

### 风险
1. **Windows 路径校验较宽**：`^[a-zA-Z]:[/\\]` 允许盘符后接反斜杠或斜杠，但未校验路径中是否包含 `..` 遍历序列；不过真正的路径验证发生在后续 `protocolHandler.ts` 的 `resolveCwd` 及文件系统层面。
2. **query 超限即拒绝**：设计选择是 "Reject, don't truncate"，避免截断改变语义；但这也意味着超长合法 prompt 会被完全拒绝。
3. **`claude-cli:` 无斜杠的兼容**：为了容错将 `claude-cli:` 自动改写为 `claude-cli://`，若 URI 来自不可信来源，这种改写本身不会引入安全问题，但增加了 parser 的复杂度。

### 边界
- `repo` 参数仅校验格式，**不验证仓库是否存在、是否被用户本地克隆**；真正的路径解析在 `protocolHandler.ts` 中通过 `getKnownPathsForRepo` 完成。
- `cwd` 校验仅确保是绝对路径且不含控制字符，**不验证目录是否存在或是否可访问**。

### 改进建议
1. **增加 `cwd` 路径遍历显式拦截**：虽然后续有文件系统过滤，但在 parser 层直接拒绝 `..` 序列可让错误信息更早、更清晰地暴露给调用方。
2. **测试覆盖**：建议补充：
   - 各类合法/非法 URI 的解析测试（含 `claude-cli:` 与 `claude-cli://` 变体）；
   - 5000/4096 长度边界的测试；
   - Unicode 隐藏字符注入测试（确保清洗后仍被拒绝或已无害）。
3. **query 长度提示**：当 query 接近 5000 时，可考虑在构造 URL 的调用方增加日志或提示，帮助排查链接生成端的问题。
