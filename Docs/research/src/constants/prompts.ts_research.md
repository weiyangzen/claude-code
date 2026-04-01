# prompts.ts 深度研究文档

## 场景与职责

`prompts.ts` 是 Claude Code CLI 中最重要的系统提示词（System Prompt）生成模块。它负责构造发送给 AI 模型的系统提示词，定义了 Claude 的行为准则、工具使用规范、输出风格等核心指令。该文件是整个 AI 行为控制的中枢。

### 核心使用场景
1. **系统提示词生成**：为每次对话生成完整的系统提示词
2. **Agent 子代理提示词**：为子代理任务生成专用提示词
3. **环境信息注入**：将工作目录、Git 状态、模型信息等注入提示词
4. **功能开关集成**：根据 Beta 功能、插件等动态调整提示词
5. **缓存优化**：通过静态/动态分离优化 prompt 缓存命中率

---

## 功能点目的

### 1. 系统提示词结构

系统提示词由多个章节组成，按顺序排列：

#### 静态内容（可缓存，cacheScope: 'global'）
1. **简介** (`getSimpleIntroSection`): 基础身份说明 + 网络安全指令
2. **系统** (`getSimpleSystemSection`): 工具执行、输出格式、自动压缩说明
3. **任务执行** (`getSimpleDoingTasksSection`): 编码风格、用户协助原则
4. **谨慎执行** (`getActionsSection`): 风险操作确认要求
5. **工具使用** (`getUsingYourToolsSection`): 工具选择指导
6. **语气风格** (`getSimpleToneAndStyleSection`): 输出格式规范
7. **输出效率** (`getOutputEfficiencySection`): 简洁性要求

#### 动态内容（会话特定，动态生成）
8. **会话指导** (`getSessionSpecificGuidanceSection`): 会话类型特定指导
9. **记忆** (`loadMemoryPrompt`): 会话记忆
10. **模型覆盖** (`getAntModelOverrideSection`): 内部模型覆盖
11. **环境信息** (`computeSimpleEnvInfo`): 工作目录、平台、模型信息
12. **语言** (`getLanguageSection`): 用户语言偏好
13. **输出样式** (`getOutputStyleSection`): 输出风格配置
14. **MCP 指令** (`getMcpInstructionsSection`): MCP 服务器指令
15. **暂存区** (`getScratchpadInstructions`): 暂存目录使用说明
16. **功能结果清理** (`getFunctionResultClearingSection`): 工具结果自动清理说明
17. **其他动态章节**: Token 预算、Brief 模式等

### 2. 关键常量

| 常量 | 用途 |
|------|------|
| `CLAUDE_CODE_DOCS_MAP_URL` | Claude Code 文档地图 URL |
| `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` | 静态/动态内容分隔标记 |
| `FRONTIER_MODEL_NAME` | 最新前沿模型名称（当前：Claude Opus 4.6） |
| `CLAUDE_4_5_OR_4_6_MODEL_IDS` | Claude 4.5/4.6 模型 ID 映射 |

### 3. 核心函数

#### `getSystemPrompt(tools, model, additionalWorkingDirectories?, mcpClients?)`

**功能**：生成完整的系统提示词数组

**特殊模式**：
- **简单模式** (`CLAUDE_CODE_SIMPLE`): 极简提示词，仅包含 CWD 和日期
- **主动模式** (`PROACTIVE`/`KAIROS`): 自主代理提示词

**流程**：
```
检查 CLAUDE_CODE_SIMPLE
    ↓
[启用] 返回极简提示词
[禁用] 继续完整流程
    ↓
并行获取：技能工具命令、输出样式、环境信息
    ↓
构建动态章节数组
    ↓
解析动态章节（systemPromptSection 注册）
    ↓
合并静态 + 动态内容
    ↓
返回完整提示词数组
```

#### `computeEnvInfo(modelId, additionalWorkingDirectories?)`

**功能**：生成详细环境信息块

**包含内容**：
- 工作目录
- Git 仓库状态
- 额外工作目录
- 平台信息
- Shell 信息
- OS 版本
- 模型描述（营销名称 + 模型 ID）
- 知识截止日期

**Undercover 模式**：
- 内部 ant 用户启用 undercover 时，隐藏所有模型名称/ID
- 防止未发布模型信息泄露到公开提交/PR

#### `computeSimpleEnvInfo(model, additionalWorkingDirectories?)`

**功能**：生成简化版环境信息（用于新系统提示词格式）

**与 `computeEnvInfo` 区别**：
- 更简洁的格式
- 使用 `prependBullets` 格式化
- 包含 git worktree 检测

#### `enhanceSystemPromptWithEnvDetails(...)`

**功能**：为子代理增强系统提示词

**特殊添加**：
- Agent 线程注意事项（使用绝对路径）
- 技能发现指导
- 环境信息

#### `DEFAULT_AGENT_PROMPT`

**功能**：子代理的默认提示词模板

### 4. 动态章节管理

使用 `systemPromptSection` 和 `DANGEROUS_uncachedSystemPromptSection` 注册动态章节：

```typescript
// 普通动态章节（可缓存）
systemPromptSection('section_name', () => sectionContent)

// 危险章节（不缓存，每次重新计算）
DANGEROUS_uncachedSystemPromptSection('section_name', () => sectionContent, '原因说明')
```

**缓存策略**：
- 静态内容：跨组织缓存（`cacheScope: 'global'`）
- 普通动态内容：会话内缓存
- 危险动态内容：每次重新计算

### 5. 模型特定逻辑

#### 知识截止日期 (`getKnowledgeCutoff`)

| 模型 | 知识截止日期 |
|------|-------------|
| claude-sonnet-4-6 | August 2025 |
| claude-opus-4-6 | May 2025 |
| claude-opus-4-5 | May 2025 |
| claude-haiku-4 | February 2025 |
| claude-opus-4 / claude-sonnet-4 | January 2025 |

#### 功能结果清理 (`getFunctionResultClearingSection`)

- 仅在 `CACHED_MICROCOMPACT` feature 启用时显示
- 告知模型旧工具结果会自动清理
- 建议模型记录重要信息

### 6. 功能开关集成

通过 `feature()` 条件编译集成多种功能：

| Feature | 影响 |
|---------|------|
| `CACHED_MICROCOMPACT` | 功能结果清理说明 |
| `PROACTIVE` / `KAIROS` | 自主工作提示词 |
| `KAIROS` / `KAIROS_BRIEF` | Brief 工具提示词 |
| `EXPERIMENTAL_SKILL_SEARCH` | 技能发现指导 |
| `VERIFICATION_AGENT` | 验证代理指导 |
| `TOKEN_BUDGET` | Token 预算说明 |

---

## 具体技术实现

### 数据结构

```typescript
// 系统提示词章节类型
type SystemPromptSection = {
  key: string
  content: string | null
  cacheScope?: 'global' | 'session'
}

// 主要导出
export const CLAUDE_CODE_DOCS_MAP_URL: string
export const SYSTEM_PROMPT_DYNAMIC_BOUNDARY: string
export const FRONTIER_MODEL_NAME: string
export const CLAUDE_4_5_OR_4_6_MODEL_IDS: { opus: string; sonnet: string; haiku: string }

export async function getSystemPrompt(
  tools: Tools,
  model: string,
  additionalWorkingDirectories?: string[],
  mcpClients?: MCPServerConnection[]
): Promise<string[]>

export async function computeEnvInfo(
  modelId: string,
  additionalWorkingDirectories?: string[]
): Promise<string>

export async function computeSimpleEnvInfo(
  model: string,
  additionalWorkingDirectories?: string[]
): Promise<string>

export async function enhanceSystemPromptWithEnvDetails(
  existingSystemPrompt: string[],
  model: string,
  additionalWorkingDirectories?: string[],
  enabledToolNames?: ReadonlySet<string>
): Promise<string[]>

export const DEFAULT_AGENT_PROMPT: string
```

### 关键代码路径

#### 1. 主对话流程

```
用户发送消息
    ↓
src/QueryEngine.ts
    ↓
调用 getSystemPrompt()
    ↓
生成系统提示词
    ↓
构造完整请求发送给 API
```

**关键文件引用**：
- `src/QueryEngine.ts`: 查询引擎

#### 2. Agent 子代理流程

```
创建子代理任务
    ↓
src/tools/AgentTool/AgentTool.tsx
    ↓
使用 DEFAULT_AGENT_PROMPT 或自定义提示词
    ↓
调用 enhanceSystemPromptWithEnvDetails()
    ↓
为子代理添加上下文信息
```

**关键文件引用**：
- `src/tools/AgentTool/AgentTool.tsx`: Agent 工具
- `src/tools/AgentTool/runAgent.ts`: 运行 Agent
- `src/tools/AgentTool/resumeAgent.ts`: 恢复 Agent

#### 3. 各种子代理类型

```
探索代理 / 计划代理 / 验证代理 / 通用代理
    ↓
src/tools/AgentTool/built-in/*.ts
    ↓
使用 prompts.ts 中的函数或常量
    ↓
生成特定用途的提示词
```

**关键文件引用**：
- `src/tools/AgentTool/built-in/exploreAgent.ts`: 探索代理
- `src/tools/AgentTool/built-in/planAgent.ts`: 计划代理
- `src/tools/AgentTool/built-in/verificationAgent.ts`: 验证代理
- `src/tools/AgentTool/built-in/generalPurposeAgent.ts`: 通用代理
- `src/tools/AgentTool/built-in/claudeCodeGuideAgent.ts`: 指南代理
- `src/tools/AgentTool/built-in/statuslineSetup.ts`: 状态栏设置代理

#### 4. 上下文管理

```
管理对话上下文
    ↓
src/context.ts
    ↓
使用 getSessionStartDate()（来自 common.ts）
    ↓
影响系统提示词中的日期信息
```

**关键文件引用**：
- `src/context.ts`: 上下文管理

#### 5. API 请求构造

```
构造 API 请求
    ↓
src/utils/api.ts
    ↓
处理系统提示词分割（splitSysPromptPrefix）
    ↓
应用缓存策略
```

**关键文件引用**：
- `src/utils/api.ts`: API 工具

#### 6. 紧凑摘要

```
生成紧凑摘要
    ↓
src/commands/compact/compact.ts
    ↓
使用系统提示词相关常量
    ↓
优化上下文压缩
```

**关键文件引用**：
- `src/commands/compact/compact.ts`: 紧凑命令

#### 7. 代理编辑器

```
编辑代理配置
    ↓
src/components/agents/AgentEditor.tsx
    ↓
使用 prompts.ts 中的提示词模板
    ↓
预览代理行为
```

**关键文件引用**：
- `src/components/agents/AgentEditor.tsx`: 代理编辑器
- `src/components/agents/AgentDetail.tsx`: 代理详情
- `src/components/agents/validateAgent.ts`: 代理验证
- `src/components/agents/new-agent-creation/wizard-steps/*.tsx`: 代理创建向导

---

## 依赖与外部交互

### 内部依赖

| 导入 | 用途 |
|------|------|
| `os` | 平台信息 |
| `../utils/env.js` | 环境检测 |
| `../utils/git.js` | Git 状态 |
| `../utils/cwd.js` | 当前工作目录 |
| `../bootstrap/state.js` | 会话状态 |
| `../utils/worktree.js` | Git worktree 检测 |
| `./common.js` | 会话日期 |
| `../utils/settings/settings.js` | 用户设置 |
| `../tools/AgentTool/constants.js` | Agent 工具常量 |
| `../tools/FileWriteTool/prompt.js` | 文件写入工具名 |
| `../tools/FileReadTool/prompt.js` | 文件读取工具名 |
| `../tools/FileEditTool/constants.js` | 文件编辑工具名 |
| `../tools/TodoWriteTool/constants.js` | Todo 工具名 |
| `../tools/TaskCreateTool/constants.js` | 任务工具名 |
| `../Tool.js` | 工具类型 |
| `../types/command.js` | 命令类型 |
| `../tools/BashTool/toolName.js` | Bash 工具名 |
| `../utils/model/model.js` | 模型信息 |
| `../commands.js` | 技能工具命令 |
| `../tools/SkillTool/constants.js` | 技能工具名 |
| `./outputStyles.js` | 输出样式 |
| `../services/mcp/types.js` | MCP 类型 |
| `../tools/GlobTool/prompt.js` | Glob 工具名 |
| `../tools/GrepTool/prompt.js` | Grep 工具名 |
| `../utils/embeddedTools.js` | 嵌入式搜索工具检测 |
| `../tools/AskUserQuestionTool/prompt.js` | 提问工具名 |
| `../tools/AgentTool/built-in/exploreAgent.js` | 探索代理常量 |
| `../tools/AgentTool/builtInAgents.js` | 内置代理 |
| `../utils/permissions/filesystem.js` | 暂存区权限 |
| `../utils/envUtils.js` | 环境变量工具 |
| `../tools/REPLTool/constants.js` | REPL 模式检测 |
| `bun:bundle` | Feature flags |
| `../services/analytics/growthbook.js` | Feature 值获取 |
| `../utils/betas.js` | Beta 功能检测 |
| `../tools/AgentTool/forkSubagent.js` | Fork 子代理检测 |
| `./systemPromptSections.js` | 系统提示词章节管理 |
| `../tools/SleepTool/prompt.js` | 睡眠工具名 |
| `./xml.js` | XML 标签 |
| `../utils/debug.js` | 调试日志 |
| `../memdir/memdir.js` | 记忆加载 |
| `../utils/undercover.js` | Undercover 模式检测 |
| `../utils/mcpInstructionsDelta.js` | MCP 指令增量 |
| `../services/compact/cachedMCConfig.js` | 微压缩配置（条件加载） |
| `../proactive/index.js` | 主动模式（条件加载） |
| `../tools/BriefTool/prompt.js` | Brief 工具（条件加载） |
| `../tools/BriefTool/BriefTool.js` | Brief 工具启用检测（条件加载） |
| `../tools/DiscoverSkillsTool/prompt.js` | 技能发现工具（条件加载） |
| `../services/skillSearch/featureCheck.js` | 技能搜索功能检查（条件加载） |

### 被依赖方

| 文件 | 使用的常量/函数 | 用途 |
|------|----------------|------|
| `src/tasks/LocalMainSessionTask.ts` | `getSystemPrompt` | 本地主会话 |
| `src/Tool.js` | `DEFAULT_AGENT_PROMPT` | 工具定义 |
| `src/entrypoints/cli.tsx` | `getSystemPrompt` | CLI 入口 |
| `src/main.tsx` | `getSystemPrompt` | 应用主入口 |
| `src/services/MagicDocs/magicDocs.ts` | `getSystemPrompt` | MagicDocs |
| `src/services/SessionMemory/sessionMemory.ts` | `getSystemPrompt` | 会话记忆 |
| `src/bootstrap/state.ts` | `getSystemPrompt` | 启动状态 |
| `src/QueryEngine.ts` | `getSystemPrompt` | 查询引擎 |
| `src/context.ts` | `getSystemPrompt` | 上下文管理 |
| `src/cli/print.ts` | `getSystemPrompt` | CLI 打印 |
| `src/commands/btw/btw.tsx` | `getSystemPrompt` | BTW 命令 |
| `src/tools/AgentTool/*.ts` | `DEFAULT_AGENT_PROMPT`, `enhanceSystemPromptWithEnvDetails` | Agent 工具 |
| `src/tools/AgentTool/built-in/*.ts` | `DEFAULT_AGENT_PROMPT` | 内置代理 |
| `src/tools/BriefTool/BriefTool.ts` | `getSystemPrompt` | Brief 工具 |
| `src/utils/analyzeContext.ts` | `getSystemPrompt` | 上下文分析 |
| `src/utils/queryContext.ts` | `getSystemPrompt` | 查询上下文 |
| `src/utils/systemPrompt.ts` | `getSystemPrompt` | 系统提示词工具 |
| `src/utils/plugins/loadPluginAgents.ts` | `getSystemPrompt` | 插件代理加载 |
| `src/utils/api.ts` | `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` | API 工具 |
| `src/utils/swarm/inProcessRunner.ts` | `enhanceSystemPromptWithEnvDetails` | 进程内运行器 |
| `src/components/agents/*.tsx` | `DEFAULT_AGENT_PROMPT` | 代理组件 |
| `src/commands/compact/compact.ts` | `getSystemPrompt` | 紧凑命令 |
| `src/screens/REPL.tsx` | `getSystemPrompt` | REPL 屏幕 |

---

## 风险、边界与改进建议

### 当前风险

1. **提示词注入风险**
   - 用户输入（如语言偏好）直接注入提示词
   - MCP 服务器指令来自外部，可能包含恶意内容

2. **缓存边界复杂性**
   - `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 位置关键
   - 错误放置会导致缓存失效或信息泄露

3. **条件编译复杂性**
   - 大量使用 `feature()` 和 `process.env.USER_TYPE`
   - 代码路径复杂，测试覆盖困难

4. **模型特定逻辑分散**
   - 知识截止日期、功能支持等逻辑分散在各函数
   - 新增模型需要多处修改

5. **提示词长度**
   - 系统提示词可能很长（数千 tokens）
   - 影响上下文窗口可用空间

### 边界情况

| 场景 | 处理 |
|------|------|
| MCP 服务器无指令 | `getMcpInstructionsSection` 返回 null |
| 无输出样式 | `getOutputStyleSection` 返回 null |
| 无语言偏好 | `getLanguageSection` 返回 null |
| Undercover 模式 | 隐藏模型名称和 ID |
| 非 Git 仓库 | `isGit` 为 false |
| Git worktree | 添加特殊说明 |
| 无 Shell 环境变量 | 显示 'unknown' |
| Windows 平台 | 添加 Unix 语法提示 |

### 改进建议

1. **提示词验证**
   ```typescript
   // 建议添加提示词验证
   export function validateSystemPrompt(prompt: string[]): void {
     // 检查必须存在的章节
     // 验证无重复内容
     // 检测潜在注入
   }
   ```

2. **提示词版本控制**
   ```typescript
   // 建议添加版本信息
   export const SYSTEM_PROMPT_VERSION = '2024.04.01'
   
   export async function getSystemPrompt(...): Promise<{
     version: string
     sections: string[]
   }>
   ```

3. **A/B 测试支持**
   ```typescript
   // 建议支持提示词变体
   export async function getSystemPrompt(
     ...,
     variant?: 'control' | 'treatment_a' | 'treatment_b'
   ): Promise<string[]>
   ```

4. **提示词分析工具**
   ```typescript
   // 建议添加分析函数
   export function analyzeSystemPrompt(prompt: string[]): {
     totalLength: number
     estimatedTokens: number
     cacheableRatio: number
     sectionBreakdown: Record<string, number>
   }
   ```

5. **模块化重构**
   ```typescript
   // 建议将大文件拆分为模块
   src/constants/prompts/
   ├── index.ts           # 主入口
   ├── sections/
   │   ├── intro.ts       # 简介章节
   │   ├── system.ts      # 系统章节
   │   ├── tasks.ts       # 任务章节
   │   └── ...
   ├── env.ts             # 环境信息
   └── agent.ts           # Agent 提示词
   ```

6. **自动化测试**
   - 提示词结构验证
   - 各功能开关组合测试
   - Token 估算准确性测试
   - 缓存边界测试

### 与 AI 模型的关系

```
prompts.ts (提示词生成)
    ↓ 构造
系统提示词 (System Prompt)
    ↓ 发送
Anthropic API
    ↓ 处理
Claude AI 模型
    ↓ 生成
响应内容
```

系统提示词是控制 AI 行为的主要机制，prompts.ts 是整个系统的核心控制点。任何修改都会直接影响 AI 的行为和输出质量，需要极其谨慎的测试和评估。
