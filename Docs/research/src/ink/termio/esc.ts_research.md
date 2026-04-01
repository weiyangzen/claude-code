# esc.ts 研究报告

## 场景与职责

`esc.ts` 实现了 **简单 ESC 序列解析器**。与 CSI 序列（`ESC [` 开头）不同，简单 ESC 序列格式为 `ESC + 一个或两个字符`，用于执行终端复位、光标保存/恢复等基础操作。

该文件的核心职责：
1. **解析简单 ESC 序列** —— 识别 `ESC <字符>` 和 `ESC <中间字符><最终字符>` 格式
2. **映射到语义动作** —— 将解析的序列转换为 `Action` 对象
3. **处理字符集选择** —— 静默忽略字符集选择序列（常见于旧系统）

## 功能点目的

### 1. 支持的 ESC 序列

| 序列 | 动作 | 说明 |
|------|------|------|
| `ESC c` | `{ type: 'reset' }` | 完全复位 (RIS - Reset to Initial State) |
| `ESC 7` | `{ type: 'cursor', action: { type: 'save' } }` | 保存光标 (DECSC) |
| `ESC 8` | `{ type: 'cursor', action: { type: 'restore' } }` | 恢复光标 (DECRC) |
| `ESC D` | `{ type: 'cursor', action: { type: 'move', direction: 'down', count: 1 } }` | 索引/下移 (IND) |
| `ESC M` | `{ type: 'cursor', action: { type: 'move', direction: 'up', count: 1 } }` | 反向索引/上移 (RI) |
| `ESC E` | `{ type: 'cursor', action: { type: 'nextLine', count: 1 } }` | 下一行 (NEL) |
| `ESC H` | `null` | 水平制表设置 (HTS) - 静默忽略 |
| `ESC ( X`, `ESC ) X` | `null` | 字符集选择 - 静默忽略 |
| 其他 | `{ type: 'unknown', sequence: ... }` | 未知序列 |

### 2. 解析函数

```typescript
/**
 * Parse a simple ESC sequence
 * @param chars - Characters after ESC (not including ESC itself)
 */
export function parseEsc(chars: string): Action | null
```

## 具体技术实现

### 解析逻辑

```typescript
export function parseEsc(chars: string): Action | null {
  if (chars.length === 0) return null
  const first = chars[0]!

  // 单字符序列
  if (first === 'c') return { type: 'reset' }
  if (first === '7') return { type: 'cursor', action: { type: 'save' } }
  if (first === '8') return { type: 'cursor', action: { type: 'restore' } }
  if (first === 'D') return { type: 'cursor', action: { type: 'move', direction: 'down', count: 1 } }
  if (first === 'M') return { type: 'cursor', action: { type: 'move', direction: 'up', count: 1 } }
  if (first === 'E') return { type: 'cursor', action: { type: 'nextLine', count: 1 } }
  if (first === 'H') return null  // HTS - 忽略

  // 双字符序列：字符集选择
  if ('()'.includes(first) && chars.length >= 2) return null

  // 未知序列
  return { type: 'unknown', sequence: `\x1b${chars}` }
}
```

### 序列分类

ESC 序列根据第二个字节分为几类：

```
ESC 0x40-0x5F ( @ A-Z [ \ ] ^ _ ): 标准单字符序列
ESC 0x20-0x2F ( 空格 ! " # $ % & ' ( ) * + , - . / ): 中间字节，后跟 0x30-0x7E
```

本解析器处理：
- **标准序列**：`c`, `7`, `8`, `D`, `M`, `E`, `H`
- **字符集选择**：`(` 和 `)` 开头的两字符序列

## 关键代码路径与文件引用

### 依赖

| 被导入 | 来源 | 用途 |
|--------|------|------|
| `Action` | `./types.js` | 返回解析结果类型 |

### 被导入方

| 导入方 | 导入内容 | 用途 |
|--------|----------|------|
| `parser.ts` | `parseEsc` | 解析 ESC 类型序列 |

### 导出内容

```typescript
export function parseEsc(chars: string): Action | null
```

## 依赖与外部交互

### 内部依赖

- `types.ts`：`Action` 类型定义

### 外部使用场景

1. **Parser 集成**：`parser.ts` 的 `processSequence` 方法调用 `parseEsc` 处理 `identifySequence` 返回的 `'esc'` 类型序列

```typescript
// parser.ts 中的使用
private processSequence(seq: string): Action[] {
  const seqType = identifySequence(seq)
  switch (seqType) {
    // ...
    case 'esc': {
      const escContent = seq.slice(1)  // 去掉 ESC
      const action = parseEsc(escContent)
      return action ? [action] : []
    }
    // ...
  }
}
```

### 与 CSI 序列的区别

| 特性 | ESC 序列 | CSI 序列 |
|------|----------|----------|
| 格式 | `ESC <1-2字符>` | `ESC [ <参数> <终止>` |
| 复杂度 | 简单，无参数 | 复杂，支持多个参数 |
| 用途 | 基础控制 | 精细控制 |
| 示例 | `ESC 7` (保存光标) | `ESC [ 10 ; 20 H` (定位) |

## 风险、边界与改进建议

### 边界情况

1. **空输入**：`parseEsc('')` 返回 `null`
2. **字符集选择**：只处理 `(` 和 `)`，其他字符集选择字符（`-`, `.`, `/` 等）会被标记为未知
3. **序列长度**：只处理 1-2 字符序列，更长序列会被截断或标记为未知

### 风险

1. **功能不完整**：未实现所有标准 ESC 序列，如：
   - `ESC =` (DECKPAM) - 启用应用键盘模式
   - `ESC >` (DECKPNM) - 启用数字键盘模式
   - `ESC N` (SS2) - 单移 2
   - `ESC O` (SS3) - 单移 3（在 tokenizer 中单独处理）

2. **静默忽略**：HTS 和字符集选择返回 `null`，调用方无法区分"有效但忽略"和"无效"

3. **无状态**：无法处理需要状态的序列（如多字节字符集序列）

### 改进建议

1. **添加更多标准序列**：
   ```typescript
   if (first === '=') return { type: 'mode', action: { type: 'keypadApp', enabled: true } }
   if (first === '>') return { type: 'mode', action: { type: 'keypadApp', enabled: false } }
   ```

2. **区分忽略原因**：
   ```typescript
   export type EscResult = 
     | { type: 'action', action: Action }
     | { type: 'ignored', reason: string }
     | { type: 'unknown', sequence: string }
   ```

3. **添加字符集处理**：
   ```typescript
   // 支持更多字符集选择序列
   if ('()*+,-./'.includes(first) && chars.length >= 2) {
     const charset = chars[1]
     // 记录字符集切换，用于后续文本解码
     return { type: 'charset', designation: first, charset }
   }
   ```

4. **添加长度验证**：
   ```typescript
   if (chars.length > 2) {
     // 可能是 CSI 序列被错误分类，或 DCS/OSC 序列
     return { type: 'unknown', sequence: `\x1b${chars}` }
   }
   ```

5. **文档改进**：
   - 添加 DEC 标准参考（DEC STD-070）
   - 添加每个序列的终端兼容性说明

### 测试建议

- 测试所有支持的序列
- 测试字符集选择序列（`ESC ( B`, `ESC ) 0` 等）
- 测试边界长度（空字符串、1字符、2字符、3+字符）
- 测试未知序列的返回格式
- 验证与 `parser.ts` 的集成
