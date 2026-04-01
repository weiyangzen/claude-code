# warn.ts 研究文档

## 场景与职责

`warn.ts` 是一个轻量级的调试警告工具模块，用于在开发/调试阶段输出类型检查警告。主要用于验证数值是否为整数，帮助捕获潜在的布局计算错误。

### 核心职责
1. **整数验证**: 检查数值是否为整数
2. **调试日志**: 通过调试系统输出警告信息
3. **非阻塞**: 验证失败不抛出错误，仅记录警告

## 功能点目的

### 1. 整数验证 (`ifNotInteger`)
验证传入的数值是否为整数，用于捕获以下问题：
- 布局计算产生的小数坐标
- 意外的浮点数值传入整数期望的 API
- 数据转换错误

```typescript
export function ifNotInteger(value: number | undefined, name: string): void
```

**参数**:
- `value`: 待验证的数值
- `name`: 数值的标识名称（用于日志）

**行为**:
- `undefined` 值通过验证（无操作）
- 整数值通过验证（无操作）
- 非整数值输出警告日志

## 具体技术实现

### 实现代码
```typescript
import { logForDebugging } from '../utils/debug.js'

export function ifNotInteger(value: number | undefined, name: string): void {
  if (value === undefined) return
  if (Number.isInteger(value)) return
  logForDebugging(`${name} should be an integer, got ${value}`, {
    level: 'warn',
  })
}
```

### 验证逻辑
1. **空值保护**: `undefined` 是合法值（可选参数场景）
2. **整数检查**: 使用 `Number.isInteger()` 严格检查
3. **日志输出**: 通过 `logForDebugging()` 输出警告级别日志

### 日志格式
```
2024-01-15T10:30:00.000Z [WARN] marginTop should be an integer, got 10.5
```

## 关键代码路径与文件引用

### 依赖
```typescript
import { logForDebugging } from '../utils/debug.js'
```

### 调用方
- **布局代码**: 验证 Yoga 布局计算结果
- **样式应用**: 检查样式值是否为整数
- **DOM 操作**: 验证坐标和尺寸值

### 使用示例
```typescript
import { ifNotInteger } from './warn.js'

// 布局计算后验证
const top = yogaNode.getComputedTop()
ifNotInteger(top, 'computedTop')

// 样式应用时验证
ifNotInteger(style.marginTop, 'marginTop')
```

## 依赖与外部交互

| 依赖 | 用途 |
|------|------|
| `utils/debug.ts` | `logForDebugging()` 日志输出 |

### 外部交互
- **调试日志系统**: 警告信息写入调试日志文件
- **开发环境**: 仅在调试模式输出可见

## 风险、边界与改进建议

### 已知风险
1. **性能**: 高频调用可能产生大量日志
2. **误报**: 合法的浮点值可能触发警告
3. **忽略**: 警告可能被开发者忽视

### 边界情况
1. **NaN**: `Number.isInteger(NaN)` 返回 `false`，会触发警告
2. **Infinity**: `Number.isInteger(Infinity)` 返回 `false`
3. **负数**: 负整数通过验证
4. **极大值**: 极大数值可能失去整数精度

### 改进建议
1. **阈值检查**: 添加可接受的误差范围（如 `Math.abs(value - Math.round(value)) < 0.001`）
2. **调用栈**: 在警告中包含调用栈信息
3. **统计**: 记录警告频率，避免重复输出相同警告
4. **严格模式**: 提供选项在严格模式下抛出错误而非警告
5. **扩展验证**: 添加范围检查（如 `ifNotInRange`）

### 代码质量
- **简单性**: 模块非常简单，职责单一
- **可测试性**: 易于单元测试
- **无副作用**: 纯函数，无副作用
