# loadPluginAgents.ts 深度研究文档

## 场景与职责

`loadPluginAgents.ts` 负责从 Claude Code 插件中加载 Agent 定义。Agent 是 Claude Code 的 AI 助手角色，插件可通过 Markdown 文件（前置元数据 + 系统提示内容）声明自定义 Agent。

核心场景：
1. **插件启用时**：加载插件声明的所有 Agent
2. **Agent 命名空间**：自动添加 `pluginName:namespace:` 前缀避免冲突
3. **变量替换**：支持 `${CLAUDE_PLUGIN_ROOT}` 和 `${user_config.X}`
4. **内存集成**：自动注入文件工具以支持记忆功能

## 功能点目的

### 1. 插件 Agent 加载 (`loadPluginAgents`)
- **备忘录化**：使用 `lodash.memoize` 缓存结果
- **并行处理**：多插件并行加载，每插件内串行去重
- **错误隔离**：单 Agent 失败不影响其他

### 2. Agent 文件解析 (`loadAgentFromFile`)
- **前置元数据解析**：名称、描述、工具、模型、颜色等
- **变量替换**：
  - `${CLAUDE_PLUGIN_ROOT}` → 插件根目录
  - `${user_config.X}` → 用户配置值（敏感值替换为占位符）
- **内容组装**：系统提示 + 记忆提示（如启用）

### 3. 多路径支持
- **默认路径**：`plugin.agentsPath`（标准 `agents/` 目录）
- **自定义路径**：`plugin.agentsPaths`（manifest 声明的额外路径）
- **文件/目录混合**：支持单文件或目录批量加载

### 4. 安全限制
- **禁止字段**：`permissionMode`, `hooks`, `mcpServers` 被忽略
- **原因**：插件是第三方代码，这些字段会提升权限，超出安装时信任边界
- **替代方案**：用户可在 `.claude/agents/` 定义以获得完全控制

### 5. 记忆功能集成
- **自动工具注入**：若启用记忆，自动添加 FileWrite/FileEdit/FileRead
- **记忆提示追加**：`loadAgentMemoryPrompt(agentType, memory)`

## 具体技术实现

### 核心类型与常量
```typescript
const VALID_MEMORY_SCOPES: AgentMemoryScope[] = ['user', 'project', 'local']

// AgentDefinition 结构（返回类型）
{
  agentType: string,        // 如 "aws:utils:helper"
  whenToUse: string,        // 使用场景描述
  tools: string[],          // 允许的工具
  disallowedTools?: string[],
  skills?: string[],
  getSystemPrompt: () => string,  // 延迟加载
  source: 'plugin',
  color?: AgentColorName,
  model?: string | 'inherit',
  filename: string,
  plugin: string,
  background?: boolean,
  memory?: AgentMemoryScope,
  isolation?: 'worktree',
  effort?: number,
  maxTurns?: number
}
```

### 加载流程算法
```
loadPluginAgents() [备忘录化]
  ├─ loadAllPluginsCacheOnly() → 获取启用插件列表
  ├─ 若存在加载错误 → 记录调试日志
  └─ Promise.all(各插件并行处理)
      └─ 每插件处理
          ├─ 创建 loadedPaths Set（去重）
          ├─ 默认 agentsPath 处理
          │   └─ loadAgentsFromDirectory()
          │       └─ walkPluginMarkdown() 遍历 .md 文件
          │           └─ loadAgentFromFile() 解析每个文件
          └─ agentsPaths 处理（自定义路径）
              └─ 并行处理各路径
                  ├─ 目录 → loadAgentsFromDirectory()
                  └─ 文件 → loadAgentFromFile()
          └─ 合并所有 Agent
  └─ flat() 合并所有插件结果
```

### 单文件解析算法 (`loadAgentFromFile`)
```
loadAgentFromFile(filePath, pluginName, namespace, sourceName, pluginPath, manifest, loadedPaths)
  ├─ 去重检查：isDuplicatePath() → 若重复返回 null
  ├─ 读取文件内容
  ├─ parseFrontmatter() 解析前置元数据
  ├─ 构建 agentType: `${pluginName}:${namespace.join(':')}:${baseName}`
  ├─ 解析元数据字段
  │   ├─ whenToUse: description / when-to-use / 默认值
  │   ├─ tools: parseAgentToolsFromFrontmatter()
  │   ├─ skills: parseSlashCommandToolsFromFrontmatter()
  │   ├─ color: 直接取值
  │   ├─ model: 'inherit' 或具体模型名
  │   ├─ background: 'true' 解析为 boolean
  │   ├─ memory: 验证为 VALID_MEMORY_SCOPES 之一
  │   ├─ isolation: 'worktree' 或 undefined
  │   ├─ effort: parseEffortValue()
  │   └─ maxTurns: parsePositiveIntFromFrontmatter()
  ├─ 安全检查：忽略 permissionMode, hooks, mcpServers（记录警告）
  ├─ 系统提示处理
  │   ├─ substitutePluginVariables() 替换 ${CLAUDE_PLUGIN_ROOT}
 │   └─ substituteUserConfigInContent() 替换 ${user_config.X}
  ├─ 记忆功能集成
  │   ├─ 若启用记忆且有 memory 字段
  │   │   ├─ 注入 FileWrite/FileEdit/FileRead 工具
  │   │   └─ getSystemPrompt() 追加记忆提示
  │   └─ 否则直接返回 systemPrompt
  └─ 返回 AgentDefinition 对象
```

### 变量替换实现
```typescript
// 插件变量替换
systemPrompt = substitutePluginVariables(markdownContent.trim(), {
  path: pluginPath,
  source: sourceName
})

// 用户配置替换（若 manifest 声明 userConfig）
if (pluginManifest.userConfig) {
  systemPrompt = substituteUserConfigInContent(
    systemPrompt,
    loadPluginOptions(sourceName),
    pluginManifest.userConfig
  )
}
```

## 关键代码路径与文件引用

### 内部依赖
```
loadPluginAgents.ts
  ├─ lodash-es/memoize.js: memoize
  ├─ path: basename
  ├─ memdir/paths.ts: isAutoMemoryEnabled
  ├─ tools/AgentTool/agentColorManager.ts: AgentColorName
  ├─ tools/AgentTool/agentMemory.ts: AgentMemoryScope, loadAgentMemoryPrompt
  ├─ tools/AgentTool/loadAgentsDir.ts: AgentDefinition
  ├─ tools/FileEditTool/constants.ts: FILE_EDIT_TOOL_NAME
  ├─ tools/FileReadTool/prompt.ts: FILE_READ_TOOL_NAME
  ├─ tools/FileWriteTool/prompt.ts: FILE_WRITE_TOOL_NAME
  ├─ types/plugin.ts: getPluginErrorMessage
  ├─ debug.ts: logForDebugging
  ├─ effort.ts: EFFORT_LEVELS, parseEffortValue
  ├─ frontmatterParser.ts: coerceDescriptionToString, parseFrontmatter, parsePositiveIntFromFrontmatter
  ├─ fsOperations.ts: getFsImplementation, isDuplicatePath
  ├─ markdownConfigLoader.ts: parseAgentToolsFromFrontmatter, parseSlashCommandToolsFromFrontmatter
  ├─ pluginLoader.ts: loadAllPluginsCacheOnly
  ├─ pluginOptionsStorage.ts: loadPluginOptions, substitutePluginVariables, substituteUserConfigInContent
  ├─ schemas.ts: PluginManifest
  └─ walkPluginMarkdown.ts: walkPluginMarkdown
```

### 外部调用方
| 调用方 | 用途 |
|--------|------|
| `tools/AgentTool/loadAgentsDir.ts` | 合并插件 Agent 与用户定义 Agent |
| `cacheUtils.ts` | 清除 Agent 缓存 |
| `init.ts` | 初始化时预加载 |

### 调用链
```
Agent 工具初始化
  └─ loadAllAgents()
      ├─ loadUserAgents() → .claude/agents/
      └─ loadPluginAgents() → 本模块
          ├─ loadAllPluginsCacheOnly()
          └─ 各插件 Agent 加载
              └─ walkPluginMarkdown() → loadAgentFromFile()
```

## 依赖与外部交互

### Agent Markdown 格式
```markdown
---
name: helper
description: AWS 辅助工具
tools: ["Bash", "FileRead"]
skills: ["aws-cli"]
memory: project
effort: low
maxTurns: 10
color: blue
---

你是 AWS 助手，帮助用户管理 AWS 资源...
```

### 命名空间规则
- 基础名：文件名（去除 `.md`）或 `frontmatter.name`
- 完整名：`${pluginName}:${namespace.join(':')}:${baseName}`
- 示例：`aws:utils:helper`（aws 插件的 utils 目录下的 helper agent）

### 安全边界
| 字段 | 插件支持 | 用户 .claude/agents/ 支持 | 原因 |
|------|----------|---------------------------|------|
| tools | ✓ | ✓ | 基础功能 |
| skills | ✓ | ✓ | 基础功能 |
| memory | ✓ | ✓ | 基础功能 |
| effort | ✓ | ✓ | 基础功能 |
| maxTurns | ✓ | ✓ | 基础功能 |
| color | ✓ | ✓ | 基础功能 |
| model | ✓ | ✓ | 基础功能 |
| background | ✓ | ✓ | 基础功能 |
| isolation | ✓ | ✓ | 基础功能 |
| permissionMode | ✗ | ✓ | 权限提升风险 |
| hooks | ✗ | ✓ | 权限提升风险 |
| mcpServers | ✗ | ✓ | 权限提升风险 |

## 风险、边界与改进建议

### 已知风险

1. **变量替换安全风险**
   - 风险：`${user_config.X}` 可能泄露敏感值到提示
   - 缓解：`substituteUserConfigInContent` 将敏感值替换为占位符
   - 局限：非敏感值仍可能包含 PII

2. **工具注入冲突**
   - 风险：记忆功能自动注入的工具可能与显式声明冲突
   - 现状：使用 Set 去重，显式声明优先

3. **文件遍历性能**
   - 风险：大型插件的 agents 目录可能包含大量文件
   - 现状：`walkPluginMarkdown` 串行遍历
   - 建议：考虑并行或惰性加载

4. **缓存失效粒度**
   - 风险：`clearPluginAgentCache` 清除全部，无法单插件清除
   - 现状：插件变更通常伴随全量重载

### 边界条件

| 场景 | 行为 |
|------|------|
| 文件解析失败 | 记录错误，返回 null，不影响其他 Agent |
| 前置元数据缺失 | 使用默认值（如文件名作为 name）|
| 无效 memory 值 | 记录警告，忽略该字段 |
| 无效 effort 值 | 记录警告，忽略该字段 |
| 无效 maxTurns 值 | 记录警告，忽略该字段 |
| 禁用字段声明 | 记录警告，忽略该字段 |
| 重复文件路径 | 首次加载，后续跳过 |

### 改进建议

1. **增量加载**
   - 当前：全量加载所有插件 Agent
   - 建议：按需加载，首次使用时解析

2. **Agent 热重载**
   - 当前：缓存至会话结束
   - 建议：文件监听，开发时自动重载

3. **更好的错误报告**
   - 当前：仅调试日志
   - 建议：收集到 PluginError 供 `/doctor` 显示

4. **Agent 依赖声明**
   - 当前：无依赖机制
   - 建议：支持 Agent 依赖其他 Agent 或技能

5. **国际化支持**
   - 当前：单语言描述
   - 建议：支持多语言 `description.i18n`
