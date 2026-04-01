# command.ts 研究文档

## 场景与职责

`src/types/command.ts` 是 Claude Code CLI 的命令系统核心类型定义文件。它定义了所有斜杠命令（slash commands）的类型结构，包括：

1. **PromptCommand**: 基于提示模板的命令（skills），由模型调用并展开为对话内容
2. **LocalCommand**: 本地 JavaScript 函数命令，返回文本结果
3. **LocalJSXCommand**: 本地 React/JSX 命令，渲染交互式 UI 组件

该文件是命令注册、发现、执行全流程的基础类型层，被 `src/commands.ts` 及所有具体命令实现所依赖。

## 功能点目的

### 1. 命令类型联合（Command Union Type）
定义 `Command = CommandBase & (PromptCommand | LocalCommand | LocalJSXCommand)`，实现：
- 统一的命令基础属性（name, description, isEnabled, availability 等）
- 差异化的命令实现机制（prompt 展开 vs 本地执行 vs JSX 渲染）

### 2. PromptCommand - Skill 系统基础
- 支持从多种来源加载：builtin, mcp, plugin, bundled, settings
- 支持 hooks 注册（Skills 可携带 hooks 配置）
- 支持执行上下文选择：inline（默认）或 fork（子代理执行）
- 支持 effort 级别控制（影响模型选择）
- 支持路径过滤（paths 属性控制 Skill 可见性）

### 3. LocalJSXCommand - 交互式命令
- 通过 `load()` 懒加载实现，延迟加载重型依赖
- 返回 React.ReactNode，支持复杂 UI 交互
- 通过 `onDone` 回调与 REPL 集成

### 4. 命令可用性控制（CommandAvailability）
- `claude-ai`: 仅限 claude.ai OAuth 订阅用户
- `console`: 仅限 Console API Key 用户（直接 api.anthropic.com）
- 通过 `meetsAvailabilityRequirement()` 在 `src/commands.ts:417-443` 实现运行时过滤

### 5. 命令来源追踪（loadedFrom）
区分命令加载来源，用于 UI 展示和权限控制：
- `commands_DEPRECATED`: 旧版 commands 目录
- `skills`: skills 目录
- `plugin`: 插件系统
- `managed`: 托管插件
- `bundled`: 内置 bundle
- `mcp`: MCP 服务器

## 具体技术实现

### 关键数据结构

```typescript
// 命令基础属性
interface CommandBase {
  name: string
  description: string
  availability?: CommandAvailability[]  // 权限控制
  isEnabled?: () => boolean             // 动态启用状态
  isHidden?: boolean                    // 是否在帮助中隐藏
  aliases?: string[]                    // 命令别名
  loadedFrom?: 'commands_DEPRECATED' | 'skills' | 'plugin' | ...  // 来源追踪
  immediate?: boolean                   // 是否立即执行（绕过队列）
  isSensitive?: boolean                 // 参数是否脱敏
}

// Prompt 命令（Skill）
interface PromptCommand {
  type: 'prompt'
  progressMessage: string
  contentLength: number                 // 用于 token 估算
  source: SettingSource | 'builtin' | 'mcp' | 'plugin' | 'bundled'
  context?: 'inline' | 'fork'           // 执行上下文
  agent?: string                        // fork 时的代理类型
  effort?: EffortValue
  paths?: string[]                      // 文件路径过滤
  getPromptForCommand(args: string, context: ToolUseContext): Promise<ContentBlockParam[]>
}

// Local JSX 命令（交互式）
interface LocalJSXCommand {
  type: 'local-jsx'
  load: () => Promise<{ call: LocalJSXCommandCall }>
}
```

### 关键流程

1. **命令注册流程**（`src/commands.ts:258-346`）
   - `COMMANDS()` memoized 函数返回所有内置命令
   - `loadAllCommands()` 异步加载 skills、plugins、workflows
   - `getCommands()` 应用 availability 和 isEnabled 过滤

2. **Skill 执行流程**（`src/tools/SkillTool/SkillTool.ts`）
   - 模型调用 SkillTool 并指定 command name
   - 查找匹配的 PromptCommand
   - 调用 `getPromptForCommand()` 展开为 ContentBlockParam[]
   - 展开内容插入对话上下文

3. **LocalJSX 命令执行流程**（`src/utils/processUserInput/processSlashCommand.tsx`）
   - 解析斜杠命令
   - 调用 `load()` 懒加载命令模块
   - 渲染返回的 ReactNode
   - 通过 `onDone` 回调返回结果到 REPL

### 工具函数

```typescript
// 获取命令显示名称（支持 userFacingName 覆盖）
export function getCommandName(cmd: CommandBase): string

// 检查命令是否启用（默认 true）
export function isCommandEnabled(cmd: CommandBase): boolean
```

## 关键代码路径与文件引用

### 类型定义
- `src/types/command.ts` - 本文件，核心类型定义

### 命令注册与发现
- `src/commands.ts` - 命令注册中心，实现 `getCommands()`, `meetsAvailabilityRequirement()`
- `src/skills/loadSkillsDir.ts` - 从 skills 目录加载命令
- `src/skills/bundledSkills.ts` - 内置 bundled skills
- `src/utils/plugins/loadPluginCommands.ts` - 从插件加载命令

### 命令执行
- `src/tools/SkillTool/SkillTool.ts` - PromptCommand 执行
- `src/utils/processUserInput/processSlashCommand.tsx` - 斜杠命令处理
- `src/utils/handlePromptSubmit.ts` - 提示提交处理

### 使用场景
- `src/screens/REPL.tsx` - REPL 主界面，命令交互
- `src/hooks/useTypeahead.tsx` - 命令自动补全
- `src/commands/help/help.tsx` - 帮助系统

## 依赖与外部交互

### 导入依赖
```typescript
import type { ContentBlockParam } from '@anthropic-ai/sdk/resources/index.mjs'  // Anthropic SDK
import type { UUID } from 'crypto'                                               // Node.js crypto
import type { CanUseToolFn } from '../hooks/useCanUseTool.js'                    // 工具使用检查
import type { CompactionResult } from '../services/compact/compact.js'           // 压缩结果
import type { ScopedMcpServerConfig } from '../services/mcp/types.js'            // MCP 配置
import type { ToolUseContext } from '../Tool.js'                                 // 工具上下文
import type { EffortValue } from '../utils/effort.js'                            // Effort 级别
import type { IDEExtensionInstallationStatus, IdeType } from '../utils/ide.js'   // IDE 状态
import type { SettingSource } from '../utils/settings/constants.js'              // 设置来源
import type { HooksSettings } from '../utils/settings/types.js'                  // Hooks 配置
import type { ThemeName } from '../utils/theme.js'                               // 主题
import type { LogOption } from './logs.js'                                       // 日志选项
import type { Message } from './message.js'                                      // 消息类型
import type { PluginManifest } from './plugin.js'                                // 插件清单
```

### 被依赖方（约 70+ 文件导入）
- 所有命令实现文件（`src/commands/*`）
- Skill 系统（`src/skills/*`）
- 插件系统（`src/utils/plugins/*`）
- REPL 和 UI 组件（`src/screens/REPL.tsx`, `src/components/*`）
- 工具执行（`src/tools/*`）

## 风险、边界与改进建议

### 潜在风险

1. **类型安全边界**
   - `LocalCommandResult` 的联合类型在扩展时需要同步更新所有消费者
   - `ResumeEntrypoint` 硬编码字符串值，缺乏与实现代码的静态关联

2. **命令来源复杂性**
   - `source` 和 `loadedFrom` 两个字段容易混淆
   - `source` 表示配置来源（settings, builtin 等）
   - `loadedFrom` 表示代码加载位置（skills, plugin 等）

3. **懒加载错误处理**
   - `LocalJSXCommand.load()` 返回 Promise，错误处理分散在各调用点
   - 建议增加统一的加载错误边界

### 边界情况

1. **命令名称冲突**
   - 同名命令按优先级覆盖：builtin < bundled < skills < plugins
   - 通过 `findCommand()` 的别名解析可能导致意外匹配

2. **Availability 检查时机**
   - `meetsAvailabilityRequirement()` 每次调用都重新计算
   - 频繁调用（如 typeahead）可能影响性能

3. **PromptCommand 的 contentLength**
   - 用于 token 估算但依赖开发者手动设置
   - 不准确的长度可能导致上下文窗口误判

### 改进建议

1. **类型改进**
   ```typescript
   // 建议：使用模板字面量类型增强 ResumeEntrypoint 的类型安全
   type ResumeEntrypoint = 
     | `cli_${string}`
     | `slash_command_${string}`
     | 'fork'
   ```

2. **命令加载优化**
   - 考虑为 `loadAllCommands` 增加增量更新能力
   - 当前全量刷新在插件频繁变更时效率低

3. **文档完善**
   - `getPromptForCommand` 的 `args` 参数格式缺乏规范
   - 建议增加参数解析工具函数的标准化

4. **测试覆盖**
   - 命令可用性矩阵（availability × isEnabled × loadedFrom）组合复杂
   - 建议增加集成测试覆盖各种组合场景
