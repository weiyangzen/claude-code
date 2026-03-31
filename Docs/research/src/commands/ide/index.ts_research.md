# 研究文档: src/commands/ide/index.ts

## 场景与职责

`index.ts` 是 `/ide` 命令的入口定义文件，采用 Claude Code 的命令注册模式。它定义了命令的元数据（名称、描述、参数提示）和懒加载配置，是连接命令注册系统和实际实现的桥梁。

该文件职责单一但关键：
1. **命令注册** - 向系统注册 `/ide` 命令
2. **懒加载配置** - 延迟加载重型实现代码（ide.tsx）
3. **类型安全** - 使用 `satisfies Command` 确保类型正确

## 功能点目的

### 1. 命令元数据定义

```typescript
const ide = {
  type: 'local-jsx',
  name: 'ide',
  description: 'Manage IDE integrations and show status',
  argumentHint: '[open]',
  load: () => import('./ide.js'),
} satisfies Command
```

各字段含义：
- `type: 'local-jsx'`: 命令类型为本地 JSX 交互式命令，使用 React + Ink 渲染 TUI
- `name: 'ide'`: 命令名称，用户通过 `/ide` 调用
- `description`: 在帮助系统和自动补全中显示的描述
- `argumentHint: '[open]'`: 参数提示，`[open]` 表示可选的 "open" 参数
- `load`: 懒加载函数，返回 `ide.js` 模块的 Promise

### 2. 懒加载机制

```typescript
load: () => import('./ide.js')
```

- **目的**: 避免在启动时加载重型依赖（React、Ink、IDE 检测逻辑等）
- **触发时机**: 用户实际调用 `/ide` 命令时
- **模块约定**: 加载的模块必须导出 `call` 函数，符合 `LocalJSXCommandCall` 类型

## 具体技术实现

### 类型系统

```typescript
import type { Command } from '../../commands.js'
```

`Command` 类型来自 `src/types/command.ts`，定义如下：

```typescript
type LocalJSXCommand = {
  type: 'local-jsx'
  load: () => Promise<LocalJSXCommandModule>
}

type LocalJSXCommandModule = {
  call: LocalJSXCommandCall
}

type LocalJSXCommandCall = (
  onDone: LocalJSXCommandOnDone,
  context: ToolUseContext & LocalJSXCommandContext,
  args: string,
) => Promise<React.ReactNode>
```

### 模块导出

```typescript
export default ide
```

默认导出供 `src/commands.ts` 导入：

```typescript
// src/commands.ts 行 24
import ide from './commands/ide/index.js'
```

## 关键代码路径与文件引用

### 直接依赖

| 文件路径 | 用途 |
|---------|------|
| `src/commands.ts` | 导入并注册该命令到全局命令列表 |
| `src/types/command.ts` | `Command` 类型定义 |
| `./ide.js` | 实际实现模块（编译后的 `ide.tsx`）|

### 调用链

```
用户输入 /ide [args]
  ↓
REPL 解析命令
  ↓
src/commands.ts 查找命令定义
  ↓
匹配到 ide 命令（type: 'local-jsx'）
  ↓
调用 ide.load() 动态导入 ./ide.js
  ↓
执行导入模块的 call 函数
  ↓
返回 React 组件树渲染 TUI
```

## 依赖与外部交互

### 与命令系统的交互

该文件是命令系统的"声明端"，与"实现端"（ide.tsx）分离：

- **声明端**（本文件）：轻量级，只包含元数据
- **实现端**（ide.tsx）：重量级，包含完整交互逻辑

### 与懒加载系统的交互

遵循 Claude Code 的懒加载模式：

1. 启动时：只加载 `index.ts`，不执行 `load()`
2. 调用时：执行 `load()`，动态导入实现模块
3. 缓存：ESM 动态导入会自动缓存，后续调用复用

## 风险、边界与改进建议

### 风险分析

1. **编译依赖**
   - 风险: `load: () => import('./ide.js')` 引用编译后的 `.js` 文件
   - 缓解: 构建系统确保 `ide.tsx` 被正确编译为 `ide.js`

2. **模块接口契约**
   - 风险: 如果 `ide.tsx` 未正确导出 `call` 函数，运行时失败
   - 缓解: TypeScript 类型检查和单元测试

### 边界情况

1. **重复导入**: ESM 模块系统确保多次 `load()` 调用返回同一模块实例
2. **加载失败**: 网络或文件系统问题可能导致动态导入失败，由调用方处理

### 改进建议

1. **类型安全增强**
   ```typescript
   // 当前
   load: () => import('./ide.js')
   
   // 建议：显式返回类型
   load: (): Promise<LocalJSXCommandModule> => import('./ide.js')
   ```

2. **错误边界**
   - 考虑在 `load()` 中添加错误处理，提供更友好的加载失败提示

3. **预加载优化**
   - 对于常用命令，可考虑在空闲时预加载实现模块
   - 例如：检测到用户在 IDE 终端中运行时，后台预加载 `ide.js`

### 相关文件变更影响

| 变更场景 | 影响 |
|---------|------|
| 修改命令描述 | 更新 `description` 字段 |
| 添加新参数 | 更新 `argumentHint` 字段 |
| 改为非交互式命令 | 修改 `type` 为 `'local'` 并调整 `load` 返回类型 |
| 重命名命令 | 修改 `name` 字段，需同步更新调用方 |
| 拆分实现文件 | 更新 `load` 中的导入路径 |
