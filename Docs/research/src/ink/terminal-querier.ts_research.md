# terminal-querier.ts 研究文档

## 场景与职责

`terminal-querier.ts` 实现终端查询系统，用于向终端发送查询序列并等待响应。这是终端能力检测和状态查询的核心机制，支持 DECRQM、DA1、OSC 等多种查询类型。

### 核心职责
1. **查询构建**: 提供便捷的查询序列构造器
2. **无超时等待**: 使用 DA1 哨兵机制避免超时
3. **响应匹配**: 将终端响应与对应查询匹配
4. **批量处理**: 支持并发查询的隔离和排序

## 功能点目的

### 1. 查询类型支持

| 查询函数 | 序列 | 响应类型 | 用途 |
|---------|------|---------|------|
| `decrqm(mode)` | CSI ? Ps $ p | DECRPM | 查询 DEC 私有模式状态 |
| `da1()` | CSI c | DA1 | 主设备属性（通用哨兵） |
| `da2()` | CSI > c | DA2 | 次设备属性（版本信息） |
| `kittyKeyboard()` | CSI ? u | Kitty flags | 查询 Kitty 键盘协议状态 |
| `cursorPosition()` | CSI ? 6 n | DECXCPR | 查询光标位置 |
| `oscColor(code)` | OSC Ps ? | OSC | 查询动态颜色（背景/前景） |
| `xtversion()` | CSI > 0 q | XTVERSION | 查询终端名称/版本 |

### 2. 无超时设计
传统查询需要设置超时，但终端响应时间不确定。本模块使用 **DA1 哨兵机制**：

```typescript
// 每个查询批次以 DA1 结尾
await Promise.all([
  querier.send(decrqm(2026)),
  querier.send(decrqm(2027)),
  querier.flush(),  // 发送 DA1 哨兵
])
```

如果 DA1 响应先于查询响应到达，说明终端不支持该查询。

### 3. 响应分发
```typescript
onResponse(r: TerminalResponse): void
```

匹配策略：
1. 优先匹配显式查询（FIFO）
2. 未匹配的 DA1 触发哨兵处理
3. 哨兵前的所有未匹配查询标记为 `undefined`（不支持）

## 具体技术实现

### 查询构造器
```typescript
export type TerminalQuery<T> = {
  request: string      // 发送的转义序列
  match: (r: TerminalResponse) => r is T  // 响应匹配器
}

export function decrqm(mode: number): TerminalQuery<DecrpmResponse> {
  return {
    request: csi(`?${mode}$p`),
    match: (r): r is DecrpmResponse => 
      r.type === 'decrpm' && r.mode === mode,
  }
}
```

### TerminalQuerier 类
```typescript
export class TerminalQuerier {
  private queue: Pending[] = []
  
  send<T>(query: TerminalQuery<T>): Promise<T | undefined>
  flush(): Promise<void>
  onResponse(r: TerminalResponse): void
}
```

### 队列管理
```typescript
type Pending =
  | { kind: 'query'; match: Function; resolve: Function }
  | { kind: 'sentinel'; resolve: Function }
```

查询和哨兵按发送顺序入队，响应按 FIFO 顺序匹配。

### 哨兵处理
```typescript
if (r.type === 'da1') {
  const s = this.queue.findIndex(p => p.kind === 'sentinel')
  if (s === -1) return
  // 解析哨兵前的所有查询
  for (const p of this.queue.splice(0, s + 1)) {
    if (p.kind === 'query') p.resolve(undefined)
    else p.resolve()
  }
}
```

## 关键代码路径与文件引用

### 依赖
```typescript
import type { TerminalResponse } from './parse-keypress.js'
import { csi } from './termio/csi.js'
import { osc } from './termio/osc.js'
```

### 调用方
- **`App.tsx`**: 初始化时查询终端能力
- **能力检测模块**: 查询 DEC 2026、Kitty 键盘协议等支持情况

### 使用示例
```typescript
const querier = new TerminalQuerier(process.stdout)

// 批量查询
const [sync, grapheme, version] = await Promise.all([
  querier.send(decrqm(2026)),  // 同步输出
  querier.send(decrqm(2027)),  // 字素簇
  querier.send(xtversion()),   // 终端版本
  querier.flush(),             // DA1 哨兵
])

// sync/grapheme 是 DECRPM 响应或 undefined（不支持）
// version 是 XTVERSION 响应或 undefined
```

## 依赖与外部交互

| 依赖 | 用途 |
|------|------|
| `parse-keypress.ts` | `TerminalResponse` 类型定义 |
| `termio/csi.ts` | CSI 序列构造 |
| `termio/osc.ts` | OSC 序列构造 |

### 外部交互
- **stdin**: 接收终端响应（通过 `onResponse`）
- **stdout**: 发送查询序列

## 风险、边界与改进建议

### 已知风险
1. **响应丢失**: 如果终端不响应 DA1，所有查询将永远 pending
2. **并发冲突**: 多个独立调用者可能干扰彼此的查询
3. **顺序依赖**: 终端必须按发送顺序响应，某些终端可能不遵守

### 边界情况
1. **无响应终端**: 查询将保持 pending，需要外部超时机制
2. **意外 DA1**: 用户代码显式发送 DA1 可能干扰哨兵机制
3. **响应解析失败**: 格式错误的响应被静默丢弃

### 改进建议
1. **外部超时**: 提供可选的超时包装器
2. **调试日志**: 记录查询-响应对，便于诊断终端兼容性问题
3. **批量优化**: 自动合并相邻的 `flush()` 调用
4. **重试机制**: 对关键查询支持重试
5. **查询缓存**: 会话期间缓存终端能力查询结果

### 相关标准
- DECRQM/DECRPM - Request/Report Mode
- DA1/DA2 - Device Attributes
- XTVERSION - XTerm 版本查询
- OSC 10/11 - 动态颜色查询
