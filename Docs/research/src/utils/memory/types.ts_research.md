# Research: src/utils/memory/types.ts

## 场景与职责

该文件定义了 Claude Code 中 Memory（记忆/上下文）系统的核心类型体系。它负责声明所有受支持的记忆类型（MemoryType），用于区分不同来源和用途的 CLAUDE.md 文件。这些类型决定了记忆文件的加载顺序、优先级以及可见范围。

记忆系统是 Claude Code 的核心功能之一，允许用户通过不同类型的记忆文件（CLAUDE.md、CLAUDE.local.md 等）向 AI 提供项目特定的上下文和指导。

## 功能点目的

1. **记忆类型枚举定义**：定义了 5 种（或 6 种，当 TEAMMEM 特性开启时）记忆类型：
   - `User`：用户级全局记忆（~/.claude/CLAUDE.md）
   - `Project`：项目级记忆（CLAUDE.md, .claude/CLAUDE.md）
   - `Local`：本地私有记忆（CLAUDE.local.md）
   - `Managed`：托管策略记忆（/etc/claude-code/CLAUDE.md）
   - `AutoMem`：自动记忆系统入口
   - `TeamMem`（可选）：团队共享记忆（需 TEAMMEM 特性开关）

2. **运行时特性开关集成**：通过 `bun:bundle` 的 `feature()` 函数动态控制 TeamMem 类型的可用性，支持功能灰度发布。

3. **类型安全导出**：提供 TypeScript 类型 `MemoryType`，确保类型安全地在整个代码库中传递记忆类型信息。

## 具体技术实现

### 关键数据结构

```typescript
// 记忆类型值数组，使用 const assertion 确保类型收窄
export const MEMORY_TYPE_VALUES = [
  'User',
  'Project', 
  'Local',
  'Managed',
  'AutoMem',
  ...(feature('TEAMMEM') ? (['TeamMem'] as const) : []),
] as const

// 派生类型：'User' | 'Project' | 'Local' | 'Managed' | 'AutoMem' | 'TeamMem'?
export type MemoryType = (typeof MEMORY_TYPE_VALUES)[number]
```

### 特性开关机制

- 使用 `bun:bundle` 提供的 `feature('TEAMMEM')` 进行编译时/运行时特性检测
- 通过条件展开运算符 `...` 将可选类型动态注入数组
- 使用 `as const` 断言确保数组元素被推断为字面量类型而非 string

## 关键代码路径与文件引用

### 被以下文件引用（调用方）

| 文件路径 | 引用方式 | 用途 |
|---------|---------|------|
| `src/utils/config.ts` | `import type { MemoryType } from './memory/types.js'` | 配置系统中处理记忆路径 |
| `src/utils/claudemd.ts` | `import type { MemoryType } from './memory/types.js'` | 核心记忆文件加载逻辑 |
| `src/utils/hooks.ts` | `import type { InstructionsMemoryType } from './claudemd.js'` → 间接使用 | 指令加载钩子 |
| `src/utils/frontmatterParser.ts` | 通过 claudemd.ts 间接使用 | Frontmatter 解析 |
| `src/utils/attachments.ts` | 通过 claudemd.ts 间接使用 | 附件生成 |
| `src/memdir/memoryTypes.ts` | 独立定义（见下文） | 记忆类型分类系统 |
| `src/memdir/memoryScan.ts` | `import { type MemoryType, parseMemoryType } from './memoryTypes.js'` | 记忆文件扫描 |
| `src/services/compact/compact.ts` | `import { MEMORY_TYPE_VALUES } from '../../utils/memory/types.js'` | 压缩服务中处理记忆 |
| `src/components/PromptInput/PromptInputFooterLeftSide.tsx` | 通过 claudemd.ts 间接使用 | UI 组件 |
| `src/components/memory/MemoryFileSelector.tsx` | 通过 claudemd.ts 间接使用 | 记忆文件选择器 UI |

### 相关文件对比

**注意**：存在两个不同的 `MemoryType` 定义：

1. **本文件** (`src/utils/memory/types.ts`)：定义记忆文件的*来源类型*（User/Project/Local/Managed/AutoMem/TeamMem），用于文件加载优先级和路径解析。

2. **`src/memdir/memoryTypes.ts`**：定义记忆内容的*语义类型*（user/feedback/project/reference），用于记忆分类和提取策略。

这两个类型系统相互独立但协同工作：
- 来源类型决定**从哪里加载**记忆文件
- 语义类型决定**如何分类**记忆内容

## 依赖与外部交互

### 直接依赖

| 依赖 | 来源 | 用途 |
|-----|------|------|
| `bun:bundle` | Bun 运行时 | 特性开关 `feature('TEAMMEM')` |

### 间接依赖（通过调用方）

- `src/utils/git.ts`：通过 `findGitRoot` 确定项目根目录
- `src/utils/config.ts`：获取记忆文件路径配置
- `src/utils/fsOperations.ts`：文件系统操作

## 风险、边界与改进建议

### 已知风险

1. **类型命名冲突**：`src/utils/memory/types.ts` 和 `src/memdir/memoryTypes.ts` 都定义了 `MemoryType`，虽然用途不同，但容易造成混淆。开发者需要明确区分：
   - 文件系统层面的记忆来源类型（本文件）
   - 内容层面的记忆语义类型（memdir/memoryTypes.ts）

2. **特性开关耦合**：TeamMem 类型的可用性完全依赖 `feature('TEAMMEM')`，如果特性开关实现有 bug，可能导致类型定义与运行时行为不一致。

3. **编译时优化依赖**：`as const` 和条件展开依赖 TypeScript 的类型收窄能力，如果未来 TypeScript 版本变更可能影响行为。

### 边界情况

1. **空数组处理**：当 TEAMMEM 关闭时，`MEMORY_TYPE_VALUES` 仍包含 5 个固定元素，类型系统正确处理。

2. **大小写敏感**：记忆类型使用 PascalCase（User, Project 等），与 memdir/memoryTypes.ts 中的 camelCase（user, project 等）不同，这是有意的设计区分。

### 改进建议

1. **重命名消除歧义**：考虑将本文件的 `MemoryType` 重命名为 `MemorySourceType` 或 `MemoryLocationType`，以明确区分于 memdir 的语义类型。

2. **文档增强**：添加 JSDoc 注释说明每个记忆类型的具体用途和加载顺序。

3. **类型守卫**：考虑添加运行时类型守卫函数（如 `isMemoryType(value): value is MemoryType`），目前调用方需要自行验证。

4. **特性开关抽象**：考虑将特性开关逻辑抽象到配置层，而非直接散落在类型定义中。
