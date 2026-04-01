# constants.ts 研究文档

## 场景与职责

`constants.ts` 是 ToolSearchTool 模块的常量定义文件，职责单一且明确：
- 定义 ToolSearchTool 的工具名称常量
- 作为模块间共享的单一事实来源（Single Source of Truth）
- 避免魔法字符串在代码库中分散使用

## 功能点目的

该文件只导出一个常量：

```typescript
export const TOOL_SEARCH_TOOL_NAME = 'ToolSearch'
```

这个常量用于：
1. **工具注册**：在 `ToolSearchTool.ts` 中作为工具的 `name` 属性
2. **延迟工具判断**：在 `prompt.ts` 的 `isDeferredTool()` 函数中排除 ToolSearchTool 本身被延迟加载
3. **外部引用**：其他模块（如 `src/utils/toolSearch.ts`）需要识别 ToolSearchTool 时使用

## 具体技术实现

### 代码结构

```typescript
export const TOOL_SEARCH_TOOL_NAME = 'ToolSearch'
```

### 使用位置

| 文件 | 使用方式 | 用途 |
|------|----------|------|
| `ToolSearchTool.ts` | `import { TOOL_SEARCH_TOOL_NAME } from './prompt.js'` | 设置工具名称 |
| `prompt.ts` | `import { TOOL_SEARCH_TOOL_NAME } from './constants.js'` | 排除自身延迟加载 |
| `src/utils/toolSearch.ts` | `import { TOOL_SEARCH_TOOL_NAME } from '../tools/ToolSearchTool/prompt.js'` | 检查工具可用性 |

### 导入路径说明

注意 `prompt.ts` 同时从 `./constants.js` 和 `./prompt.js` 导出了 `TOOL_SEARCH_TOOL_NAME`：

```typescript
// prompt.ts 第 23-25 行
export { TOOL_SEARCH_TOOL_NAME } from './constants.js'
import { TOOL_SEARCH_TOOL_NAME } from './constants.js'
```

这意味着外部模块可以通过两种方式导入：
- `from './constants.js'` - 直接导入，无额外依赖
- `from './prompt.js'` - 间接导入，会加载 prompt.ts 的全部依赖

## 关键代码路径与文件引用

```
src/tools/ToolSearchTool/
├── constants.ts          # 本文件 - 常量定义
├── prompt.ts             # 导入并重新导出 TOOL_SEARCH_TOOL_NAME
└── ToolSearchTool.ts     # 通过 prompt.ts 导入使用
```

外部引用：
```
src/utils/toolSearch.ts   # 导入 TOOL_SEARCH_TOOL_NAME 用于工具可用性检查
```

## 依赖与外部交互

### 无运行时依赖

该文件是纯粹的常量定义，没有任何导入语句，也没有副作用。

### 被依赖关系

- `prompt.ts`：导入并重新导出
- `ToolSearchTool.ts`：通过 `prompt.js` 导入使用
- `src/utils/toolSearch.ts`：通过 `prompt.js` 导入使用

## 风险、边界与改进建议

### 潜在风险

1. **命名不一致风险**
   - 风险：如果常量值被修改，但其他地方的硬编码字符串未同步更新
   - 当前：值为 `'ToolSearch'`，与文件名一致
   - 缓解：所有使用方都应通过此常量导入，不使用硬编码

2. **循环导入风险**
   - 风险：虽然当前文件无依赖，但如果未来添加依赖可能引发循环
   - 当前：安全，无导入语句
   - 建议：保持此文件纯净，不添加任何导入

### 边界情况

1. **常量值变更影响**
   - 如果 `'ToolSearch'` 被修改，会影响：
     - 模型调用工具时使用的名称
     - 权限规则中引用的工具名
     - 分析事件中记录的工具名
   - 建议：此值应视为不可变，修改需谨慎

2. **大小写敏感性**
   - 工具名称是大小写敏感的
   - 模型必须使用完全匹配的 `ToolSearch` 才能调用

### 改进建议

1. **添加类型约束**
   ```typescript
   export const TOOL_SEARCH_TOOL_NAME: 'ToolSearch' = 'ToolSearch'
   ```
   或使用 const assertion：
   ```typescript
   export const TOOL_SEARCH_TOOL_NAME = 'ToolSearch' as const
   ```

2. **考虑添加 JSDoc 注释**
   ```typescript
   /**
    * ToolSearchTool 的正式名称。
    * 用于工具注册、延迟加载排除和工具可用性检查。
    * @readonly
    */
   export const TOOL_SEARCH_TOOL_NAME = 'ToolSearch'
   ```

3. **文件大小考虑**
   - 当前文件仅 50 字节，非常精简
   - 如果未来需要添加更多 ToolSearchTool 相关常量，此文件是合适的位置
   - 如果常量数量增长，可考虑按类别拆分

4. **与 prompt.ts 的关系**
   - 当前 `prompt.ts` 重新导出此常量，增加了间接性
   - 建议：外部模块可直接从 `constants.js` 导入以减少依赖
   - 权衡：从 `prompt.js` 导入可以确保相关常量的集中管理
