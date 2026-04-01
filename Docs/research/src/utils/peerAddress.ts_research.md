# peerAddress.ts 研究文档

## 场景与职责

本模块提供对等地址解析功能，用于 SendMessageTool 的消息路由。核心职责包括：

1. **地址解析**：将 URI 风格的对等地址解析为 scheme + target
2. **传输层解耦**：避免 SendMessageTool 直接依赖 bridge 和 UDS 模块
3. **遗留兼容**：支持裸 UDS 套接字路径的遗留格式

该模块是消息路由系统的基础设施，确保 SendMessageTool 在工具枚举时不会间接加载重型依赖（axios、fs、net）。

## 功能点目的

### `parseAddress()` - 地址解析
- **目的**：解析对等地址字符串，确定消息传输方式
- **支持的 scheme**：
  - `uds:` - Unix Domain Socket 传输
  - `bridge:` - Bridge（HTTP）传输
  - `/path`（遗留）- 裸 UDS 路径，自动路由到 UDS
  - 其他 - 其他传输方式（如 teammate 名称）
- **设计意图**：
  - 保持与 peerRegistry.ts 分离，避免循环依赖
  - 避免在工具枚举时加载 bridge（axios）和 UDS（fs、net）模块

## 具体技术实现

### 算法实现

```typescript
export function parseAddress(to: string): {
  scheme: 'uds' | 'bridge' | 'other'
  target: string
} {
  if (to.startsWith('uds:')) return { scheme: 'uds', target: to.slice(4) }
  if (to.startsWith('bridge:')) return { scheme: 'bridge', target: to.slice(7) }
  // 遗留：旧代码 UDS 发送者在 from= 中发送裸套接字路径
  // 通过 UDS 分支路由，确保回复不会静默进入 teammate 路由
  if (to.startsWith('/')) return { scheme: 'uds', target: to }
  return { scheme: 'other', target: to }
}
```

### 关键设计决策

| 决策 | 说明 |
|------|------|
| 无裸 session ID 回退 | Bridge 消息是新功能，没有旧发送者，无前缀不会劫持 teammate 名称（如 session_manager） |
| `/` 前缀检测 | 简单有效的 UDS 路径检测（Unix 绝对路径） |
| 大小写敏感 | Scheme 检测区分大小写（`UDS:` 不匹配） |
| 无验证 | 不验证 target 格式，由调用方处理 |

### 地址格式映射

| 输入 | scheme | target | 说明 |
|------|--------|--------|------|
| `uds:/tmp/socket` | `uds` | `/tmp/socket` | 标准 UDS 格式 |
| `bridge:session123` | `bridge` | `session123` | Bridge 传输 |
| `/tmp/socket` | `uds` | `/tmp/socket` | 遗留 UDS 格式 |
| `teammate_name` | `other` | `teammate_name` | 其他（如 teammate） |
| `session_manager` | `other` | `session_manager` | 其他 |

## 依赖与外部交互

### 直接依赖

无外部依赖（纯函数实现）。

### 调用方

| 调用方 | 用途 |
|--------|------|
| `src/tools/SendMessageTool/SendMessageTool.ts` | 消息发送地址解析 |

### 架构关系

```
SendMessageTool
    ↓
peerAddress.ts (本模块) - 轻量级，无重型依赖
    ↓
peerRegistry.ts - 重型依赖（axios、fs、net）
    ↓
Bridge / UDS 实现
```

### 设计目标

- **工具枚举时**：仅加载 `peerAddress.ts`（轻量）
- **实际发送时**：按需加载 `peerRegistry.ts` 和相关模块

## 风险、边界与改进建议

### 已知风险

1. **Scheme 命名冲突**
   - 风险：未来可能添加与 `uds:`、`bridge:` 冲突的 scheme
   - 缓解：使用明确的 URI 风格前缀
   - 建议：注册 scheme 命名空间

2. **Windows UDS 支持**
   - 风险：Windows 10+ 支持 UDS，但路径格式不同
   - 现状：`/path` 检测假设 Unix 风格
   - 潜在问题：Windows UDS 路径（`\\.\pipe\...`）不被识别

3. **相对路径误识别**
   - 风险：`./path` 或 `../path` 不被识别为 UDS
   - 现状：仅检测 `/` 开头的绝对路径
   - 潜在问题：相对 UDS 路径不被支持

4. **无格式验证**
   - 风险：`uds:invalid path` 等格式问题延迟发现
   - 现状：简单切片提取 target
   - 建议：添加基本验证

### 边界情况

| 场景 | 行为 |
|------|------|
| 空字符串 | `scheme: 'other'`, `target: ''` |
| 仅 scheme（`uds:`） | `scheme: 'uds'`, `target: ''` |
| 大写 scheme（`UDS:`） | `scheme: 'other'`, `target: 'UDS:'` |
| 多个 scheme（`uds:bridge:x`） | `scheme: 'uds'`, `target: 'bridge:x'` |
| 带查询参数（`uds:/path?query`） | `scheme: 'uds'`, `target: '/path?query'` |
| Windows 路径（`C:\path`） | `scheme: 'other'`, `target: 'C:\path'` |

### 改进建议

1. **Scheme 注册机制**
   - 当前：硬编码 scheme 检测
   - 建议：
     ```typescript
     const SCHEMES = ['uds', 'bridge', 'ws', 'http'] as const
     ```
   - 好处：便于扩展和维护

2. **URI 规范解析**
   - 当前：简单前缀检测
   - 建议：使用 `URL` 类解析
   - 注意：自定义 scheme 需要特殊处理

3. **Windows UDS 支持**
   - 建议：
     - 检测 Windows UDS 路径格式
     - 支持命名管道（`\\.\pipe\...`）

4. **验证和清理**
   - 建议：
     - 验证 UDS 路径存在性（可选）
     - 清理 target（去除空白）
     - 规范化路径

5. **错误处理**
   - 当前：总是返回成功
   - 建议：
     - 返回解析错误信息
     - 区分未知 scheme 和格式错误

6. **类型安全增强**
   - 当前：返回联合类型
   - 建议：
     ```typescript
     type UDSAddress = { scheme: 'uds'; target: string }
     type BridgeAddress = { scheme: 'bridge'; target: string }
     type OtherAddress = { scheme: 'other'; target: string }
     type ParsedAddress = UDSAddress | BridgeAddress | OtherAddress
     ```

7. **文档和示例**
   - 建议：
     - 添加 JSDoc 示例
     - 说明各 scheme 用途
     - 记录遗留格式支持

8. **测试覆盖**
   - 建议：
     - 各种 scheme 格式测试
     - 边界情况测试
     - 遗留格式兼容性测试
