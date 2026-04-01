# termio.ts 研究文档

## 场景与职责

`termio.ts` 是 Ink 框架的 ANSI 终端 IO 模块入口文件，提供统一的 API 导出点。该模块封装了完整的 ANSI 转义序列解析和生成功能，灵感来自 Ghostty、tmux 和 iTerm2 的实现。

### 核心职责
1. **模块聚合**: 统一导出 Parser 和相关类型
2. **API 抽象**: 提供语义化的 ANSI 操作接口
3. **类型导出**: 暴露完整的类型系统供外部使用

## 功能点目的

### 1. 解析器导出
```typescript
export { Parser } from './termio/parser.js'
```

`Parser` 类是 ANSI 序列解析的核心，支持：
- **流式解析**: 增量处理输入数据
- **状态跟踪**: 维护文本样式状态
- **语义输出**: 生成结构化 Action 而非原始 token

### 2. 类型系统导出
导出完整的类型定义，包括：
- **Action 类型**: 所有可能的解析动作
- **Color 类型**: 颜色表示（named/indexed/rgb/default）
- **TextStyle 类型**: 文本样式属性
- **Cursor/Erase/Scroll/Mode/Link/Title Action**: 各类具体操作
- **工具类型**: `Grapheme`, `TextSegment`, `UnderlineStyle`

### 3. 工具函数导出
```typescript
export { colorsEqual, defaultStyle, stylesEqual } from './termio/types.js'
```

提供样式和颜色的比较、创建工具。

## 具体技术实现

### 模块结构
```
termio/
├── ansi.ts      # C0 控制字符和 ESC 类型定义
├── csi.ts       # CSI 序列（光标、擦除、滚动等）
├── dec.ts       # DEC 私有模式（同步更新、焦点事件等）
├── esc.ts       # ESC 序列解析
├── osc.ts       # OSC 序列（超链接、标题、颜色等）
├── parser.ts    # 主解析器类
├── sgr.ts       # SGR 样式解析
├── tokenize.ts  # 输入分词器
└── types.ts     # 类型定义
```

### 使用模式
```typescript
import { Parser } from './termio.js'

const parser = new Parser()
const actions = parser.feed('\x1b[31mred\x1b[0m')
// => [
//   { type: 'text', graphemes: [...], style: { fg: { type: 'named', name: 'red' }, ... } }
// ]
```

### 解析器特性
1. **增量解析**: `feed()` 方法可多次调用，状态持续保持
2. **样式跟踪**: 自动维护当前文本样式（SGR 状态机）
3. **链接跟踪**: 维护 OSC 8 超链接的开启/关闭状态
4. **重置能力**: `reset()` 方法恢复初始状态

## 关键代码路径与文件引用

### 导出内容
| 导出 | 来源 | 用途 |
|------|------|------|
| `Parser` | `parser.ts` | ANSI 序列解析 |
| `Action` | `types.ts` | 解析动作联合类型 |
| `Color` | `types.ts` | 颜色表示类型 |
| `TextStyle` | `types.ts` | 文本样式类型 |
| `colorsEqual` | `types.ts` | 颜色比较 |
| `defaultStyle` | `types.ts` | 创建默认样式 |
| `stylesEqual` | `types.ts` | 样式比较 |

### 调用方
- **`output.ts`**: 使用 Parser 处理原始 ANSI 输入
- **测试代码**: 验证 ANSI 序列解析正确性
- **外部工具**: 需要解析终端输出的模块

## 依赖与外部交互

| 依赖 | 用途 |
|------|------|
| `termio/parser.ts` | Parser 类实现 |
| `termio/types.ts` | 类型定义和工具函数 |

### 外部交互
- **stdin**: 通过 Parser 解析终端输入
- **stdout**: 通过 csi/dec/osc 模块生成输出序列

## 风险、边界与改进建议

### 已知风险
1. **API 稳定性**: 作为入口文件，导出变更影响面广
2. **循环依赖**: 子模块间可能存在循环依赖风险
3. **包大小**: 导出所有类型可能增加打包体积

### 边界情况
1. **部分序列**: 流式解析需要处理不完整的转义序列
2. **未知序列**: 无法识别的序列标记为 `unknown` 类型
3. **状态丢失**: 解析器重置后样式状态清空

### 改进建议
1. **按需导出**: 提供子路径导入（如 `termio/parser`）减少打包体积
2. **文档生成**: 从类型定义自动生成 API 文档
3. **性能优化**: 对高频解析场景进行性能优化
4. **扩展性**: 提供更简单的自定义序列注册机制

### 相关标准
- ECMA-48 / ANSI X3.64 - 控制字符标准
- XTerm 控制序列参考
- OSC 8 超链接规范
- Kitty 键盘协议
