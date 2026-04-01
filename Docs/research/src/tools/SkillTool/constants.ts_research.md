# constants.ts 研究文档

## 场景与职责

`constants.ts` 是 SkillTool 模块的**常量定义文件**，职责单一明确：

1. **定义工具名称常量** - 提供统一的工具标识符，避免魔法字符串
2. **支持跨文件引用** - 被 `SkillTool.ts` 和 `UI.tsx` 共享
3. **便于重构维护** - 工具名称变更只需修改一处

## 功能点目的

该文件仅导出一个常量：

```typescript
export const SKILL_TOOL_NAME = 'Skill'
```

此常量用于：
- 工具注册时的名称标识
- 遥测事件中的工具类型标记
- 工具使用 ID 生成（`getToolUseIDFromParentMessage`）
- 权限规则匹配

## 具体技术实现

### 常量定义

```typescript
export const SKILL_TOOL_NAME = 'Skill'
```

- **类型**: 字符串字面量
- **命名规范**: 大写下划线（SCREAMING_SNAKE_CASE）
- **导出方式**: 命名导出（named export）

## 关键代码路径与文件引用

### 被引用位置

| 文件路径 | 引用方式 | 用途 |
|---------|---------|------|
| `src/tools/SkillTool/SkillTool.ts` | `import { SKILL_TOOL_NAME } from './constants.js'` | 工具名称定义、遥测标记 |
| `src/tools/SkillTool/UI.tsx` | 未直接引用 | （通过 SkillTool.ts 间接使用）|

### 引用代码示例

```typescript
// SkillTool.ts 第 67 行
import { SKILL_TOOL_NAME } from './constants.js'

// SkillTool.ts 第 332 行
export const SkillTool: Tool<InputSchema, Output, Progress> = buildTool({
  name: SKILL_TOOL_NAME,
  // ...
})

// SkillTool.ts 第 729-732 行
const toolUseID = getToolUseIDFromParentMessage(
  parentMessage,
  SKILL_TOOL_NAME,
)
```

## 依赖与外部交互

该文件无任何外部依赖，是纯常量定义文件。

## 风险、边界与改进建议

### 风险分析

1. **命名冲突风险**
   - 当前常量名 `SKILL_TOOL_NAME` 较为通用
   - 如项目中有多个 SkillTool 实现可能产生混淆
   - 风险等级：低（模块作用域隔离）

2. **重构遗漏风险**
   - 如直接硬编码 'Skill' 字符串而未使用常量
   - 当前代码库中已通过 ESLint/类型检查确保一致性

### 边界条件

该文件无运行时边界条件，仅包含编译时常量。

### 改进建议

1. **扩展常量定义**
   当前文件过于简单，可考虑合并其他 SkillTool 相关常量：
   
   ```typescript
   // 建议添加（如需要）
   export const SKILL_TOOL_NAME = 'Skill'
   export const SKILL_TOOL_SEARCH_HINT = 'invoke a slash-command skill'
   export const SKILL_TOOL_MAX_RESULT_SIZE = 100_000
   ```

2. **类型安全增强**
   ```typescript
   export const SKILL_TOOL_NAME = 'Skill' as const
   // 或使用满足类型
   export type SkillToolName = typeof SKILL_TOOL_NAME  // 'Skill'
   ```

3. **文档注释**
   ```typescript
   /**
    * SkillTool 的工具名称标识符。
    * 用于工具注册、遥测标记和权限规则匹配。
    * @see src/tools/SkillTool/SkillTool.ts
    */
   export const SKILL_TOOL_NAME = 'Skill'
   ```

4. **考虑合并**
   - 如常量数量始终很少，可考虑直接内联到主文件
   - 当前分离模式有利于测试和模块化，建议保持

### 维护建议

- 变更工具名称时需同步检查：
  1. 权限系统中的规则匹配
  2. 遥测事件中的工具标识
  3. 文档和用户手册中的引用
  4. 测试用例中的断言

- 该文件变更频率预计极低，适合作为稳定接口维护
