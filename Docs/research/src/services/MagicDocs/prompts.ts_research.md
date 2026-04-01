# MagicDocs Prompts 研究文档

## 场景与职责

MagicDocs Prompts 模块负责构建用于指导 Magic Docs 自动更新过程的 prompt 模板。它定义了如何生成发送给 AI agent 的指令，使其能够智能地更新 Markdown 文档。

**核心职责：**
1. 提供默认的文档更新 prompt 模板
2. 支持从用户自定义文件加载 prompt 模板
3. 实现模板变量替换机制
4. 构建完整的更新 prompt，整合文档内容、路径、标题和自定义指令

**使用场景：**
- Magic Docs 服务需要生成更新指令时调用
- 用户希望通过自定义 prompt 调整文档更新行为

---

## 功能点目的

### 1. 默认 Prompt 模板

**功能：** 提供内置的文档更新指令模板

**模板内容特点：**
- 明确告知 AI 这是内部指令，不应出现在最终文档中
- 指导 AI 基于对话内容识别新的知识和洞察
- 定义文档更新的核心原则（当前状态导向、简洁性等）
- 明确什么应该记录，什么不应该记录

**代码位置：** `getUpdatePromptTemplate()` 函数（line 8-59）

### 2. 自定义 Prompt 加载

**功能：** 允许用户通过文件自定义更新 prompt

**自定义文件位置：** `~/.claude/magic-docs/prompt.md`

**加载逻辑：**
- 如果文件存在且可读，使用其内容
- 如果不存在或读取失败，静默回退到默认模板

**代码位置：** `loadMagicDocsPrompt()` 函数（line 66-76）

### 3. 变量替换机制

**功能：** 将模板中的占位符替换为实际值

**支持的变量：**
- `{{docContents}}` - 文档当前内容
- `{{docPath}}` - 文档文件路径
- `{{docTitle}}` - Magic Doc 标题
- `{{customInstructions}}` - 文档特定的自定义指令

**替换策略：**
- 单次替换（避免二次替换问题）
- 保留未匹配变量的原始格式
- 使用 `Object.prototype.hasOwnProperty.call` 安全检查

**代码位置：** `substituteVariables()` 函数（line 81-93）

### 4. Prompt 构建

**功能：** 整合所有组件生成最终的更新 prompt

**构建流程：**
1. 加载 prompt 模板（自定义或默认）
2. 构建自定义指令部分（如果存在）
3. 执行变量替换
4. 返回完整的 prompt 字符串

**代码位置：** `buildMagicDocsUpdatePrompt()` 函数（line 98-127）

---

## 具体技术实现

### 关键流程

#### Prompt 加载流程

```typescript
async function loadMagicDocsPrompt(): Promise<string> {
  const fs = getFsImplementation()
  const promptPath = join(getClaudeConfigHomeDir(), 'magic-docs', 'prompt.md')
  
  try {
    return await fs.readFile(promptPath, { encoding: 'utf-8' })
  } catch {
    // 静默回退到默认模板
    return getUpdatePromptTemplate()
  }
}
```

**特点：**
- 使用 `getFsImplementation()` 获取文件系统实现（支持虚拟化）
- 使用 `getClaudeConfigHomeDir()` 获取配置目录路径
- 失败时静默回退，不打扰用户

#### 变量替换流程

```typescript
function substituteVariables(
  template: string,
  variables: Record<string, string>,
): string {
  // 单次替换避免两个问题：
  // 1. $ 后引用损坏（替换器函数将 $ 视为字面量）
  // 2. 当用户内容恰好包含 {{varName}} 匹配后续变量时的双重替换
  return template.replace(/\{\{(\w+)\}\}/g, (match, key: string) =>
    Object.prototype.hasOwnProperty.call(variables, key)
      ? variables[key]!
      : match,
  )
}
```

**正则表达式分析：**
- `/\{\{(\w+)\}\}/g` - 匹配 `{{variableName}}` 格式
- `\w+` - 匹配变量名（字母、数字、下划线）
- `g` 标志 - 全局替换

#### Prompt 构建流程

```typescript
export async function buildMagicDocsUpdatePrompt(
  docContents: string,
  docPath: string,
  docTitle: string,
  instructions?: string,
): Promise<string> {
  // 1. 加载模板
  const promptTemplate = await loadMagicDocsPrompt()
  
  // 2. 构建自定义指令部分
  const customInstructions = instructions
    ? `\n\nDOCUMENT-SPECIFIC UPDATE INSTRUCTIONS:\n..."${instructions}"...`
    : ''
  
  // 3. 准备变量映射
  const variables = {
    docContents,
    docPath,
    docTitle,
    customInstructions,
  }
  
  // 4. 执行替换
  return substituteVariables(promptTemplate, variables)
}
```

### 数据结构

#### 变量映射类型
```typescript
// 隐式类型，实际使用 Record<string, string>
{
  docContents: string      // 文档完整内容
  docPath: string          // 文件绝对路径
  docTitle: string         // Magic Doc 标题（从头部提取）
  customInstructions: string  // 可选的自定义指令
}
```

### 默认 Prompt 模板结构

```
1. 重要声明
   - 说明这是内部指令，不应出现在文档中
   
2. 任务描述
   - 基于对话更新文档
   - 排除指令消息本身
   
3. 文档内容占位
   - <current_doc_content>{{docContents}}</current_doc_content>
   - 文档标题和自定义指令
   
4. 核心任务
   - 仅在有实质性新信息时更新
   - 使用 Edit 工具
   - 支持并行编辑
   
5. 关键规则
   - 保留 Magic Doc 头部
   - 保留斜体指令行
   - 保持文档当前状态（非历史记录）
   - 就地更新，不附加历史注释
   - 删除过时信息
   - 修复明显错误
   - 保持文档组织良好
   
6. 文档哲学
   - 简洁，高信噪比
   - 用于概览、架构、入口点
   - 不重复源代码中明显的信息
   - 不记录每个函数、参数
   - 关注：为什么存在、如何连接、从哪里开始、使用什么模式
   
7. 记录内容指南
   - 应该记录：架构、非明显模式、入口点、设计决策、关键依赖、相关文件引用
   - 不应记录：代码中明显的信息、详尽列表、逐步实现细节、底层机制、CLAUDE.md 中已有的信息
   
8. 工具使用提示
   - 使用 Edit 工具，指定 file_path: {{docPath}}
   - 提醒保留头部不变
```

---

## 关键代码路径与文件引用

### 核心文件

| 文件路径 | 职责 |
|---------|------|
| `src/services/MagicDocs/prompts.ts` | Prompt 构建实现 |

### 依赖文件

| 文件路径 | 用途 |
|---------|------|
| `src/utils/envUtils.ts` | `getClaudeConfigHomeDir()` - 获取配置目录 |
| `src/utils/fsOperations.ts` | `getFsImplementation()` - 获取文件系统实现 |

### 调用方文件

| 文件路径 | 调用方式 |
|---------|---------|
| `src/services/MagicDocs/magicDocs.ts:164-169` | `buildMagicDocsUpdatePrompt()` - 构建更新 prompt |

### 关键代码行号

```
prompts.ts
├── 1-3:      导入依赖（path, envUtils, fsOperations）
├── 8-59:     getUpdatePromptTemplate 函数 - 默认模板
├── 66-76:    loadMagicDocsPrompt 函数 - 加载自定义模板
├── 81-93:    substituteVariables 函数 - 变量替换
└── 98-127:   buildMagicDocsUpdatePrompt 函数 - 构建最终 prompt
```

---

## 依赖与外部交互

### 外部依赖

1. **Node.js path 模块**
   - `join` - 拼接文件路径

2. **envUtils 模块**
   - `getClaudeConfigHomeDir()` - 获取 Claude 配置主目录（通常是 `~/.claude`）

3. **fsOperations 模块**
   - `getFsImplementation()` - 获取文件系统实现（支持真实文件系统或虚拟化）

### 文件系统交互

**读取路径：** `~/.claude/magic-docs/prompt.md`

**读取方式：**
```typescript
const fs = getFsImplementation()
const promptPath = join(getClaudeConfigHomeDir(), 'magic-docs', 'prompt.md')
const content = await fs.readFile(promptPath, { encoding: 'utf-8' })
```

**错误处理：** 静默回退到默认模板

### 调用关系

```
magicDocs.ts
    │
    ▼
buildMagicDocsUpdatePrompt()
    │
    ├──► loadMagicDocsPrompt()
    │       ├──► getFsImplementation()
    │       ├──► getClaudeConfigHomeDir()
    │       └──► getUpdatePromptTemplate() [fallback]
    │
    └──► substituteVariables()
```

---

## 风险、边界与改进建议

### 潜在风险

1. **模板注入风险**
   - 用户自定义 prompt 文件中的变量名可能与系统变量冲突
   - 如果用户模板包含 `{{docContents}}` 等占位符，会被替换
   - 风险较低，因为仅影响用户自己的文档更新

2. **文件读取失败风险**
   - 自定义 prompt 文件可能存在权限问题
   - 当前处理：静默回退到默认模板（合理）

3. **变量未定义风险**
   - 如果模板使用了未定义的变量，会保留原始格式
   - 可能导致 AI 困惑，但不会造成系统错误

4. **编码问题**
   - 假设 prompt 文件使用 UTF-8 编码
   - 如果文件使用其他编码，可能导致乱码

### 边界情况

1. **自定义 prompt 文件不存在**
   - 行为：使用默认模板
   - 处理：静默回退

2. **自定义 prompt 文件为空**
   - 行为：使用空模板，可能导致 AI 无法正确理解任务
   - 建议：添加空文件检查

3. **自定义 prompt 缺少必要变量占位符**
   - 行为：变量不会被替换，保留原始格式
   - 结果：AI 看到 `{{docContents}}` 而不是实际内容

4. **文档内容包含 `{{...}}` 格式文本**
   - 风险：可能被误认为是变量占位符
   - 当前处理：由于变量映射中没有这些键，会保留原样
   - 建议：使用更独特的变量分隔符，如 `{{MAGIC_DOC:varName}}`

5. **自定义指令包含特殊字符**
   - 风险：如果指令包含 `"`，可能破坏 prompt 结构
   - 当前处理：直接嵌入，未进行转义
   - 建议：考虑对指令内容进行适当的转义处理

### 改进建议

1. **模板验证**
   - 加载自定义模板后验证是否包含必要的变量占位符
   - 如果缺少关键占位符，发出警告或回退到默认模板

2. **编码检测**
   - 添加文件编码检测，支持多种编码格式
   - 或使用更明确的编码声明机制

3. **变量命名空间**
   - 使用带命名空间的变量名，如 `{{magicdoc:contents}}`
   - 减少与用户内容冲突的可能性

4. **模板缓存**
   - 自定义模板文件通常不会频繁变更
   - 可以添加简单的内存缓存，避免重复读取文件

5. **调试支持**
   - 添加环境变量控制，允许打印生成的 prompt 用于调试
   - 例如：`CLAUDE_DEBUG_MAGIC_DOCS=1`

6. **模板热重载**
   - 当前在进程生命周期内模板是固定的
   - 可以添加文件监听，支持运行时更新模板

7. **多模板支持**
   - 支持按文档类型或项目使用不同的模板
   - 例如：`~/.claude/magic-docs/prompts/architecture.md`

8. **指令长度限制**
   - 自定义指令可能非常长，影响 prompt 大小
   - 建议添加长度检查或截断机制

9. **错误报告改进**
   - 当前自定义模板加载失败是静默的
   - 建议：在调试模式下记录失败原因

10. **文档化**
    - 提供自定义 prompt 的详细文档
    - 包括可用变量列表、最佳实践、示例模板
