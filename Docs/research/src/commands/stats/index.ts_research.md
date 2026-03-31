# src/commands/stats/index.ts 研究文档

## 场景与职责

`src/commands/stats/index.ts` 是 Claude Code 的 `/stats` 命令的入口定义文件。它遵循项目统一的命令注册模式，负责声明命令元数据并配置懒加载机制。该命令为用户提供 Claude Code 使用统计信息的可视化展示，包括会话统计、令牌使用量、活跃热力图等核心指标。

## 功能点目的

1. **命令注册**: 将 `/stats` 命令注册到 Claude Code 的命令系统中
2. **懒加载配置**: 通过 `load` 函数实现按需加载，减少启动时的内存占用
3. **类型安全**: 使用 `satisfies Command` 确保命令定义符合类型约束

## 具体技术实现

### 关键数据结构

```typescript
// 命令定义结构
const stats = {
  type: 'local-jsx',           // 命令类型：本地 JSX 组件
  name: 'stats',               // 命令名称（用户输入 /stats）
  description: 'Show your Claude Code usage statistics and activity',
  load: () => import('./stats.js'),  // 懒加载实现
} satisfies Command
```

### 命令类型说明

- `type: 'local-jsx'`: 表示这是一个本地 JSX 命令，使用 React 组件渲染交互式界面
- `load`: 返回动态导入的 Promise，实现代码分割和懒加载

### 关键代码路径

```
用户输入 /stats
    ↓
commands.ts 中的命令查找
    ↓
匹配到 stats 命令定义
    ↓
调用 load() 动态导入 ./stats.js
    ↓
执行 stats.tsx 中的 call 函数
    ↓
渲染 Stats 组件
```

## 依赖与外部交互

### 直接依赖

| 依赖 | 路径 | 用途 |
|------|------|------|
| Command 类型 | `../../commands.js` | 命令定义类型 |
| 命令实现 | `./stats.js` | 实际的命令逻辑（编译后的 stats.tsx） |

### 调用关系

**被调用方**:
- `src/commands.ts` (第187行): 导入并注册到全局命令列表
  ```typescript
  import stats from './commands/stats/index.js'
  ```

**调用方**:
- `src/commands/stats/stats.tsx`: 实际的命令实现，导出 `call` 函数

## 风险、边界与改进建议

### 风险点

1. **编译依赖**: 该文件导入 `./stats.js`，但实际源码是 `stats.tsx`，依赖构建系统正确编译
2. **懒加载失败**: 如果 `stats.tsx` 编译失败或缺失，动态导入会抛出异常

### 边界情况

1. **无数据场景**: 实际逻辑在 `Stats` 组件中处理空数据状态
2. **性能考虑**: 统计数据量大时，懒加载确保不会阻塞启动

### 改进建议

1. **错误处理**: 可考虑在 `load` 函数中添加错误边界处理
   ```typescript
   load: () => import('./stats.js').catch(() => ({
     call: async () => <Text>Error loading stats</Text>
   }))
   ```

2. **类型导出**: 考虑导出命令配置类型供测试使用

3. **描述国际化**: 当前描述为硬编码英文，未来可考虑 i18n 支持

---

**关联文件**:
- 实现: `src/commands/stats/stats.tsx`
- 注册: `src/commands.ts`
- 类型定义: `src/types/command.ts`
- 核心组件: `src/components/Stats.tsx`
