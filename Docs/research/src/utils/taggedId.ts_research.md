# taggedId.ts 研究文档

## 场景与职责

`taggedId.ts` 实现了与 Claude API 的 `tagged_id.py` 格式兼容的标记 ID 编码。该模块将 UUID 转换为特定格式的字符串 ID，用于账户、组织等实体的标识。

## 功能点目的

### API 兼容的 ID 编码
- **目标**: 生成与后端 API 兼容的标记 ID
- **格式**: `{tag}_{version}{base58(uuid_as_128bit_int)}`
- **示例**: `user_01PaGUP2rbg1XDh7Z9W1CEpd`

### Base58 编码
- **目的**: 紧凑的 UUID 表示
- **特点**: 
  - 避免混淆字符（0/O, I/l）
  - 比 Base64 更短且 URL 安全
  - 固定 22 字符长度

## 具体技术实现

### 核心常量

```typescript
const BASE_58_CHARS = '123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz'
const VERSION = '01'
const ENCODED_LENGTH = 22  // ceil(128 / log2(58))
```

### Base58 编码

```typescript
function base58Encode(n: bigint): string {
  const base = BigInt(BASE_58_CHARS.length)
  const result = new Array<string>(ENCODED_LENGTH).fill(BASE_58_CHARS[0]!)
  let i = ENCODED_LENGTH - 1
  let value = n
  
  while (value > 0n) {
    const rem = Number(value % base)
    result[i] = BASE_58_CHARS[rem]!
    value = value / base
    i--
  }
  
  return result.join('')
}
```

### UUID 到 BigInt 转换

```typescript
function uuidToBigInt(uuid: string): bigint {
  const hex = uuid.replace(/-/g, '')
  if (hex.length !== 32) {
    throw new Error(`Invalid UUID hex length: ${hex.length}`)
  }
  return BigInt('0x' + hex)
}
```

### 主函数

```typescript
export function toTaggedId(tag: string, uuid: string): string {
  const n = uuidToBigInt(uuid)
  return `${tag}_${VERSION}${base58Encode(n)}`
}
```

## 关键代码路径与文件引用

### 本文件导出
- `toTaggedId(tag, uuid)`: 将 UUID 转换为标记 ID

### 依赖模块
无外部依赖。

### 调用方

| 文件 | 用途 |
|------|------|
| `src/utils/telemetryAttributes.ts` | 遥测属性中的 ID 编码 |

## 依赖与外部交互

### 与 API 的兼容性
- 必须与 `api/api/common/utils/tagged_id.py` 保持同步
- 使用相同的 Base58 字符集和编码算法
- 版本号 `01` 用于未来格式变更

### UUID 格式支持
- 支持带连字符的标准 UUID 格式（`xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`）
- 支持无连字符的紧凑格式（`xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`）
- 验证 UUID 长度为 32 个十六进制字符

## 风险、边界与改进建议

### 潜在风险

1. **与后端不同步**: 如果后端 Python 实现变更，此模块需要同步更新
2. **BigInt 兼容性**: 使用 ES2020 的 BigInt，旧环境可能不支持
3. **UUID 验证**: 只验证长度，不验证 UUID 版本或变体

### 边界情况

1. **零值 UUID**: `00000000-0000-0000-0000-000000000000` 编码为 `111...`（Base58 的零）
2. **最大 UUID**: `ffffffff-ffff-ffff-ffff-ffffffffffff` 正确编码
3. **无效 UUID**: 非 32 字符十六进制字符串抛出错误

### 改进建议

1. **反向解码**: 添加从标记 ID 解码回 UUID 的功能
```typescript
export function fromTaggedId(taggedId: string): { tag: string; uuid: string } {
  const [tag, encoded] = taggedId.split('_')
  if (!tag || !encoded || encoded.length < 2) {
    throw new Error('Invalid tagged ID format')
  }
  const version = encoded.slice(0, 2)
  const base58 = encoded.slice(2)
  const uuid = base58Decode(base58)
  return { tag, uuid }
}
```

2. **版本验证**: 验证版本号
```typescript
if (version !== VERSION) {
  throw new Error(`Unsupported version: ${version}, expected ${VERSION}`)
}
```

3. **UUID 格式验证**: 更严格的 UUID 验证
```typescript
const UUID_REGEX = /^[0-9a-f]{8}-?[0-9a-f]{4}-?[0-9a-f]{4}-?[0-9a-f]{4}-?[0-9a-f]{12}$/i
function isValidUuid(uuid: string): boolean {
  return UUID_REGEX.test(uuid)
}
```

4. **性能优化**: 对于高频调用，考虑缓存结果
```typescript
const cache = new Map<string, string>()
export function toTaggedId(tag: string, uuid: string): string {
  const key = `${tag}:${uuid}`
  if (cache.has(key)) return cache.get(key)!
  const result = /* ... */
  cache.set(key, result)
  return result
}
```

5. **测试向量**: 添加与 Python 实现对比的测试
```typescript
test('toTaggedId matches Python implementation', () => {
  expect(toTaggedId('user', '550e8400-e29b-41d4-a716-446655440000'))
    .toBe('user_01PaGUP2rbg1XDh7Z9W1CEpd')
})
```
