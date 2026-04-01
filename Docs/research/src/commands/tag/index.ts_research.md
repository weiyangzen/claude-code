# 研究文档: src/commands/tag/index.ts

## 场景与职责

本文件是 `/tag` 命令的入口定义文件，采用 Claude Code 的本地 JSX 命令架构。该命令允许用户为当前会话添加或移除可搜索的标签（tag），用于在 `/resume` 会话恢复列表中快速识别和筛选会话。

**主要使用场景：**
- 用户为当前会话添加标签（如 `/tag bugfix`）以便后续识别
- 用户通过再次执行相同命令移除标签（toggle 行为）
- 在 `/resume` 列表中显示标签，帮助用户快速定位特定会话
- 支持通过搜索（`/`）按标签筛选会话

**访问控制：** 该命令仅限内部员工使用（`USER_TYPE === 'ant'`），通过 `isEnabled` 函数进行条件控制。

## 功能点目的

1. **标签管理**：提供添加/移除当前会话标签的能力
2. **会话标识**：标签显示在分支名称之后，增强会话可识别性
3. **搜索支持**：标签可被搜索，支持快速定位会话
4. **权限控制**：通过环境变量限制命令仅对内部员工可见

## 具体技术实现

### 命令结构定义

```typescript
const tag = {
  type: 'local-jsx',           // 本地 JSX 命令类型
  name: 'tag',                 // 命令名称
  description: 'Toggle a searchable tag on the current session',
  isEnabled: () => process.env.USER_TYPE === 'ant',  // 权限控制
  argumentHint: '<tag-name>',  // 参数提示
  load: () => import('./tag.js'),  // 懒加载实现
} satisfies Command
```

### 关键字段说明

| 字段 | 值 | 说明 |
|------|-----|------|
| `type` | `'local-jsx'` | 表示这是一个本地 JSX 命令，使用 Ink 渲染交互式 UI |
| `isEnabled` | `() => process.env.USER_TYPE === 'ant'` | 动态启用检查，仅内部员工可见 |
| `argumentHint` | `'<tag-name>'` | 在命令补全时显示参数提示 |
| `load` | 懒加载函数 | 延迟加载 `./tag.js` 模块，优化启动性能 |

### 命令注册

在 `src/commands.ts` 中导入并注册：
```typescript
import tag from './commands/tag/index.js'
// ...
const COMMANDS = memoize((): Command[] => [
  // ...
  tag,
  // ...
])
```

## 关键代码路径与文件引用

### 当前文件
- **路径**: `src/commands/tag/index.ts`
- **大小**: 321 bytes
- **职责**: 命令定义和元数据配置

### 依赖文件

| 文件路径 | 用途 |
|---------|------|
| `src/commands/tag/tag.tsx` | 实际命令实现，包含 UI 组件和业务逻辑 |
| `src/commands.ts` | 命令注册中心，导入并导出所有可用命令 |
| `src/types/command.ts` | `Command` 类型定义 |

### 调用关系

```
src/commands/tag/index.ts
    ├── imports: src/commands.js (Command type)
    ├── lazy-loads: ./tag.js (实际实现)
    └── registered-by: src/commands.ts
```

## 依赖与外部交互

### 类型依赖
- `Command` 类型来自 `src/commands.js`，包含命令的基础结构和类型定义

### 运行时依赖
- 通过 `load()` 函数懒加载 `./tag.js`，实际执行业务逻辑
- 依赖 `process.env.USER_TYPE` 环境变量进行权限控制

### 外部系统交互
- 无直接外部 API 调用
- 权限控制完全基于本地环境变量

## 风险、边界与改进建议

### 潜在风险

1. **权限绕过风险**
   - 当前仅通过 `USER_TYPE` 环境变量控制访问
   - 建议：考虑增加更细粒度的权限检查机制

2. **懒加载失败**
   - 如果 `./tag.js` 文件缺失或损坏，命令将无法正常执行
   - 建议：增加加载失败的错误处理和用户提示

3. **环境变量依赖**
   - 功能完全依赖 `USER_TYPE` 环境变量
   - 建议：考虑增加配置项或 feature flag 作为备选控制机制

### 边界情况

1. **空标签处理**：实际实现在 `tag.tsx` 中处理空标签输入
2. **并发标签操作**：多会话同时操作标签的竞态条件在 `sessionStorage.ts` 中处理
3. **标签持久化**：标签数据持久化到 JSONL 文件，具体实现在 `saveTag()` 函数中

### 改进建议

1. **错误处理增强**
   ```typescript
   load: async () => {
     try {
       return await import('./tag.js')
     } catch (error) {
       logError(error)
       throw new Error('Failed to load tag command implementation')
     }
   }
   ```

2. **权限控制扩展**
   - 考虑支持基于用户角色的细粒度权限控制
   - 增加配置项允许特定用户组访问

3. **文档完善**
   - 在 `argumentHint` 中增加更多使用示例
   - 考虑添加 `aliases` 支持快捷命令（如 `/t`）

4. **类型安全**
   - 当前使用 `satisfies Command` 确保类型兼容性
   - 建议保持与 `Command` 类型的同步更新

### 相关测试建议

- 测试 `isEnabled()` 在不同 `USER_TYPE` 环境下的返回值
- 测试懒加载机制的正确性
- 测试命令在命令列表中的正确显示/隐藏
