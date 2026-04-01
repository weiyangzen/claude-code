# permissionExplainer.ts 深度研究文档

## 场景与职责

`permissionExplainer.ts` 是 Claude Code 的权限解释器模块，利用 AI 模型（Haiku）为 shell 命令和工具调用生成风险解释。该模块在权限提示对话框中为用户提供上下文感知的风险评估，帮助用户理解为什么某个操作需要权限确认。

**核心职责：**
1. **风险解释生成**：使用 AI 分析工具输入，生成人类可读的操作说明
2. **风险等级评估**：将操作分类为 LOW/MEDIUM/HIGH 风险级别
3. **对话上下文整合**：提取最近的助手消息提供操作背景
4. **分析遥测**：记录解释器使用情况和性能指标

## 功能点目的

### 1. 风险等级定义

**`RiskLevel`** - 三级风险分类
- `LOW`: 安全的开发工作流（如读取配置文件、列出目录）
- `MEDIUM`: 可恢复的更改（如编辑非关键文件、安装依赖）
- `HIGH`: 危险/不可逆操作（如删除系统文件、执行任意代码）

**`RISK_LEVEL_NUMERIC`** - 分析用数值映射
- 用于遥测数据的标准化（LOW=1, MEDIUM=2, HIGH=3）

### 2. 结构化输出模式

**`EXPLAIN_COMMAND_TOOL`** - 强制结构化输出的工具定义
- 使用 Anthropic API 的 tool use 功能确保返回格式一致性
- 定义四个字段：
  - `explanation`: 操作说明（1-2 句话）
  - `reasoning`: 执行原因（以 "I" 开头）
  - `risk`: 风险描述（15 字以内）
  - `riskLevel`: 风险等级枚举

**`RiskAssessmentSchema`** - Zod 验证模式
- 使用 `lazySchema` 延迟初始化避免循环依赖
- 验证 AI 返回数据的类型安全

### 3. 对话上下文提取

**`extractConversationContext`** - 提取最近对话历史
- 获取最近 3 条助手消息
- 提取文本内容块（过滤掉其他类型）
- 限制总字符数（默认 1000）以控制提示大小
- 倒序排列保持时间线逻辑

### 4. 功能开关检查

**`isPermissionExplainerEnabled`** - 检查功能是否启用
- 默认启用（`undefined` 视为 true）
- 用户可通过配置显式禁用
- 配置路径：`getGlobalConfig().permissionExplainerEnabled`

### 5. 核心解释生成

**`generatePermissionExplanation`** - 主入口函数
- **前置检查**：功能开关、中止信号
- **输入格式化**：处理字符串或对象输入
- **上下文构建**：整合工具名称、描述、输入、对话历史
- **API 调用**：使用 `sideQuery` 调用 Haiku 模型
- **结果解析**：提取 tool use 块并验证
- **遥测记录**：成功/失败事件、延迟指标

## 具体技术实现

### 关键数据结构

```typescript
// 风险等级类型
type RiskLevel = 'LOW' | 'MEDIUM' | 'HIGH'

// 权限解释结果
type PermissionExplanation = {
  riskLevel: RiskLevel
  explanation: string      // 操作说明
  reasoning: string        // 执行原因
  risk: string            // 风险描述
}

// 生成参数
type GenerateExplanationParams = {
  toolName: string
  toolInput: unknown
  toolDescription?: string
  messages?: Message[]
  signal: AbortSignal
}
```

### 系统提示词设计

```
SYSTEM_PROMPT = "Analyze shell commands and explain what they do, 
why you're running them, and potential risks."
```

**设计要点：**
- 简洁明确，不暴露内部实现细节
- 要求以第一人称解释（"I need to..."）增强用户理解
- 强调风险识别

### 用户提示词构建

```
Tool: {toolName}
Description: {toolDescription}
Input:
{formattedInput}

Recent conversation context:
{conversationContext}

Explain this command in context.
```

### 关键流程

```
输入: toolName, toolInput, toolDescription?, messages?, signal
  ↓
1. 功能开关检查 → 禁用则返回 null
  ↓
2. 输入格式化:
   - 字符串类型直接使用
   - 其他类型 JSON 序列化
  ↓
3. 对话上下文提取（最近 3 条助手消息）
  ↓
4. 构建用户提示词
  ↓
5. 调用 sideQuery:
   - 模型: getMainLoopModel()
   - 强制 tool_choice: explain_command
   - querySource: 'permission_explainer'
  ↓
6. 结果处理:
   - 提取 tool_use 块
   - Zod 验证
   - 记录遥测
  ↓
返回: PermissionExplanation | null
```

### 错误处理策略

| 错误类型 | 处理 | 遥测 |
|---------|------|------|
| 用户中止 | 静默返回 null | 无 |
| 解析失败 | 返回 null | `tengu_permission_explainer_error` (type=1) |
| 网络错误 | 返回 null | `tengu_permission_explainer_error` (type=2) |
| 未知错误 | 返回 null | `tengu_permission_explainer_error` (type=3) |

### 性能优化

1. **异步执行**：解释生成是非阻塞的，不延迟权限提示显示
2. **超时控制**：通过 `AbortSignal` 支持取消
3. **延迟记录**：记录 API 调用耗时用于性能监控

## 关键代码路径与文件引用

### 核心依赖

| 导入路径 | 用途 |
|---------|------|
| `zod/v4` | 结构化数据验证 |
| `../../services/analytics/index.js` | 遥测事件记录 |
| `../sideQuery.js` | AI 模型调用（非主循环） |
| `../model/model.js` | 获取主循环模型配置 |
| `../lazySchema.js` | 延迟初始化 Zod 模式 |

### 被调用方

- **权限提示组件**: 在显示权限对话框时调用
- **BashTool**: 为 shell 命令生成解释
- **其他工具**: 为复杂操作提供上下文

### 调用时序

```
用户触发工具
  ↓
权限检查 → 需要确认
  ↓
并行启动:
  - 显示权限对话框
  - 调用 generatePermissionExplanation (异步)
  ↓
解释生成完成 → 更新对话框显示
```

## 依赖与外部交互

### 运行时依赖

1. **AI 模型服务**:
   - `sideQuery` - 独立的 API 调用通道，不影响主对话
   - 使用主循环模型（通常为 Haiku）以平衡成本和速度

2. **分析服务**:
   - `logEvent` - 记录使用情况和性能指标
   - `sanitizeToolNameForAnalytics` - 工具名称脱敏

3. **配置系统**:
   - `getGlobalConfig` - 获取用户功能开关设置

### 数据流

```
工具输入 → 格式化 → 提示词构建 → sideQuery → AI 模型
                                              ↓
用户界面 ← 结果解析 ← Zod 验证 ← tool_use 提取
```

## 风险、边界与改进建议

### 已知风险

1. **AI 幻觉**:
   - 模型可能生成不准确的风险评估
   - 缓解：仅作为参考信息，不用于安全决策

2. **提示注入**:
   - 恶意工具输入可能操纵解释内容
   - 缓解：工具输入经过权限系统验证后才传递给解释器

3. **延迟问题**:
   - API 调用可能延迟权限提示显示
   - 缓解：异步执行，超时控制

4. **成本累积**:
   - 每个权限提示都触发 API 调用
   - 缓解：使用轻量级模型（Haiku），可配置禁用

### 边界条件

1. **上下文长度**:
   - 对话历史限制 1000 字符
   - 超长输入可能被截断

2. **模型可用性**:
   - 模型服务不可用时 gracefully 降级
   - 返回 null 不影响核心功能

3. **工具输入类型**:
   - 支持字符串和对象类型
   - 循环引用对象可能导致 JSON 序列化失败（已捕获）

### 改进建议

1. **缓存机制**:
   - 相同命令的解释可缓存复用
   - 使用 LRU 缓存避免重复 API 调用

2. **本地化支持**:
   - 当前仅支持英文输出
   - 可根据用户设置选择解释语言

3. **解释质量反馈**:
   - 添加用户反馈机制收集解释质量
   - 用于优化提示词设计

4. **成本优化**:
   - 对简单命令使用规则-based 解释，避免 API 调用
   - 仅在复杂命令时启用 AI 解释

5. **安全增强**:
   - 添加输出内容过滤，防止生成有害建议
   - 对高风险操作添加额外警告标识

6. **性能监控**:
   - 添加成功率、延迟百分位指标
   - 监控模型响应质量趋势
