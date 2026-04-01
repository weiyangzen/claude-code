# memoryTypes.ts 研究文档

## 场景与职责

`memoryTypes.ts` 是记忆系统的**类型定义和提示词模板中心**，定义了记忆类型分类学（taxonomy）并提供了完整的提示词章节模板。这是记忆系统的"知识库"，规定了什么应该被记住、如何分类、以及如何指导模型使用记忆。

### 核心职责
1. **记忆类型定义**：定义四种核心记忆类型（user/feedback/project/reference）
2. **类型解析**：将原始 frontmatter 值解析为类型枚举
3. **提示词模板**：提供完整的提示词章节（类型说明、保存指南、访问时机等）
4. **双模式支持**：支持个人模式（INDIVIDUAL）和团队模式（COMBINED）的提示词变体

### 使用场景
- `memdir.ts` 构建记忆提示词时导入模板
- `teamMemPrompts.ts` 构建团队记忆提示词
- `memoryScan.ts` 解析记忆类型
- 任何需要引用记忆类型系统的组件

---

## 功能点目的

### 1. 记忆类型定义

#### MEMORY_TYPES 常量
```typescript
export const MEMORY_TYPES = ['user', 'feedback', 'project', 'reference'] as const
```

**设计原则**：
- 封闭式分类（closed taxonomy）
- 只捕获无法从当前项目状态推导的信息
- 明确排除：代码模式、架构、文件路径、Git 历史等

### 2. `parseMemoryType()` - 类型解析
**目的**：将 frontmatter 中的原始值解析为类型枚举

**行为**：
- 无效或缺失值返回 `undefined`（向后兼容）
- 未知类型优雅降级

### 3. `TYPES_SECTION_COMBINED` - 团队模式类型说明
**目的**：为团队记忆场景提供完整的四种类型说明

**包含内容**：
- 每种类型的 `<scope>` 标签（private/team）
- 详细的 `<description>`、`<when_to_save>`、`<how_to_use>`
- 具体的对话示例
- `<body_structure>` 指导（feedback/project 类型）

### 4. `TYPES_SECTION_INDIVIDUAL` - 个人模式类型说明
**目的**：为单用户场景提供简化的类型说明

**差异点**：
- 无 `<scope>` 标签
- 示例使用 `[saves X memory: …]` 格式（无 private/team 限定）
- 移除了团队相关的措辞

### 5. `WHAT_NOT_TO_SAVE_SECTION` - 不应保存的内容
**目的**：明确排除不应保存为记忆的信息类型

**排除项**：
- 代码模式、约定、架构、文件路径
- Git 历史、最近更改
- 调试解决方案
- CLAUDE.md 中已有的内容
- 临时任务详情

**显式保存门控**：即使用户明确要求保存 PR 列表或活动摘要，也应询问其中的"惊喜"或"非显而易见"之处。

### 6. `WHEN_TO_ACCESS_SECTION` - 访问时机
**目的**：指导模型何时应该访问记忆

**关键要点**：
- 记忆相关或用户引用先前工作时
- 用户明确要求检查/回忆/记住时**必须**访问
- 用户要求忽略记忆时，完全跳过（不引用、不比较、不提及）
- `MEMORY_DRIFT_CAVEAT`：记忆可能过时，使用前验证

### 7. `TRUSTING_RECALL_SECTION` - 信任召回内容
**目的**： heavier-weight 指导，说明如何处理已召回的记忆

**关键指导**：
- 记忆命名特定函数/文件/标志时，先验证其存在
- 用户即将基于建议行动时，先验证
- 记忆总结仓库状态时，优先使用 `git log` 或读取代码

### 8. `MEMORY_FRONTMATTER_EXAMPLE` - Frontmatter 示例
**目的**：提供标准的记忆文件 frontmatter 格式示例

**格式**：
```yaml
---
name: {{memory name}}
description: {{one-line description}}
type: {{user, feedback, project, reference}}
---
```

---

## 具体技术实现

### 类型系统

```typescript
export const MEMORY_TYPES = [
  'user',
  'feedback', 
  'project',
  'reference'
] as const

export type MemoryType = (typeof MEMORY_TYPES)[number]

export function parseMemoryType(raw: unknown): MemoryType | undefined {
  if (typeof raw !== 'string') return undefined
  return MEMORY_TYPES.find(t => t === raw)
}
```

### 提示词模板结构

所有提示词章节都是 `readonly string[]` 数组，便于拼接：

```typescript
export const TYPES_SECTION_COMBINED: readonly string[] = [
  '## Types of memory',
  '',
  'There are several discrete types...',
  // ...
]

export const WHAT_NOT_TO_SAVE_SECTION: readonly string[] = [
  '## What NOT to save in memory',
  '',
  '- Code patterns...',
  // ...
]
```

### 团队 vs 个人模式差异

| 方面 | COMBINED | INDIVIDUAL |
|------|----------|------------|
| Scope 标签 | 有 `<scope>` | 无 |
| 示例格式 | `[saves private/team X memory: …]` | `[saves X memory: …]` |
| 团队协调 | 提及跨用户协调 | 仅关注当前用户 |
| 使用场景 | 团队记忆启用时 | 仅个人记忆 |

---

## 关键代码路径与文件引用

### 被调用方
| 文件 | 用途 |
|------|------|
| `src/memdir/memdir.ts` | `TYPES_SECTION_INDIVIDUAL`, `WHAT_NOT_TO_SAVE_SECTION`, `WHEN_TO_ACCESS_SECTION`, `TRUSTING_RECALL_SECTION`, `MEMORY_FRONTMATTER_EXAMPLE` |
| `src/memdir/teamMemPrompts.ts` | `TYPES_SECTION_COMBINED`, `WHAT_NOT_TO_SAVE_SECTION`, `MEMORY_DRIFT_CAVEAT`, `TRUSTING_RECALL_SECTION`, `MEMORY_FRONTMATTER_EXAMPLE` |
| `src/memdir/memoryScan.ts` | `MemoryType`, `parseMemoryType()` |
| `src/utils/memory/types.ts` | 可能导入 `MemoryType` |

### 无内部依赖
该模块是纯定义模块，不依赖其他模块。

### 无外部依赖
仅使用 TypeScript 类型系统。

---

## 依赖与外部交互

### 纯数据模块
- 所有导出都是常量或纯函数
- 无副作用
- 不依赖运行时状态

### 设计文档引用
代码注释中引用了多个设计决策和评估结果：
- `memory-prompt-iteration.eval.ts` - 提示词迭代评估
- `#22856` - 分支污染评估
- `#25372` - 循环依赖修复

---

## 风险、边界与改进建议

### 已知风险

1. **提示词膨胀**
   - 完整的类型说明非常长（数百行）
   - 每个会话都要加载到系统提示词中
   - 可能占用大量上下文窗口

2. **僵化分类**
   - 四种类型可能无法覆盖所有场景
   - 用户可能有自定义需求

3. **翻译成本**
   - 提示词模板是英文的
   - 多语言支持需要完整重写

4. **评估依赖**
   - 许多设计决策基于特定评估案例
   - 新场景可能需要重新评估

### 边界情况

| 场景 | 处理 |
|------|------|
| Frontmatter 无 type 字段 | `parseMemoryType` 返回 `undefined` |
| Frontmatter type 为未知值 | 返回 `undefined`，优雅降级 |
| Type 字段为 null | 返回 `undefined` |
| 需要添加新类型 | 需修改 `MEMORY_TYPES` 数组和所有模板 |

### 改进建议

1. **动态提示词长度**
   ```typescript
   // 根据上下文窗口大小选择详细程度
   export function getTypesSection(mode: 'combined', detail: 'full' | 'brief'): readonly string[]
   ```

2. **可扩展类型系统**
   ```typescript
   // 支持自定义类型（需谨慎）
   export const EXTENSIBLE_MEMORY_TYPES = [...MEMORY_TYPES, 'custom'] as const
   ```

3. **国际化支持**
   ```typescript
   // 多语言模板
   export const TYPES_SECTION_COMBINED_I18N: Record<Locale, readonly string[]>
   ```

4. **版本化模板**
   ```typescript
   // 支持模板版本控制，便于 A/B 测试
   export const TYPES_SECTION_COMBINED_V2: readonly string[]
   ```

5. **运行时配置**
   ```typescript
   // 允许通过 feature flag 启用/禁用某些章节
   export function buildTypesSection(options: TypesSectionOptions): readonly string[]
   ```

6. **评估追踪**
   ```typescript
   // 在注释中链接到具体的评估案例
   // @eval memory-prompt-iteration case 3
   ```

### 维护建议

1. **同步更新**：修改 COMBINED 模板时，检查是否需要同步更新 INDIVIDUAL 模板
2. **评估驱动**：任何提示词修改都应通过评估验证
3. **文档化决策**：在注释中记录设计决策的原因和评估结果
4. **定期审查**：随着模型能力变化，定期审查提示词有效性
