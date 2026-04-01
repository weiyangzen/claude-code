# src/utils/controlMessageCompat.ts 深度研究文档

## 1. 场景与职责

`controlMessageCompat.ts` 是一个兼容性垫片（shim）模块，专门处理 iOS 应用与 Claude Code CLI 之间的控制消息格式兼容问题。它解决了由于 Swift CodingKeys 映射缺失导致的 `requestId` / `request_id` 命名不一致问题。

### 核心职责
- **键名规范化**: 将驼峰命名 `requestId` 转换为蛇形命名 `request_id`
- **向后兼容**: 支持旧版 iOS 应用发送的控制消息
- **消息结构修复**: 处理嵌套在 `response` 对象中的键名

## 2. 功能点目的

### 2.1 问题背景

旧版 iOS 应用构建存在 Swift CodingKeys 映射缺失问题：
- iOS 发送: `{ "requestId": "abc-123", ... }`
- CLI 期望: `{ "request_id": "abc-123", ... }`

这导致：
1. `isSDKControlRequest` 在 `replBridge.ts` 中拒绝消息（检查 `'request_id' in value`）
2. `structuredIO.ts` 读取 `message.response.request_id` 为 `undefined`
3. 消息被静默丢弃

### 2.2 解决方案

提供规范化函数，在消息处理前转换键名：

```typescript
export function normalizeControlMessageKeys(obj: unknown): unknown
```

**转换规则**:
1. 如果存在 `requestId` 但不存在 `request_id`，进行转换
2. 如果两者都存在，保留 `request_id`（蛇形命名优先）
3. 递归处理嵌套的 `response` 对象

## 3. 具体技术实现

### 3.1 核心算法

```typescript
export function normalizeControlMessageKeys(obj: unknown): unknown {
  // 1. 类型检查
  if (obj === null || typeof obj !== 'object') return obj
  
  const record = obj as Record<string, unknown>
  
  // 2. 顶层键名转换
  if ('requestId' in record && !('request_id' in record)) {
    record.request_id = record.requestId
    delete record.requestId
  }
  
  // 3. 嵌套 response 对象处理
  if (
    'response' in record &&
    record.response !== null &&
    typeof record.response === 'object'
  ) {
    const response = record.response as Record<string, unknown>
    if ('requestId' in response && !('request_id' in response)) {
      response.request_id = response.requestId
      delete response.requestId
    }
  }
  
  return obj
}
```

### 3.2 设计特点

**原地修改**: 
- 函数直接修改传入对象，而非创建新对象
- 减少内存分配，提高性能
- 调用方需注意对象已被修改

**防御性编程**:
- 严格的类型检查（`typeof obj !== 'object'`）
- `null` 检查（`record.response !== null`）
- 属性存在性检查（`'requestId' in record`）

**优先级处理**:
- 蛇形命名优先（`!('request_id' in record)` 条件）
- 避免覆盖已存在的正确键名

## 4. 关键代码路径与文件引用

### 4.1 调用方

```
src/cli/structuredIO.ts
└── normalizeControlMessageKeys(message)
    └── 处理从 iOS 接收的控制消息

src/bridge/bridgeMessaging.ts
└── normalizeControlMessageKeys(message)
    └── 处理桥接消息
```

### 4.2 调用链示例

**CLI 结构化输入处理**:
```typescript
// src/cli/structuredIO.ts
import { normalizeControlMessageKeys } from '../utils/controlMessageCompat.js'

function processControlMessage(rawMessage: unknown) {
  const message = normalizeControlMessageKeys(rawMessage)
  
  // 现在可以安全地检查 request_id
  if ('request_id' in message) {
    // 处理控制请求
  }
}
```

**桥接消息处理**:
```typescript
// src/bridge/bridgeMessaging.ts
import { normalizeControlMessageKeys } from '../utils/controlMessageCompat.js'

function handleBridgeMessage(data: unknown) {
  const normalized = normalizeControlMessageKeys(data)
  // 继续处理...
}
```

## 5. 依赖与外部交互

### 5.1 无外部依赖

```typescript
// 文件内容：纯函数实现，无 import
```

这是一个纯工具函数，不依赖任何外部模块。

### 5.2 依赖关系

```
controlMessageCompat.ts（无依赖）
    ↑
    ├── src/cli/structuredIO.ts
    └── src/bridge/bridgeMessaging.ts
```

### 5.3 与 iOS 应用的交互

```
iOS App (旧版本)
    ↓ 发送控制消息 { requestId: "..." }
Claude Code CLI
    ↓ 接收消息
structuredIO.ts / bridgeMessaging.ts
    ↓ 调用 normalizeControlMessageKeys()
    ↓ 转换为 { request_id: "..." }
replBridge.ts
    ↓ isSDKControlRequest() 检查通过
    ↓ 正常处理消息
```

## 6. 风险、边界与改进建议

### 6.1 风险分析

| 风险 | 可能性 | 影响 | 说明 |
|------|--------|------|------|
| iOS 应用更新后冗余 | 高 | 低 | 新版 iOS 修复了 CodingKeys，此垫片不再需要 |
| 其他键名不一致 | 低 | 中 | 可能存在其他未发现的命名不一致问题 |
| 深层嵌套未处理 | 中 | 低 | 当前只处理顶层和 response 一层，更深的嵌套未处理 |
| 类型安全 | 低 | 低 | 使用 `unknown` 和类型断言，缺乏编译时检查 |

### 6.2 边界情况

| 场景 | 行为 |
|------|------|
| `null` | 原样返回 |
| 非对象（字符串、数字等） | 原样返回 |
| 数组 | 原样返回（数组也是 object，但无属性） |
| 两者都存在 | 保留 `request_id`，删除 `requestId` |
| `response` 为 `null` | 跳过嵌套处理 |
| `response` 为非对象 | 跳过嵌套处理（会抛出异常，但外层有保护） |

### 6.3 改进建议

#### 短期
1. **添加日志**: 记录何时进行了键名转换，便于监控旧版 iOS 使用情况：
   ```typescript
   export function normalizeControlMessageKeys(obj: unknown): unknown {
     // ... 现有逻辑
     if ('requestId' in record && !('request_id' in record)) {
       logForDebugging('Normalized requestId to request_id (legacy iOS app)')
       // ...
     }
   }
   ```

2. **递归处理**: 支持任意深度的嵌套对象：
   ```typescript
   function normalizeDeep(obj: unknown): unknown {
     if (obj === null || typeof obj !== 'object') return obj
     if (Array.isArray(obj)) return obj.map(normalizeDeep)
     // ... 处理对象
     for (const key in obj) {
       obj[key] = normalizeDeep(obj[key])
     }
   }
   ```

#### 中期
3. **版本检测**: 根据 iOS 应用版本决定是否应用转换：
   ```typescript
   export function normalizeControlMessageKeys(
     obj: unknown,
     clientVersion?: string,
   ): unknown {
     if (clientVersion && semver.gte(clientVersion, '2.0.0')) {
       return obj // 新版不需要转换
     }
     // ... 转换逻辑
   }
   ```

4. **通用键名转换**: 支持更多潜在的命名不一致：
   ```typescript
   const KEY_MAPPINGS = {
     requestId: 'request_id',
     sessionId: 'session_id',
     // ... 其他映射
   }
   ```

#### 长期
5. **移除垫片**: 当旧版 iOS 应用使用率足够低时，移除此垫片：
   ```typescript
   // 添加 deprecation 警告
   logForDebugging('controlMessageCompat is deprecated and will be removed')
   ```

6. **协议版本协商**: 在连接建立时协商协议版本，避免运行时转换：
   ```typescript
   // 在握手阶段确定协议版本
   const protocolVersion = negotiateProtocolVersion()
   // 后续消息按协议版本解析
   ```

### 6.4 维护指南

**监控指标**:
- 记录 `normalizeControlMessageKeys` 被调用的频率
- 记录实际发生键名转换的频率
- 当转换频率低于阈值时，考虑移除此垫片

**iOS 应用更新后**:
1. 确认新版 iOS 应用已修复 CodingKeys
2. 监控一段时间确保无旧版用户
3. 添加 deprecation 警告
4. 最终移除垫片

### 6.5 测试建议

当前无直接测试，建议添加：
- 正常转换测试（`requestId` → `request_id`）
- 两者都存在时的优先级测试
- 嵌套 response 对象测试
- `null` 和非对象输入测试
- 深层嵌套对象测试（如果使用递归版本）
