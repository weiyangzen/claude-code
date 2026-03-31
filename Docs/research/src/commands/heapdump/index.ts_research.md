# 研究文档：src/commands/heapdump/index.ts

## 场景与职责

本文件是 `/heapdump` 命令的**入口定义文件**，负责声明和导出命令配置。它是 Claude Code CLI 命令系统的标准模式实现，采用"索引文件 + 实现文件"的分离架构：

- **索引文件**（本文件）：定义命令元数据、类型、加载方式
- **实现文件**（`heapdump.ts`）：包含实际的命令执行逻辑

该命令是一个**内部调试工具**，仅对 Anthropic 内部用户（`USER_TYPE === 'ant'`）可见，用于生成 V8 JavaScript 堆内存快照以诊断内存问题。

## 功能点目的

### 命令元数据定义

| 属性 | 值 | 说明 |
|-----|-----|------|
| `type` | `'local'` | 本地执行命令，不经过模型处理 |
| `name` | `'heapdump'` | 命令名称，用户通过 `/heapdump` 调用 |
| `description` | `'Dump the JS heap to ~/Desktop'` | 简短描述 |
| `isHidden` | `true` | 隐藏命令，不在帮助/补全中显示 |
| `supportsNonInteractive` | `true` | 支持非交互模式（如 CI/自动化） |
| `load` | 懒加载函数 | 动态导入实际实现，减少启动开销 |

### 设计意图

1. **懒加载优化**：使用 `() => import('./heapdump.js')` 延迟加载实现，避免启动时加载不必要代码
2. **类型安全**：使用 `satisfies Command` 确保配置符合 `Command` 类型约束
3. **模块化**：分离元数据定义与实际实现，便于维护和测试

## 具体技术实现

### 类型系统

```typescript
import type { Command } from '../../commands.js'
```

从中央命令模块导入 `Command` 类型定义，确保类型一致性。

### 懒加载模式

```typescript
load: () => import('./heapdump.js')
```

这是 CLI 命令系统的标准懒加载模式：
- 返回一个函数，调用时动态导入模块
- 导入路径使用 `.js` 扩展名（符合 ESM 规范）
- 实际导入的模块必须导出 `call` 函数，符合 `LocalCommandCall` 类型

### 模块导出

```typescript
export default heapDump
```

使用默认导出，便于在 `commands.ts` 中导入：

```typescript
// src/commands.ts
import heapDump from './commands/heapdump/index.js'
```

## 关键代码路径与文件引用

### 文件关系图

```
src/commands/heapdump/
├── index.ts      <-- 本文件：命令定义、元数据、懒加载配置
└── heapdump.ts   <-- 实现文件：实际执行逻辑

src/commands.ts   <-- 命令注册中心，导入并注册本命令
```

### 导入依赖

| 文件路径 | 用途 |
|---------|------|
| `../../commands.js` | 导入 `Command` 类型定义 |

### 被导入关系

| 文件路径 | 用途 |
|---------|------|
| `src/commands.ts` | 导入并注册到命令列表（第138行导入，第280行加入数组） |

### 命令注册流程

```typescript
// src/commands.ts
import heapDump from './commands/heapdump/index.js'  // 第138行

// ...

const COMMANDS = memoize((): Command[] => [
  // ...
  heapDump,  // 第280行
  // ...
])
```

### 执行流程

```
用户输入 /heapdump
    ↓
命令解析器匹配到 heapDump 命令
    ↓
调用 heapDump.load() → import('./heapdump.js')
    ↓
执行 heapdump.ts 中的 call() 函数
    ↓
返回文本结果
```

## 依赖与外部交互

### 编译时依赖

- **TypeScript 类型系统**：使用 `satisfies` 关键字进行类型约束（TS 4.9+ 特性）
- **ESM 模块系统**：使用 `.js` 扩展名的导入路径

### 运行时依赖

本文件本身无运行时依赖，实际运行时依赖通过懒加载的 `heapdump.js` 引入：

- `heapDumpService.ts`：核心堆转储服务
- Node.js `v8` 模块：堆统计和快照 API
- 文件系统 API：写入快照文件

### 与命令系统集成

作为 `local` 类型命令，本命令遵循以下集成点：

1. **REPL 命令解析**：通过 `name` 属性注册到命令补全系统
2. **执行调度**：通过 `load()` 获取实现，调用 `call()` 执行
3. **结果处理**：返回 `LocalCommandResult` 类型结果

## 风险、边界与改进建议

### 架构风险

1. **文件分离带来的维护成本**：
   - 索引文件和实现文件分离，修改时需要同步更新
   - 建议：保持简单，索引文件只包含元数据

2. **懒加载错误延迟暴露**：
   - 实现文件的语法错误要到命令首次执行时才暴露
   - 缓解：CI 中应包含命令加载测试

### 边界情况

| 场景 | 行为 |
|-----|------|
| 模块导入失败 | 命令执行时抛出错误，由命令系统捕获处理 |
| 重复注册 | `commands.ts` 中的数组顺序决定，后覆盖前 |
| 名称冲突 | 与其他命令同名时，后加载的命令生效 |

### 改进建议

1. **添加版本信息**：
   ```typescript
   version: '1.0.0'  // 便于追踪命令变更
   ```

2. **明确权限声明**：
   ```typescript
   availability: ['ant']  // 明确限定内部用户
   ```

3. **添加别名支持**（如需要）：
   ```typescript
   aliases: ['hd', 'dumpheap']
   ```

4. **参数提示**：
   ```typescript
   argumentHint: '[filename]'  // 如支持自定义文件名
   ```

### 与相关组件的协作建议

1. **MemoryUsageIndicator 组件**（`src/components/MemoryUsageIndicator.tsx`）：
   - 当前在内存高使用时显示 `/heapdump` 提示
   - 建议：可考虑添加点击/快捷键直接触发命令

2. **useMemoryUsage Hook**（`src/hooks/useMemoryUsage.ts`）：
   - 定义了 1.5GB 和 2.5GB 内存阈值
   - 建议：阈值可配置化，与自动触发逻辑统一

3. **heapDumpService**（`src/utils/heapDumpService.ts`）：
   - 支持 `auto-1.5GB` 触发类型
   - 建议：在命令元数据中声明自动触发能力
