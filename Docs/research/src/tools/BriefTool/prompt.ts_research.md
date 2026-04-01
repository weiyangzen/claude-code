# prompt.ts 深度研究文档

## 场景与职责

prompt.ts 是 BriefTool（SendUserMessage）的**配置和提示词中心**，负责定义工具的标识常量、描述文本和系统提示片段。虽然代码量小（仅 22 行），但它是模型理解和正确使用 SendUserMessage 工具的关键。

该文件的核心职责：
1. **工具命名**：定义工具的正式名称和别名
2. **工具描述**：提供简洁的工具用途说明
3. **系统提示**：指导模型何时以及如何使用 SendUserMessage
4. **Proactive 消息指南**：详细说明主动式消息的最佳实践

## 功能点目的

### 1. 工具名称常量

```typescript
export const BRIEF_TOOL_NAME = 'SendUserMessage'
export const LEGACY_BRIEF_TOOL_NAME = 'Brief'
```

**设计考量**：
- `SendUserMessage`：明确表达工具用途，降低模型理解成本
- `Brief`：向后兼容，确保旧版本或习惯使用旧名称的代码继续工作

### 2. 工具描述

```typescript
export const DESCRIPTION = 'Send a message to the user'
```

**用途**：
- 在工具列表中显示给模型
- 帮助模型识别何时需要使用此工具
- 简洁明了，直接传达核心功能

### 3. 工具使用提示（BRIEF_TOOL_PROMPT）

**内容**：
```
Send a message the user will read. Text outside this tool is visible in the 
detail view, but most won't open it — the answer lives here.

`message` supports markdown. `attachments` takes file paths (absolute or 
cwd-relative) for images, diffs, logs.

`status` labels intent: 'normal' when replying to what they just asked; 
'proactive' when you're initiating — a scheduled task finished, a blocker 
surfaced during background work, you need input on something they haven't 
asked about. Set it honestly; downstream routing uses it.
```

**关键信息点**：
1. **核心原则**：此工具中的内容才是用户真正会读的，工具外的文本仅在详情视图中可见
2. **Markdown 支持**：消息支持格式化
3. **附件功能**：可附加图片、差异文件、日志等
4. **状态语义**：
   - `normal`：回复用户刚刚提出的问题
   - `proactive`：主动发起（任务完成、阻塞通知、需要输入）
   - 诚实设置，下游路由会使用此信息

### 4. Proactive 消息指南（BRIEF_PROACTIVE_SECTION）

这是系统提示中插入的详细指导，教导模型如何与用户有效沟通：

**核心要点**：

#### 消息定位
- SendUserMessage 是回复的归宿
- 工具外的文本用户大概率不会看到
- **失败模式警示**：真实答案在普通文本中，而 SendUserMessage 只说"done!"，用户只看到"done!"

#### 回复策略
- 每次用户发言，他们实际阅读的回复都通过 SendUserMessage
- 即使是"hi"或"thanks"也要用 SendUserMessage

#### 延迟响应模式
- 能立即回答 → 直接发送答案
- 需要查找/执行 → 先确认（"On it — checking..."），然后工作，最后发送结果
- 没有确认，用户只能盯着转圈

#### 长任务处理
- 模式：确认 → 工作 → 结果
- 中间检查点：当有用的事情发生时发送（决策、意外发现、阶段边界）
- 跳过无意义的填充（"running tests..."）

#### 消息风格
- 简洁：决策、文件:行号、PR 编号
- 第二人称（"your config"），禁用第三人称

## 具体技术实现

### 代码结构

```typescript
// 命名常量
export const BRIEF_TOOL_NAME = 'SendUserMessage'
export const LEGACY_BRIEF_TOOL_NAME = 'Brief'

// 简短描述
export const DESCRIPTION = 'Send a message to the user'

// 工具提示（嵌入工具定义）
export const BRIEF_TOOL_PROMPT = `...`

// 系统提示片段（插入系统提示）
export const BRIEF_PROACTIVE_SECTION = `...`
```

### 使用方式

#### 在 BriefTool.ts 中
```typescript
import {
  BRIEF_TOOL_NAME,
  BRIEF_TOOL_PROMPT,
  DESCRIPTION,
  LEGACY_BRIEF_TOOL_NAME,
} from './prompt.js'

buildTool({
  name: BRIEF_TOOL_NAME,
  aliases: [LEGACY_BRIEF_TOOL_NAME],
  // ...
  description: async () => DESCRIPTION,
  prompt: async () => BRIEF_TOOL_PROMPT,
})
```

#### 在系统提示生成中
```typescript
// 伪代码示例
if (isBriefEnabled()) {
  systemPrompt += BRIEF_PROACTIVE_SECTION
}
```

## 关键代码路径与文件引用

### 核心文件
| 文件 | 职责 |
|------|------|
| `prompt.ts` | 定义工具名称、描述、提示词常量 |

### 引用方
| 文件 | 用途 |
|------|------|
| `BriefTool.ts` | 导入 `BRIEF_TOOL_NAME`, `LEGACY_BRIEF_TOOL_NAME`, `DESCRIPTION`, `BRIEF_TOOL_PROMPT` |
| 系统提示生成器 | 导入 `BRIEF_PROACTIVE_SECTION` 插入系统提示 |

## 依赖与外部交互

该模块是**纯常量定义文件**，无任何外部依赖：
- 无运行时依赖
- 无类型导入
- 无构建时特性标志

## 风险、边界与改进建议

### 风险点

1. **提示词漂移（Prompt Drift）**
   - 提示词内容与工具实现不同步
   - 例如：提示词说支持某种功能，但代码未实现
   - 建议：将提示词测试纳入 CI，验证与实现的一致性

2. **国际化缺失**
   - 所有文本均为英文
   - 非英语模型可能理解偏差
   - 建议：考虑多语言提示词支持

3. **过度指导**
   - `BRIEF_PROACTIVE_SECTION` 较长，占用系统提示 token
   - 可能影响模型对其他指令的关注
   - 建议：A/B 测试简化版本的效果

4. **状态语义混淆**
   - `normal` vs `proactive` 的区分可能不够直观
   - 模型可能错误标记消息状态
   - 建议：添加更多示例或简化状态模型

### 边界情况

1. **提示词截断**
   - 系统提示长度限制可能截断 `BRIEF_PROACTIVE_SECTION`
   - 截断后的提示可能语义不完整
   - 建议：添加提示词长度监控

2. **模型版本差异**
   - 不同模型版本对提示词的理解可能不同
   - 某些模型可能忽略长提示中的部分指令
   - 建议：针对不同模型优化提示词

### 改进建议

1. **提示词工程**
   - 添加具体的使用示例（正例和反例）
   - 使用更结构化的格式（如 Markdown 列表）
   - 考虑添加"常见错误"部分

2. **动态提示词**
   - 根据上下文动态调整提示词（如项目类型、用户偏好）
   - 基于用户反馈优化提示词

3. **可配置性**
   - 允许高级用户自定义消息风格指南
   - 支持不同场景的消息模板

4. **度量和优化**
   - 跟踪模型对 SendUserMessage 的使用频率和方式
   - 分析 `status` 字段的分布
   - 基于数据优化提示词

5. **文档化**
   - 添加提示词版本历史
   - 记录提示词变更的动机和影响
