# prompts.ts 深度研究文档

## 场景与职责

`prompts.ts` 是自动记忆提取服务的 **prompt 构建模块**，负责为 forked extraction agent 生成用户提示（user prompt）。该模块根据系统配置（是否启用团队记忆）生成两种变体的提取 prompt。

### 核心定位
- **调用方**: `extractMemories.ts` 中的 `runExtraction` 函数
- **执行时机**: 每次记忆提取前动态构建 prompt
- **设计哲学**: 分离 prompt 构建逻辑与执行逻辑，便于独立迭代和测试

### 与主系统的关系
- 提取 agent 作为主会话的 "完美分叉"（perfect fork），共享相同的 system prompt 和消息前缀
- 当主代理已写入记忆时，提取会被跳过（在 `extractMemories.ts` 中处理）
- 这里的保存标准与主代理系统 prompt 中的标准重叠，但不会冲突

---

## 功能点目的

### 1. 双模式 Prompt 生成
根据功能开关生成两种 prompt：
- **`buildExtractAutoOnlyPrompt`**: 仅个人记忆模式（单目录）
- **`buildExtractCombinedPrompt`**: 个人+团队记忆模式（双目录）

### 2. 记忆类型指导
通过导入的 `TYPES_SECTION_*` 向提取 agent 说明四种记忆类型：
- **user**: 用户信息（始终 private）
- **feedback**: 用户反馈（默认 private，项目级约定可 team）
- **project**: 项目信息（偏向 team）
- **reference**: 外部系统引用（通常 team）

### 3. 现有记忆清单注入
将 `formatMemoryManifest` 生成的记忆目录清单注入 prompt，帮助 agent：
- 了解已存在的记忆文件
- 避免创建重复记忆
- 识别需要更新的现有记忆

### 4. 工具使用指导
明确告知提取 agent 可用的工具及其限制：
- 允许的工具: `Read`, `Grep`, `Glob`, 只读 `Bash`, `Edit`/`Write`（仅限记忆目录）
- 禁止的操作: `rm`, MCP, Agent, 写操作 Bash 等

---

## 具体技术实现

### 关键流程

#### 共享开场白 (`opener`)
```typescript
function opener(newMessageCount: number, existingMemories: string): string
```
构建 prompt 的通用开头部分：
1. 角色定义："You are now acting as the memory extraction subagent"
2. 分析范围：最近的 ~N 条消息
3. 可用工具列表及限制
4. 执行策略指导（Turn 1 并行读取，Turn 2 并行写入）
5. 内容来源限制（仅使用最近消息，禁止额外调查）
6. 现有记忆清单（如存在）

#### Auto-Only Prompt 构建
```typescript
export function buildExtractAutoOnlyPrompt(
  newMessageCount: number,
  existingMemories: string,
  skipIndex = false,
): string
```

**结构组成**:
1. `opener()` - 共享开场白
2. 显式保存/遗忘指令
3. `TYPES_SECTION_INDIVIDUAL` - 四种记忆类型说明（无 scope 标签）
4. `WHAT_NOT_TO_SAVE_SECTION` - 明确排除的内容
5. `howToSave` - 保存指南（单文件 vs 两步流程）

**索引模式控制** (`skipIndex`):
- `false` (默认): 两步保存流程（写文件 + 更新 MEMORY.md 索引）
- `true`: 单步保存（仅写文件，不维护索引）

#### Combined Prompt 构建
```typescript
export function buildExtractCombinedPrompt(
  newMessageCount: number,
  existingMemories: string,
  skipIndex = false,
): string
```

**特殊处理**:
- 检查 `TEAMMEM` feature flag，未启用时回退到 `buildExtractAutoOnlyPrompt`
- 使用 `TYPES_SECTION_COMBINED`（带 `<scope>` 标签的类型说明）
- 额外的安全警告："You MUST avoid saving sensitive data within shared team memories"
- 目录选择指导：根据类型 scope 选择 private 或 team 目录

### 关键数据结构

#### Prompt 内容片段来源
```typescript
// 从 memoryTypes.ts 导入
import {
  MEMORY_FRONTMATTER_EXAMPLE,    // Frontmatter 格式示例
  TYPES_SECTION_COMBINED,        // 团队模式类型说明（含 scope）
  TYPES_SECTION_INDIVIDUAL,      // 个人模式类型说明
  WHAT_NOT_TO_SAVE_SECTION,      // 排除内容说明
} from '../../memdir/memoryTypes.js'

// 工具名称常量
import { BASH_TOOL_NAME } from '../../tools/BashTool/toolName.js'
import { FILE_EDIT_TOOL_NAME } from '../../tools/FileEditTool/constants.js'
import { FILE_READ_TOOL_NAME } from '../../tools/FileReadTool/prompt.js'
import { FILE_WRITE_TOOL_NAME } from '../../tools/FileWriteTool/prompt.js'
import { GLOB_TOOL_NAME } from '../../tools/GlobTool/prompt.js'
import { GREP_TOOL_NAME } from '../../tools/GrepTool/prompt.js'
```

### 保存指南差异

#### 带索引模式 (`skipIndex = false`)
```markdown
## How to save memories

Saving a memory is a two-step process:

**Step 1** — write the memory to its own file...

**Step 2** — add a pointer to that file in `MEMORY.md`...

- `MEMORY.md` is always loaded into your system prompt — lines after 200 will be truncated
```

#### 无索引模式 (`skipIndex = true`)
```markdown
## How to save memories

Write each memory to its own file...
```

---

## 关键代码路径与文件引用

### 核心文件
| 文件 | 职责 |
|------|------|
| `src/services/extractMemories/prompts.ts` | Prompt 构建实现 |
| `src/services/extractMemories/extractMemories.ts` | 调用方，传入参数并执行提取 |

### 依赖文件
| 文件 | 用途 |
|------|------|
| `src/memdir/memoryTypes.ts` | 记忆类型定义、frontmatter 示例、类型说明文本 |
| `src/tools/BashTool/toolName.ts` | `BASH_TOOL_NAME` 常量 |
| `src/tools/FileEditTool/constants.ts` | `FILE_EDIT_TOOL_NAME` 常量 |
| `src/tools/FileReadTool/prompt.ts` | `FILE_READ_TOOL_NAME` 常量 |
| `src/tools/FileWriteTool/prompt.ts` | `FILE_WRITE_TOOL_NAME` 常量 |
| `src/tools/GlobTool/prompt.ts` | `GLOB_TOOL_NAME` 常量 |
| `src/tools/GrepTool/prompt.ts` | `GREP_TOOL_NAME` 常量 |

### 调用链
```
runExtraction (extractMemories.ts:329)
  └── buildExtractAutoOnlyPrompt / buildExtractCombinedPrompt (prompts.ts:50/101)
        └── opener (prompts.ts:29)
```

### 参数流向
```typescript
// extractMemories.ts 中的调用
const userPrompt =
  feature('TEAMMEM') && teamMemoryEnabled
    ? buildExtractCombinedPrompt(
        newMessageCount,      // 自上次提取以来的新消息数
        existingMemories,     // formatMemoryManifest 输出
        skipIndex,            // 是否跳过索引更新
      )
    : buildExtractAutoOnlyPrompt(...)
```

---

## 依赖与外部交互

### 外部依赖

#### 1. Feature Flag (Bun bundle)
```typescript
import { feature } from 'bun:bundle'
```
用于条件性导入和团队记忆功能开关检查。

#### 2. 记忆类型定义 (`src/memdir/memoryTypes.ts`)
导出的关键常量：
- `MEMORY_FRONTMATTER_EXAMPLE`: Frontmatter YAML 格式示例
- `TYPES_SECTION_COMBINED`: XML 风格的类型说明（含 `<scope>` 标签）
- `TYPES_SECTION_INDIVIDUAL`: 纯文本类型说明
- `WHAT_NOT_TO_SAVE_SECTION`: 排除列表

#### 3. 工具名称常量
从各工具模块导入的工具标识符，用于在 prompt 中明确列出可用工具。

### 无外部服务调用
本模块纯为字符串构建逻辑，无网络、文件系统或数据库交互。

---

## 风险、边界与改进建议

### 已知风险

#### 1. Prompt 膨胀
- **风险**: `existingMemories` 清单可能很长，导致 prompt 过大
- **现状**: `memoryScan.ts` 中 `MAX_MEMORY_FILES = 200` 限制扫描数量
- **缓解**: 按修改时间排序，仅返回最新的 200 个文件

#### 2. Feature Flag 不一致
- **风险**: `TEAMMEM` flag 在运行时动态检查，可能与 `extractMemories.ts` 中的检查不一致
- **现状**: `buildExtractCombinedPrompt` 内部有防御性检查，未启用时回退到 auto-only

#### 3. 工具名称硬编码
- **风险**: 工具重命名后 prompt 中的名称与实际不匹配
- **现状**: 使用从各模块导入的常量，修改时会同步更新

### 边界情况

#### 1. 空记忆目录
```typescript
const manifest =
  existingMemories.length > 0
    ? `\n\n## Existing memory files\n\n${existingMemories}\n\nCheck this list before writing...`
    : ''
```
当目录为空时，不显示现有记忆部分。

#### 2. Skip Index 模式
- 用于特定场景（如 `tengu_moth_copse` feature 启用）
- 简化保存流程，适合不需要索引的用例

### 改进建议

#### 1. Prompt 版本控制
- 当前 prompt 文本分散在多个文件中
- 建议：添加版本标识，便于 A/B 测试和回滚

#### 2. 动态长度控制
- `existingMemories` 长度可能波动较大
- 建议：添加 token 预算控制，超限时截断或摘要

#### 3. 国际化准备
- 当前所有文本硬编码为英文
- 建议：虽然记忆系统面向开发者，但可考虑提取文本到配置

#### 4. 类型安全增强
- 当前 `skipIndex` 为布尔值，含义不够明确
- 建议：使用枚举或更具描述性的参数名

#### 5. Prompt 缓存优化
- 每次提取都重新构建 prompt，无法利用缓存
- 建议：对于相同的 `newMessageCount` 和 `existingMemories` 指纹，考虑缓存

### 测试建议

#### 1. Prompt 快照测试
```typescript
// 建议添加的测试模式
expect(buildExtractAutoOnlyPrompt(10, '', false)).toMatchSnapshot()
expect(buildExtractCombinedPrompt(10, 'existing...', false)).toMatchSnapshot()
```

#### 2. 内容包含验证
- 验证所有必需部分存在（工具列表、类型说明、保存指南）
- 验证 `skipIndex` 模式正确影响保存指南

#### 3. 团队记忆回退验证
- 验证 `TEAMMEM` 未启用时正确回退到 auto-only
