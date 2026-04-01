# tempfile.ts 研究文档

## 场景与职责

`tempfile.ts` 是 Claude Code CLI 的临时文件路径生成工具模块。其核心职责是为应用提供可预测的临时文件路径生成能力，特别针对与 Anthropic API 交互的场景进行了优化设计。

主要使用场景：
1. **Prompt 缓存优化**：生成用于 sandbox deny lists 等工具描述的临时文件路径，确保跨进程路径一致性以利用 API 的 prompt cache 机制
2. **编辑器临时文件**：为外部编辑器（如 promptEditor.ts）提供临时文件路径
3. **Git 操作临时文件**：为 teleport/gitBundle.ts 等 Git 相关操作提供临时存储

## 功能点目的

### 1. 可预测的临时文件路径生成
通过 `contentHash` 选项，可以基于内容生成确定性的路径。这在以下场景至关重要：
- 当路径会出现在发送给 Anthropic API 的内容中时（如工具描述中的 sandbox deny lists）
- 避免因随机 UUID 导致每次子进程生成时路径变化，从而破坏 prompt cache 前缀

### 2. 灵活的命名控制
- 支持自定义前缀（默认 `claude-prompt`）
- 支持自定义扩展名（默认 `.md`）
- 自动使用系统临时目录（`os.tmpdir()`）

## 具体技术实现

### 关键函数

```typescript
export function generateTempFilePath(
  prefix: string = 'claude-prompt',
  extension: string = '.md',
  options?: { contentHash?: string },
): string
```

**算法流程**：
1. 如果提供 `contentHash`：
   - 使用 SHA-256 对内容进行哈希
   - 取前 16 个十六进制字符作为标识符
2. 否则：
   - 使用 `crypto.randomUUID()` 生成随机 UUID
3. 使用 `path.join()` 组合：`tmpdir() + prefix + id + extension`

**示例**：
```typescript
// 基于内容的确定性路径
generateTempFilePath('denylist', '.txt', { contentHash: 'sandbox-config-v1' })
// 结果：/tmp/denylist-a1b2c3d4e5f6789a.txt（每次调用相同）

// 随机路径
generateTempFilePath('edit', '.md')
// 结果：/tmp/edit-550e8400-e29b-41d4-a716-446655440000.md（每次不同）
```

### 数据结构

```typescript
interface TempFileOptions {
  contentHash?: string  // 用于生成确定性路径的内容字符串
}
```

## 关键代码路径与文件引用

### 调用方（被谁使用）

| 文件路径 | 使用场景 |
|---------|---------|
| `src/main.tsx` | 启动时的临时文件生成 |
| `src/utils/promptEditor.ts` | 外部编辑器集成时的临时文件 |
| `src/utils/teleport/gitBundle.ts` | Git bundle 操作的临时存储 |

### 依赖模块

| 模块 | 用途 |
|-----|------|
| `crypto` (Node.js) | SHA-256 哈希和 UUID 生成 |
| `os.tmpdir()` | 获取系统临时目录 |
| `path.join` | 路径拼接 |

## 依赖与外部交互

### 无运行时依赖
本模块是纯工具函数，除 Node.js 内置模块外无其他依赖。

### 与 prompt cache 的关联
关键设计决策：当路径会出现在 API 请求内容中时，使用 `contentHash` 确保跨进程路径一致性。这是为了配合 Anthropic API 的 prompt caching 机制：
- 如果每次子进程都生成不同的随机路径，API 会将它们视为不同的内容
- 使用基于内容的哈希确保相同内容总是产生相同路径，提高缓存命中率

## 风险、边界与改进建议

### 潜在风险

1. **哈希冲突风险**
   - 使用 SHA-256 前 16 字符（64 位），在极端情况下可能存在冲突风险
   - 但在实际使用场景（临时文件）中，冲突概率可接受

2. **临时文件清理**
   - 本模块仅生成路径，不负责文件创建或清理
   - 调用方需自行管理文件生命周期

3. **并发安全**
   - 基于内容的确定性路径在并发场景下可能产生竞争
   - 调用方需确保适当的文件锁或原子操作

### 边界条件

| 场景 | 行为 |
|-----|------|
| `contentHash` 为空字符串 | 视为未提供，使用随机 UUID |
| `prefix` 包含路径分隔符 | 直接拼接，可能产生非预期目录结构 |
| `extension` 不含前导点 | 直接拼接，如 `"md"` → `"prefix-idmd"` |
| 系统临时目录不可写 | 仅返回路径字符串，不验证可写性 |

### 改进建议

1. **添加路径验证**
   ```typescript
   // 建议：验证生成的路径在系统临时目录内
   if (!generatedPath.startsWith(tmpdir())) {
     throw new Error('Path traversal detected')
   }
   ```

2. **自动清理机制**
   - 可考虑集成 `cleanupRegistry.ts` 提供自动清理能力
   - 或提供配套的 `createTempFile()` 函数，返回文件句柄和自动清理函数

3. **扩展选项**
   - 支持自定义哈希算法（当前硬编码 SHA-256）
   - 支持指定哈希输出长度（当前硬编码 16 字符）

4. **类型安全增强**
   ```typescript
   // 建议使用 branded type 区分不同类型的临时文件路径
   type TempFilePath = string & { __brand: 'TempFilePath' }
   ```

### 测试建议

应覆盖以下场景：
- `contentHash` 相同产生相同路径
- `contentHash` 不同产生不同路径
- 无 `contentHash` 时产生不同路径
- 特殊字符在 prefix/extension 中的处理
- 超长 contentHash 的处理
