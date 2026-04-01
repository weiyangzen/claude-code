# ansi.ts 研究报告

## 场景与职责

`ansi.ts` 是 termio 模块的基础层，定义了 ANSI 控制字符和转义序列引入符（Escape Sequence Introducers）。它是整个终端 IO 系统的字节级协议基础，基于 ECMA-48 / ANSI X3.64 标准实现。

该文件的核心职责：
1. **定义 C0 控制字符集**（0x00-0x1F 和 0x7F）—— 包括 NUL、BEL、ESC、CR、LF 等基础控制字符
2. **定义 ESC 序列类型引入符** —— CSI (`[`), OSC (`]`), DCS (`P`), APC (`_`), PM (`^`), SOS (`X`), ST (`\`)
3. **提供字节分类工具函数** —— 判断是否为 C0 控制字符、是否为 ESC 序列终止字节

## 功能点目的

### 1. C0 控制字符常量 (C0 对象)

```typescript
export const C0 = {
  NUL: 0x00, SOH: 0x01, ..., ESC: 0x1b, ..., DEL: 0x7f,
} as const
```

- **目的**：标准化所有 7-bit C0 控制字符的数值表示
- **关键字符**：
  - `ESC` (0x1b)：所有转义序列的起始字节
  - `BEL` (0x07)：响铃/OSC 序列终止符
  - `CR` (0x0d)、`LF` (0x0a)：行尾控制
  - `HT` (0x09)：水平制表符

### 2. 字符串常量

```typescript
export const ESC = '\x1b'  // 转义序列起始
export const BEL = '\x07'  // OSC 终止符（响铃）
export const SEP = ';'     // 参数分隔符
```

### 3. ESC 序列类型引入符 (ESC_TYPE)

| 常量 | 字节值 | 字符 | 含义 |
|------|--------|------|------|
| CSI | 0x5b | `[` | Control Sequence Introducer - 控制序列引入符 |
| OSC | 0x5d | `]` | Operating System Command - 操作系统命令 |
| DCS | 0x50 | `P` | Device Control String - 设备控制字符串 |
| APC | 0x5f | `_` | Application Program Command - 应用程序命令 |
| PM | 0x5e | `^` | Privacy Message - 隐私消息 |
| SOS | 0x58 | `X` | Start of String - 字符串开始 |
| ST | 0x5c | `\` | String Terminator - 字符串终止符 |

### 4. 字节分类函数

```typescript
// 判断是否为 C0 控制字符（0x00-0x1F 或 0x7F）
export function isC0(byte: number): boolean

// 判断是否为 ESC 序列终止字节（0x30-0x7E）
export function isEscFinal(byte: number): boolean
```

## 具体技术实现

### 字节范围定义

```
C0 范围: 0x00 - 0x1F, 0x7F
ESC 终止字节范围: 0x30 - 0x7E (0-9, :, ;, <, =, >, ?, @ through ~)
```

### 与其他模块的关系

- **csi.ts**：导入 `ESC`, `ESC_TYPE`, `SEP` 用于构建 CSI 序列
- **osc.ts**：导入 `ESC`, `ESC_TYPE`, `BEL`, `SEP` 用于构建 OSC 序列
- **tokenize.ts**：导入 `C0`, `ESC_TYPE`, `isEscFinal` 用于状态机解析

## 关键代码路径与文件引用

### 被导入方

| 导入方 | 导入内容 | 用途 |
|--------|----------|------|
| `csi.ts` | `ESC`, `ESC_TYPE`, `SEP` | 构建 CSI 序列前缀和参数分隔 |
| `osc.ts` | `BEL`, `ESC`, `ESC_TYPE`, `SEP` | 构建 OSC 序列和终止符 |
| `tokenize.ts` | `C0`, `ESC_TYPE`, `isEscFinal` | 分词器状态机解析 |

### 导出内容

```typescript
// 常量
export const C0          // C0 控制字符数值映射
export const ESC         // ESC 字符串 '\x1b'
export const BEL         // BEL 字符串 '\x07'
export const SEP         // 参数分隔符 ';'
export const ESC_TYPE    // ESC 序列类型引入符字节值

// 函数
export function isC0(byte: number): boolean
export function isEscFinal(byte: number): boolean
```

## 依赖与外部交互

### 无外部依赖

`ansi.ts` 是 termio 模块的最底层，**不依赖任何其他模块**，仅使用 TypeScript 内置类型。

### 标准遵循

- **ECMA-48**：ECMA Standard for Control Functions for Coded Character Sets
- **ANSI X3.64**：American National Standard for Additional Controls for Use with ASCII

## 风险、边界与改进建议

### 边界情况

1. **C1 控制字符未定义**：当前仅定义 C0（7-bit），未定义 C1（8-bit，0x80-0x9F）
2. **ESC 终止字节范围较宽**：`isEscFinal` 包含 0x30-0x7E，比 CSI 终止字节范围（0x40-0x7E）更宽

### 风险

1. **硬编码假设**：假设终端使用 7-bit 控制字符，现代终端可能使用 8-bit C1
2. **无验证**：导出的常量没有运行时验证，错误使用可能导致序列构造错误

### 改进建议

1. **添加 C1 支持**：考虑添加 C1 控制字符定义（0x80-0x9F）用于 UTF-8 环境
2. **添加注释文档**：为每个 C0 字符添加 JSDoc 说明其用途
3. **考虑添加 `isPrintable` 函数**：用于区分可打印字符和控制字符
4. **添加序列验证工具**：提供函数验证构造的转义序列是否合法

### 测试建议

- 测试 `isC0` 边界值（0x1F, 0x20, 0x7E, 0x7F）
- 测试 `isEscFinal` 边界值（0x2F, 0x30, 0x7E, 0x7F）
- 验证所有 C0 常量值符合 ECMA-48 标准
