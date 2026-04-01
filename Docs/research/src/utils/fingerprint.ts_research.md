# 研究文档：src/utils/fingerprint.ts

## 场景与职责

`fingerprint.ts` 负责计算 Claude Code 的**归因指纹（attribution fingerprint）**，用于 API 请求中的 OAuth 验证和调用来源标识。该指纹是一个 3 字符的十六进制字符串，必须与服务端预期的算法严格一致，否则可能导致 1P（Anthropic 官方）和 3P（Bedrock、Vertex、Azure）API 的验证失败。

## 功能点目的

| 导出项 | 目的 |
|--------|------|
| `FINGERPRINT_SALT` | 硬编码盐值 `'59cf53e54c78'`，用于指纹计算。 |
| `extractFirstMessageText(messages)` | 从消息数组中提取第一条用户消息的纯文本内容。 |
| `computeFingerprint(messageText, version)` | 根据首条用户消息文本和版本号计算 3 字符指纹。 |
| `computeFingerprintFromMessages(messages)` | 便捷函数，直接使用 `MACRO.VERSION` 计算指纹。 |

## 具体技术实现

### 算法

```ts
export function computeFingerprint(messageText: string, version: string): string {
  const indices = [4, 7, 20]
  const chars = indices.map(i => messageText[i] || '0').join('')
  const fingerprintInput = `${FINGERPRINT_SALT}${chars}${version}`
  const hash = createHash('sha256').update(fingerprintInput).digest('hex')
  return hash.slice(0, 3)
}
```

- 取首条用户消息文本的第 4、7、20 个字符（越界时用 `'0'` 补齐）。
- 拼接格式：`SALT + char4 + char7 + char20 + VERSION`
- 对拼接结果做 SHA-256，取前 3 位十六进制字符。

### 首条消息提取

```ts
export function extractFirstMessageText(messages): string {
  const firstUserMessage = messages.find(msg => msg.type === 'user')
  // 支持 string content 和 array content block（取第一个 text block）
}
```

- 兼容早期 string-only 消息格式和现代 content-block 数组格式。

## 关键代码路径与文件引用

### 调用方

| 文件 | 导入内容 | 说明 |
|------|----------|------|
| `src/utils/sideQuery.ts:18` | `computeFingerprint` | `sideQuery` 轻量 API 调用时注入指纹头。 |
| `src/services/api/claude.ts:73` | `computeFingerprintFromMessages` | 主对话循环在发送 API 请求前计算指纹，注入到 attribution header 中。 |

### 被调用方

- Node.js `crypto`：`createHash`
- `MACRO.VERSION`：构建时注入的全局常量（通过 Bun `--define` 或等价机制）。

## 依赖与外部交互

- 无网络依赖。
- 无持久化。
- 与后端验证强耦合：注释明确警告 **"Do not change this method without careful coordination with 1P and 3P APIs"**。

## 风险、边界与改进建议

### 风险

1. **算法变更的级联影响**：任何对 `computeFingerprint` 逻辑、`FINGERPRINT_SALT`、或字符索引的修改，都会导致所有后端验证失败，属于**全局 P0 级风险**。
2. **空消息/极短消息**：当首条用户消息长度不足 20 时，大量 `'0'` 填充会降低指纹的熵，但这不是安全漏洞，只是统计上的碰撞概率略增。
3. **MACRO.VERSION 缺失**：若构建系统未正确注入 `MACRO`，运行时访问 `MACRO.VERSION` 会抛出 ReferenceError。代码中其他模块（如 `sessionStorage.ts`）有 `typeof MACRO !== 'undefined'` 的防御，但 `fingerprint.ts` 没有，存在潜在崩溃风险。

### 边界

- 只处理 `msg.type === 'user'` 的消息；系统消息、assistant 消息、附件消息不参与指纹计算。
- 若首条用户消息是纯图片（无 text block），返回空字符串，指纹变为基于 `'000'` 的固定值。
- 指纹仅用于归因验证，不参与任何加密或签名。

### 改进建议

1. **防御性编程**：在 `computeFingerprintFromMessages` 中增加 `typeof MACRO !== 'undefined'` 检查，fallback 到 `'unknown'`，避免未定义全局变量导致的运行时异常。
2. **单元测试**：建议补充对空消息、短消息、超长消息、纯图片消息、以及已知输入输出对的固定测试向量，确保算法不会因依赖库升级而漂移。
3. **文档同步**：将算法细节同步到后端团队的 API 契约文档中，确保 1P/3P 在更新时有统一参考。
