# 研究文档: src/commands/skills/index.ts

## 场景与职责

`src/commands/skills/index.ts` 是 `/skills` 命令的入口定义文件，负责将 skills 命令注册到 Claude Code 的命令系统中。该文件采用声明式配置方式，定义了一个 `local-jsx` 类型的命令，当用户输入 `/skills` 时会触发交互式的技能列表展示界面。

核心职责：
- 定义 skills 命令的元数据（名称、描述、类型）
- 配置懒加载机制，延迟加载实际的命令实现
- 将命令导出给命令中心（commands.ts）统一注册

## 功能点目的

1. **命令注册**: 将 `/skills` 命令注册到系统，使用户可以通过输入 `/skills` 查看所有可用技能
2. **懒加载优化**: 通过 `load: () => import('./skills.js')` 实现按需加载，避免启动时加载不必要的代码
3. **类型安全**: 使用 `satisfies Command` 确保命令定义符合系统约定的 Command 接口

## 具体技术实现

### 关键数据结构

```typescript
// 命令定义结构
const skills = {
  type: 'local-jsx',        // 命令类型：本地 JSX 交互式命令
  name: 'skills',           // 命令名称
  description: 'List available skills',  // 命令描述
  load: () => import('./skills.js'),     // 懒加载函数
} satisfies Command
```

### 命令类型说明

`local-jsx` 类型命令的特点：
- 渲染 React/Ink 组件提供交互式 UI
- 通过 `load()` 函数异步加载实现模块
- 加载的模块必须导出 `call` 函数，签名如下：
  ```typescript
  type LocalJSXCommandCall = (
    onDone: LocalJSXCommandOnDone,
    context: ToolUseContext & LocalJSXCommandContext,
    args: string,
  ) => Promise<React.ReactNode>
  ```

### 关键流程

1. **系统启动**: `commands.ts` 导入并收集所有命令定义
2. **命令触发**: 用户输入 `/skills`
3. **懒加载执行**: 调用 `load()` 动态导入 `./skills.js`
4. **渲染**: 执行 `call()` 函数返回 React 节点，由 Ink 渲染到终端

## 关键代码路径与文件引用

### 当前文件
- `src/commands/skills/index.ts` - 命令定义入口

### 直接依赖
- `src/commands/skills/skills.tsx` - 实际命令实现，导出 `call` 函数
- `src/commands.ts` - 导入并注册本命令（第43行: `import skills from './commands/skills/index.js'`）
- `src/types/command.ts` - Command 类型定义

### 调用链
```
用户输入 /skills
  → commands.ts 路由到 skills 命令
  → index.ts 的 load() 被调用
  → skills.tsx 被动态导入
  → skills.tsx 的 call() 执行
  → SkillsMenu 组件渲染
```

## 依赖与外部交互

### 导入依赖
| 模块 | 用途 |
|------|------|
|`../../commands.js`|导入 Command 类型定义|

### 被谁依赖
| 模块 | 用途 |
|------|------|
|`src/commands.ts`|收集并注册所有命令（第300行加入 COMMANDS 数组）|

## 风险、边界与改进建议

### 风险点
1. **模块路径硬编码**: `load: () => import('./skills.js')` 中的路径是编译后的 `.js` 路径，依赖构建系统正确处理
2. **无错误处理**: 若 `skills.tsx` 加载失败，错误会向上传播，需调用方处理

### 边界情况
1. **类型约束**: `satisfies Command` 在编译时检查，确保命令定义完整性
2. **懒加载延迟**: 首次执行 `/skills` 会有轻微延迟（模块加载时间）

### 改进建议
1. **添加加载错误处理**: 可考虑在 load 函数中包装错误处理逻辑
2. **添加加载状态**: 对于较重的组件，可考虑添加加载指示器
3. **类型导出**: 当前仅默认导出，如需扩展可考虑命名导出

---

*文档生成时间: 2026-04-01*
*研究范围: 代码实现、类型定义、命令系统架构*
