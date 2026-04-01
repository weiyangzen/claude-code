# teamMemPrompts.ts 研究文档

## 场景与职责

`teamMemPrompts.ts` 是团队记忆系统的**提示词构建模块**，专门负责构建当同时启用自动记忆和团队记忆时的组合提示词。它定义了双目录记忆系统的使用指南，指导模型如何在私人记忆和团队记忆之间做出选择。

### 核心职责
1. **组合提示词构建**：生成同时包含私人记忆和团队记忆的统一提示词
2. **作用域指导**：为每种记忆类型提供 private/team 选择指导
3. **双索引管理**：指导模型维护两个 MEMORY.md 索引（私人 + 团队）
4. **安全提醒**：强调不在团队记忆中保存敏感数据

### 使用场景
- 用户同时启用自动记忆和团队记忆时
- 系统提示词初始化时加载组合记忆指南
- 需要区分私人记忆和团队记忆保存位置的场景

---

## 功能点目的

### 1. `buildCombinedMemoryPrompt()` - 组合记忆提示词构建
**目的**：当自动记忆和团队记忆都启用时，构建统一的记忆系统提示词

**关键参数**：
- `extraGuidelines`: 额外的指导原则（来自 Cowork 环境变量）
- `skipIndex`: 是否跳过索引步骤（feature flag 控制）

**返回值**：完整的记忆系统提示词字符串

### 提示词结构

组合提示词包含以下章节：

1. **标题和介绍**
   - 说明有两个记忆目录：私人目录和团队目录
   - `DIRS_EXIST_GUIDANCE`：告知目录已存在，可直接写入

2. **记忆作用域**
   - `private`: 仅当前用户可见，跨对话持久化
   - `team`: 项目内所有用户共享，每会话同步

3. **类型说明** (`TYPES_SECTION_COMBINED`)
   - 四种记忆类型的详细说明
   - 每种类型包含 `<scope>` 标签（always private / default to private / bias toward team / usually team）
   - 具体对话示例

4. **不应保存的内容** (`WHAT_NOT_TO_SAVE_SECTION`)
   - 与 `memoryTypes.ts` 共享的排除项
   - 额外强调：不要在团队记忆中保存敏感数据

5. **保存方法**
   - 单步模式（skipIndex）：直接写入文件
   - 两步模式：写入文件 + 更新 MEMORY.md 索引
   - 两个目录各自有独立的 MEMORY.md 索引

6. **访问时机** (`WHEN_TO_ACCESS_SECTION` 变体)
   - 个人或团队记忆相关时
   - 用户明确要求时
   - 忽略记忆的显式指令

7. **信任召回内容** (`TRUSTING_RECALL_SECTION`)
   - 验证记忆内容前的检查清单

8. **与其他持久化机制的对比**
   - Plan vs Memory
   - Task vs Memory

9. **额外指导** (`extraGuidelines`)
   - 来自 `CLAUDE_COWORK_MEMORY_EXTRA_GUIDELINES` 环境变量

10. **搜索历史上下文** (`buildSearchingPastContextSection`)
    - 如何搜索记忆目录和会话日志

---

## 具体技术实现

### 函数实现

```typescript
export function buildCombinedMemoryPrompt(
  extraGuidelines?: string[],
  skipIndex = false,
): string {
  const autoDir = getAutoMemPath()    // 私人记忆目录
  const teamDir = getTeamMemPath()    // 团队记忆目录

  const howToSave = skipIndex
    ? [/* 单步保存指南 */]
    : [/* 两步保存指南 */]

  const lines = [
    '# Memory',
    '',
    `You have a persistent, file-based memory system with two directories... ${DIRS_EXIST_GUIDANCE}`,
    '',
    '## Memory scope',
    '',
    `- private: ...`,
    `- team: ...`,
    '',
    ...TYPES_SECTION_COMBINED,      // 来自 memoryTypes.ts
    ...WHAT_NOT_TO_SAVE_SECTION,    // 来自 memoryTypes.ts
    '- You MUST avoid saving sensitive data within shared team memories...',
    '',
    ...howToSave,
    '',
    '## When to access memories',
    // ... 访问时机指南
    ...TRUSTING_RECALL_SECTION,     // 来自 memoryTypes.ts
    '',
    '## Memory and other forms of persistence',
    // ...
    ...(extraGuidelines ?? []),
    '',
    ...buildSearchingPastContextSection(autoDir),  // 来自 memdir.ts
  ]

  return lines.join('\n')
}
```

### 保存方法差异

#### 单步模式（skipIndex = true）
```markdown
## How to save memories

Write each memory to its own file in the chosen directory 
(private or team, per the type's scope guidance) using this frontmatter format:

```yaml
---
name: {{memory name}}
description: {{one-line description}}
type: {{user, feedback, project, reference}}
---
```

- Keep the name, description, and type fields up-to-date
- Organize memory semantically by topic
- Update or remove outdated memories
- Do not write duplicate memories
```

#### 两步模式（skipIndex = false）
```markdown
## How to save memories

Saving a memory is a two-step process:

**Step 1** — write the memory to its own file...

**Step 2** — add a pointer to that file in the same directory's `MEMORY.md`...

- Both `MEMORY.md` indexes are loaded into your conversation context
- Lines after 200 will be truncated
```

---

## 关键代码路径与文件引用

### 内部依赖
| 文件 | 用途 |
|------|------|
| `memdir.ts` | `buildSearchingPastContextSection()`, `DIRS_EXIST_GUIDANCE`, `ENTRYPOINT_NAME`, `MAX_ENTRYPOINT_LINES` |
| `memoryTypes.ts` | `TYPES_SECTION_COMBINED`, `WHAT_NOT_TO_SAVE_SECTION`, `MEMORY_DRIFT_CAVEAT`, `TRUSTING_RECALL_SECTION`, `MEMORY_FRONTMATTER_EXAMPLE` |
| `paths.ts` | `getAutoMemPath()` |
| `teamMemPaths.ts` | `getTeamMemPath()` |

### 外部依赖
无（纯提示词构建模块）

### 调用方
| 文件 | 用途 |
|------|------|
| `src/memdir/memdir.ts` | `buildCombinedMemoryPrompt()`（条件加载） |

---

## 依赖与外部交互

### 导入方式
在 `memdir.ts` 中通过条件 require 加载：

```typescript
/* eslint-disable @typescript-eslint/no-require-imports */
const teamMemPrompts = feature('TEAMMEM')
  ? (require('./teamMemPrompts.js') as typeof import('./teamMemPrompts.js'))
  : null
/* eslint-enable @typescript-eslint/no-require-imports */
```

这种导入方式确保：
- 仅在 `TEAMMEM` feature flag 启用时加载
- 避免不必要的模块加载和依赖解析

### 环境变量
| 变量 | 用途 |
|------|------|
| `CLAUDE_COWORK_MEMORY_EXTRA_GUIDELINES` | 注入额外的记忆策略指导 |

### Feature Flags
| Flag | 用途 |
|------|------|
| `TEAMMEM` | 控制模块是否被加载 |
| `tengu_moth_copse` | 控制 `skipIndex` 参数 |

---

## 风险、边界与改进建议

### 已知风险

1. **提示词长度**
   - 组合提示词非常长（包含两套类型说明）
   - 占用大量上下文窗口
   - 可能影响模型对其他指令的关注

2. **复杂性**
   - 两个目录、两个索引、四种类型 × 作用域组合
   - 模型可能混淆何时保存到私人 vs 团队

3. **同步延迟**
   - 团队记忆"每会话同步"的说明可能产生期望落差
   - 实际同步可能有延迟

4. **敏感数据泄露风险**
   - 虽然提示词强调不要在团队记忆中保存敏感数据
   - 模型可能误判什么是"敏感"

### 边界情况

| 场景 | 处理 |
|------|------|
| `extraGuidelines` 为空 | 不添加额外章节 |
| `skipIndex` 为 true | 使用单步保存指南 |
| 团队目录不存在 | 由调用方（`memdir.ts`）确保目录存在 |
| 两种记忆都禁用 | 不调用此函数（由 `loadMemoryPrompt` 分发） |

### 与个人模式提示词的差异

| 方面 | 组合模式（本模块） | 个人模式（memdir.ts） |
|------|-------------------|----------------------|
| 目录数量 | 2个（私人 + 团队） | 1个 |
| 类型说明 | 含 `<scope>` 标签 | 无 scope 标签 |
| 示例格式 | `[saves private/team X memory: …]` | `[saves X memory: …]` |
| 安全提醒 | 显式敏感数据警告 | 无 |
| 索引管理 | 两个独立 MEMORY.md | 单个 MEMORY.md |

### 改进建议

1. **动态简化**
   ```typescript
   // 根据用户实际使用模式简化提示词
   export function buildCombinedMemoryPrompt(
     usageStats: MemoryUsageStats,  // 用户使用统计
     options: PromptOptions,
   ): string {
     // 如果用户从不使用某种类型，简化该类型的说明
   }
   ```

2. **交互式指导**
   ```typescript
   // 首次保存到团队记忆时提供额外指导
   export function getFirstTimeTeamSaveGuidance(): string
   ```

3. **作用域决策辅助**
   ```typescript
   // 提供决策树帮助模型选择作用域
   const SCOPE_DECISION_TREE = `
   Is this memory about:
   - The user's personal preferences? → private
   - A project-wide convention? → team
   - External system pointers? → usually team
   `
   ```

4. **冲突检测提示**
   ```typescript
   // 提醒检查私人记忆是否与团队记忆冲突
   '- Before saving a private feedback memory, check that it doesn\'t contradict a team feedback memory'
   ```

5. **可配置作用域默认**
   ```typescript
   // 允许通过设置调整类型的默认作用域
   export interface ScopeDefaults {
     feedback: 'private' | 'team'
     project: 'private' | 'team'
     reference: 'private' | 'team'
   }
   ```

6. **更好的错误示例**
   ```typescript
   // 添加"不要做"的示例
   '<example type="bad">user: Save my API key to team memory</example>',
   '<example type="bad">assistant: [INCORRECT - never save credentials to team memory]</example>',
   ```
