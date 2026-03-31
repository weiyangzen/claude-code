# 研究文档: src/commands/output-style/index.ts

## 场景与职责

### 文件定位
本文件是 `/output-style` 命令的**入口定义文件**，采用 Claude Code 命令系统的标准双文件结构模式（`index.ts` + 实现文件）。该命令目前处于**已弃用 (Deprecated)** 状态，仅作为向后兼容的占位符存在。

### 历史演进
- **早期版本**: `/output-style` 是一个独立的交互式命令，用于直接修改输出风格设置
- **架构调整**: 随着配置系统的发展，所有配置项被统一整合到 `/config` 命令中
- **当前状态**: 命令保留但标记为隐藏，执行时仅返回弃用提示

### 设计意图
1. **向后兼容性**: 避免老用户因命令突然消失而产生困惑
2. **迁移引导**: 明确告知用户功能已迁移至 `/config`
3. **代码组织**: 遵循命令系统的标准结构（`index.ts` 定义元数据，`.tsx` 实现逻辑）

---

## 功能点目的

### 核心功能
| 属性 | 值 | 说明 |
|------|-----|------|
| `type` | `'local-jsx'` | 本地 JSX 命令类型，支持交互式 UI |
| `name` | `'output-style'` | 命令名称，用户通过 `/output-style` 调用 |
| `description` | `'Deprecated: use /config to change output style'` | 命令描述，明确标注已弃用 |
| `isHidden` | `true` | 隐藏于命令列表和自动补全中 |
| `load` | `() => import('./output-style.js')` | 懒加载实现模块 |

### 命令类型说明
`local-jsx` 类型的命令特点：
- 使用 React/JSX 渲染交互式界面
- 通过 `LocalJSXCommandOnDone` 回调返回结果
- 支持 `ToolUseContext` 访问应用状态
- 适合需要复杂 UI 交互的场景

---

## 具体技术实现

### 1. 命令定义结构

```typescript
import type { Command } from '../../commands.js'

const outputStyle = {
  type: 'local-jsx',
  name: 'output-style',
  description: 'Deprecated: use /config to change output style',
  isHidden: true,
  load: () => import('./output-style.js'),
} satisfies Command

export default outputStyle
```

**关键实现细节**:
- 使用 `satisfies Command` 进行类型约束，确保符合 `Command` 接口
- `isHidden: true` 使命令在帮助文档和自动补全中不可见
- 懒加载模式避免在启动时加载未使用的代码

### 2. 懒加载机制

```typescript
load: () => import('./output-style.js')
```

- 使用动态 `import()` 实现代码分割
- 仅在命令被调用时加载实现模块
- 返回的 Promise 解析为包含 `call` 函数的对象

### 3. 与命令系统的集成

命令注册流程：
```
index.ts (定义)
    ↓
import 到 src/commands.ts
    ↓
添加到 COMMANDS 数组
    ↓
通过 getCommands() 暴露给 REPL
```

---

## 关键代码路径与文件引用

### 直接依赖
| 文件 | 导入内容 | 用途 |
|------|----------|------|
| `../../commands.js` | `Command` 类型 | 类型约束 |

### 被依赖方
| 文件 | 引用方式 | 行号 |
|------|----------|------|
| `src/commands.ts` | `import outputStyle from './commands/output-style/index.js'` | Line 177 |
| `src/commands.ts` | 添加到 `COMMANDS` 数组 | Line 291 |

### 关联文件
| 文件 | 职责 |
|------|------|
| `src/commands/output-style/output-style.tsx` | 命令实际执行逻辑 |
| `src/types/command.ts` | `Command` 类型定义 |

---

## 依赖与外部交互

### 类型依赖
```typescript
// 来自 src/types/command.ts
export type Command = CommandBase & (PromptCommand | LocalCommand | LocalJSXCommand)

export type LocalJSXCommand = {
  type: 'local-jsx'
  load: () => Promise<LocalJSXCommandModule>
}

export type LocalJSXCommandModule = {
  call: LocalJSXCommandCall
}
```

### 模块关系图
```
src/commands/output-style/index.ts
    ├── imports: ../../commands.js (Command type)
    ├── lazy-loads: ./output-style.js (implementation)
    └── exported to: ../../commands.ts (registration)
```

---

## 风险、边界与改进建议

### 当前风险

1. **弃用命令维护负担**
   - 风险: 长期保留无用代码增加维护成本
   - 现状: 命令仅返回提示，无实际业务逻辑
   - 建议: 考虑在主要版本更新中彻底移除

2. **类型定义耦合**
   - 风险: 依赖 `Command` 类型，若接口变更需同步更新
   - 缓解: 使用 `satisfies` 关键字确保类型安全

### 边界情况

1. **命令发现**
   - `isHidden: true` 确保命令不会出现在自动补全中
   - 但直接输入 `/output-style` 仍可执行

2. **懒加载失败**
   - 若 `output-style.js` 文件缺失，动态导入将抛出错误
   - 错误处理在命令调度层统一处理

### 改进建议

1. **添加版本标记**
   ```typescript
   /**
    * @deprecated Since vX.Y.Z. Will be removed in vX.Y+1.Z. Use /config instead.
    */
   const outputStyle = { ... }
   ```

2. **考虑彻底移除**
   - 评估用户迁移周期后，可在未来版本中完全删除该命令
   - 移除前需在更新日志中明确通知

3. **监控使用情况**
   - 添加埋点统计该命令的调用频率
   - 若使用量极低，可加速移除进程

---

## 附录: 命令注册完整流程

```typescript
// src/commands.ts
import outputStyle from './commands/output-style/index.js'

const COMMANDS = memoize((): Command[] => [
  // ... 其他命令
  outputStyle,  // Line 291
  // ... 其他命令
])

export async function getCommands(cwd: string): Promise<Command[]> {
  const allCommands = await loadAllCommands(cwd)
  return allCommands.filter(
    _ => meetsAvailabilityRequirement(_) && isCommandEnabled(_)
  )
}
```

命令执行流程：
```
用户输入 /output-style
    ↓
REPL 解析命令名
    ↓
findCommand('output-style', commands)
    ↓
匹配到 outputStyle 命令对象
    ↓
调用 command.load()
    ↓
动态导入 output-style.js
    ↓
执行 call(onDone, context, args)
    ↓
onDone 返回结果给 REPL
```
