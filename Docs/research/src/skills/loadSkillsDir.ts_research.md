# loadSkillsDir.ts 研究文档

## 场景与职责

`loadSkillsDir.ts` 是 Claude Code CLI 中**技能加载系统的核心引擎**，负责从多个来源加载、解析、去重和管理技能。它是连接用户自定义技能（文件系统）与 CLI 命令系统的桥梁。

### 核心职责

1. **多源技能加载**：从以下位置加载技能：
   - 托管设置目录（managed settings）
   - 用户设置目录（`~/.claude/skills/`）
   - 项目设置目录（`./.claude/skills/`）
   - 额外指定目录（`--add-dir`）
   - 遗留命令目录（`./.claude/commands/`）

2. **技能解析与转换**：
   - 解析 Markdown 前 matter（frontmatter）
   - 提取技能元数据（名称、描述、工具权限等）
   - 将文件内容转换为可执行的 `Command` 对象

3. **动态技能发现**：
   - 根据文件操作路径动态发现嵌套技能目录
   - 实现条件技能（paths frontmatter）的懒加载

4. **MCP 技能构建器注册**：
   - 向 `mcpSkillBuilders.ts` 注册技能构建函数
   - 支持 MCP 服务器提供的技能

### 在系统中的位置

```
src/
├── skills/
│   ├── loadSkillsDir.ts      # 本文件：技能加载核心
│   ├── bundledSkills.ts      # 内置技能注册
│   ├── mcpSkillBuilders.ts   # MCP 技能构建器注册表
│   └── bundled/              # 内置技能实现
├── commands.ts               # 命令总线，调用 getSkillDirCommands
├── types/
│   └── command.ts            # Command 类型定义
└── utils/
    ├── markdownConfigLoader.ts  # Markdown 文件加载
    ├── frontmatterParser.ts     # Frontmatter 解析
    └── settings/                # 设置系统
```

## 功能点目的

### 1. 技能来源类型 (LoadedFrom)

```typescript
type LoadedFrom = 
  | 'commands_DEPRECATED'  // 遗留 /commands/ 目录
  | 'skills'               // /skills/ 目录
  | 'plugin'               // 插件提供的技能
  | 'managed'              // 托管设置
  | 'bundled'              // 内置技能（由 bundledSkills.ts 处理）
  | 'mcp'                  // MCP 服务器提供的技能
```

### 2. 技能目录结构

**标准格式（/skills/）**：
```
.claude/skills/
├── skill-name-1/
│   └── SKILL.md           # 技能定义文件
├── skill-name-2/
│   └── SKILL.md
```

**遗留格式（/commands/）**：
```
.claude/commands/
├── command-name.md        # 单文件命令
└── skill-name/
    └── SKILL.md           # 目录格式技能
```

### 3. Frontmatter 解析

支持的前 matter 字段：

| 字段 | 说明 |
|------|------|
| `name` | 显示名称 |
| `description` | 技能描述 |
| `allowed-tools` | 允许的工具列表 |
| `argument-hint` | 参数提示 |
| `arguments` | 参数名称列表 |
| `when_to_use` | 使用场景 |
| `version` | 版本号 |
| `model` | 指定模型 |
| `disable-model-invocation` | 禁用模型调用 |
| `user-invocable` | 用户是否可直接调用 |
| `hooks` | Hook 配置 |
| `context` | 执行上下文（inline/fork） |
| `agent` | Agent 类型 |
| `effort` | 努力级别 |
| `shell` | Shell 配置 |
| `paths` | 条件技能路径模式 |

### 4. 条件技能（Conditional Skills）

**目的**：技能仅在用户操作特定文件时才激活，减少上下文噪音。

**实现机制**：
- 使用 `ignore` 库进行 gitignore 风格的路径匹配
- 存储在 `conditionalSkills` Map 中等待激活
- 当 `activateConditionalSkillsForPaths()` 被调用时检查匹配

### 5. 动态技能发现

**目的**：支持单一代码库中不同子项目拥有各自的技能。

**发现逻辑**：
1. 从文件路径向上遍历到 CWD
2. 检查每层目录的 `.claude/skills/` 是否存在
3. 检查目录是否在 `.gitignore` 中（防止加载 node_modules 中的技能）
4. 按深度排序（最深的优先）

## 具体技术实现

### 关键流程

#### 技能加载主流程 (getSkillDirCommands)

```typescript
async function getSkillDirCommands(cwd: string): Promise<Command[]> {
  // 1. 确定加载来源
  const userSkillsDir = join(getClaudeConfigHomeDir(), 'skills')
  const managedSkillsDir = join(getManagedFilePath(), '.claude', 'skills')
  const projectSkillsDirs = getProjectDirsUpToHome('skills', cwd)
  
  // 2. 并行加载所有来源
  const [managedSkills, userSkills, projectSkills, additionalSkills, legacyCommands] = 
    await Promise.all([...])
  
  // 3. 合并并去重（基于 realpath）
  const allSkills = [...managedSkills, ...userSkills, ...projectSkills, ...]
  const deduplicated = deduplicateByRealpath(allSkills)
  
  // 4. 分离条件技能
  const unconditionalSkills = []
  const conditionalSkills = []
  for (const skill of deduplicated) {
    if (skill.paths && !activatedConditionalSkillNames.has(skill.name)) {
      conditionalSkills.push(skill)
    } else {
      unconditionalSkills.push(skill)
    }
  }
  
  // 5. 存储条件技能供后续激活
  for (const skill of conditionalSkills) {
    conditionalSkills.set(skill.name, skill)
  }
  
  return unconditionalSkills
}
```

#### 单技能目录加载 (loadSkillsFromSkillsDir)

```typescript
async function loadSkillsFromSkillsDir(basePath: string, source: SettingSource): Promise<SkillWithPath[]> {
  // 1. 读取目录
  const entries = await fs.readdir(basePath)
  
  // 2. 遍历每个子目录
  for (const entry of entries) {
    // 只接受目录格式：skill-name/SKILL.md
    if (!entry.isDirectory() && !entry.isSymbolicLink()) continue
    
    const skillFilePath = join(skillDirPath, 'SKILL.md')
    
    // 3. 读取并解析 SKILL.md
    const content = await fs.readFile(skillFilePath, 'utf-8')
    const { frontmatter, content: markdownContent } = parseFrontmatter(content, skillFilePath)
    
    // 4. 解析前 matter 字段
    const parsed = parseSkillFrontmatterFields(frontmatter, markdownContent, skillName)
    const paths = parseSkillPaths(frontmatter)
    
    // 5. 创建 Command 对象
    return {
      skill: createSkillCommand({ ...parsed, skillName, markdownContent, source, baseDir, paths }),
      filePath: skillFilePath
    }
  }
}
```

#### 技能命令创建 (createSkillCommand)

```typescript
function createSkillCommand({ ... }): Command {
  return {
    type: 'prompt',
    name: skillName,
    // ... 元数据字段
    
    async getPromptForCommand(args, toolUseContext) {
      // 1. 添加基础目录前缀
      let finalContent = baseDir 
        ? `Base directory for this skill: ${baseDir}\n\n${markdownContent}`
        : markdownContent
      
      // 2. 参数替换
      finalContent = substituteArguments(finalContent, args, true, argumentNames)
      
      // 3. 变量替换
      finalContent = finalContent.replace(/\${CLAUDE_SKILL_DIR}/g, skillDir)
      finalContent = finalContent.replace(/\${CLAUDE_SESSION_ID}/g, getSessionId())
      
      // 4. 执行内联 shell 命令（!`...`）
      if (loadedFrom !== 'mcp') {
        finalContent = await executeShellCommandsInPrompt(finalContent, ...)
      }
      
      return [{ type: 'text', text: finalContent }]
    }
  }
}
```

#### 条件技能激活

```typescript
export function activateConditionalSkillsForPaths(filePaths: string[], cwd: string): string[] {
  for (const [name, skill] of conditionalSkills) {
    const skillIgnore = ignore().add(skill.paths)
    
    for (const filePath of filePaths) {
      const relativePath = relative(cwd, filePath)
      
      // 安全检查：跳过空路径、遍历路径、绝对路径
      if (!relativePath || relativePath.startsWith('..') || isAbsolute(relativePath)) continue
      
      // 匹配成功，激活技能
      if (skillIgnore.ignores(relativePath)) {
        dynamicSkills.set(name, skill)
        conditionalSkills.delete(name)
        activatedConditionalSkillNames.add(name)
        activated.push(name)
        break
      }
    }
  }
  
  return activated
}
```

### 数据结构

```typescript
// 带路径的技能（用于去重）
type SkillWithPath = {
  skill: Command
  filePath: string
}

// 动态技能状态
const dynamicSkillDirs = new Set<string>()      // 已检查的目录
const dynamicSkills = new Map<string, Command>() // 已加载的动态技能
const conditionalSkills = new Map<string, Command>() // 待激活的条件技能
const activatedConditionalSkillNames = new Set<string>() // 已激活的技能名

// 技能加载信号
const skillsLoaded = createSignal()
```

### 去重机制

使用 `realpath` 解析符号链接，检测通过不同路径访问的同一文件：

```typescript
async function getFileIdentity(filePath: string): Promise<string | null> {
  try {
    return await realpath(filePath)
  } catch {
    return null
  }
}

// 加载时并行计算所有文件的 identity
const fileIds = await Promise.all(
  allSkillsWithPaths.map(({ filePath }) => getFileIdentity(filePath))
)

// 同步去重（保持首次加载优先）
const seenFileIds = new Map<string, SettingSource>()
for (let i = 0; i < allSkillsWithPaths.length; i++) {
  const fileId = fileIds[i]
  if (seenFileIds.has(fileId)) {
    // 跳过重复
    continue
  }
  seenFileIds.set(fileId, skill.source)
  deduplicatedSkills.push(skill)
}
```

## 关键代码路径与文件引用

### 核心导出

| 导出 | 类型 | 用途 |
|------|------|------|
| `getSkillDirCommands` | function | 主入口：获取所有技能目录命令 |
| `createSkillCommand` | function | 从解析数据创建 Command 对象 |
| `parseSkillFrontmatterFields` | function | 解析技能前 matter 字段 |
| `activateConditionalSkillsForPaths` | function | 激活匹配路径的条件技能 |
| `discoverSkillDirsForPaths` | function | 动态发现技能目录 |
| `addSkillDirectories` | function | 添加动态技能目录 |
| `getDynamicSkills` | function | 获取动态发现的技能 |
| `onDynamicSkillsLoaded` | function | 订阅技能加载事件 |
| `clearSkillCaches` | function | 清理缓存（测试用） |
| `LoadedFrom` | type | 技能来源类型 |

### 调用方

1. **`src/commands.ts`**
   - `getSkills()` 调用 `getSkillDirCommands(cwd)`
   - `getSkillToolCommands()` 和 `getSlashCommandToolSkills()` 过滤技能

2. **`src/tools/SkillTool/SkillTool.ts`**
   - 执行技能时获取技能定义

3. **文件操作工具**
   - 调用 `activateConditionalSkillsForPaths()` 在文件操作时激活条件技能
   - 调用 `discoverSkillDirsForPaths()` 发现新的技能目录

### 被调用方（依赖）

| 依赖 | 路径 | 用途 |
|------|------|------|
| `ignore` | npm | gitignore 风格路径匹配 |
| `parseFrontmatter` | `../utils/frontmatterParser.js` | 解析 Markdown 前 matter |
| `loadMarkdownFilesForSubdir` | `../utils/markdownConfigLoader.js` | 加载 Markdown 文件 |
| `executeShellCommandsInPrompt` | `../utils/promptShellExecution.js` | 执行内联 shell 命令 |
| `substituteArguments` | `../utils/argumentSubstitution.js` | 参数替换 |
| `isPathGitignored` | `../utils/git/gitignore.js` | 检查 gitignore |
| `registerMCPSkillBuilders` | `./mcpSkillBuilders.js` | 注册 MCP 构建器 |

## 依赖与外部交互

### 文件系统交互

- **读取**：
  - `~/.claude/skills/*/SKILL.md`
  - `./.claude/skills/*/SKILL.md`
  - `{managed}/.claude/skills/*/SKILL.md`
  - `./.claude/commands/**/*.md`

- **路径解析**：
  - 使用 `realpath` 解析符号链接进行去重

### 与设置系统的交互

- 检查 `isSettingSourceEnabled('projectSettings')` 决定是否加载项目技能
- 检查 `isRestrictedToPluginOnly('skills')` 决定是否锁定到仅插件技能
- `--bare` 模式跳过自动发现，仅加载 `--add-dir` 指定的目录

### 与 Git 的交互

- 使用 `isPathGitignored()` 检查技能目录是否在 `.gitignore` 中
- 防止加载 `node_modules` 等被忽略目录中的技能

### 与 MCP 的交互

- 模块初始化时调用 `registerMCPSkillBuilders()` 注册构建函数
- 使 MCP 模块能够使用相同的技能创建逻辑

## 风险、边界与改进建议

### 安全风险与防护

| 风险 | 防护措施 |
|------|----------|
| 加载恶意技能文件 | 技能在沙箱中执行，权限由系统控制 |
| 路径遍历 | `resolveSkillFilePath` 验证 |
| 符号链接攻击 | `realpath` 解析后去重 |
| gitignored 目录中的技能 | `isPathGitignored` 检查跳过 |
| MCP 技能执行 shell | `loadedFrom !== 'mcp'` 检查禁用 shell 执行 |

### 边界情况

1. **并发加载**：使用 `memoize` 确保多次调用返回相同 Promise
2. **重复技能**：基于 realpath 的去重，首次加载优先
3. **损坏的 SKILL.md**：捕获错误，记录日志，跳过该技能
4. **空技能目录**：返回空数组，不影响其他技能
5. **循环符号链接**：`realpath` 自动处理

### 性能考虑

1. **Memoization**：`getSkillDirCommands` 使用 lodash memoize，按 cwd 缓存
2. **并行加载**：所有来源并行加载
3. **懒加载**：条件技能只在需要时激活
4. **增量发现**：动态技能目录只检查一次（`dynamicSkillDirs` Set）

### 已知限制

1. **缓存失效**：技能文件修改后需要重启 CLI 或手动清除缓存
2. **条件技能延迟**：首次文件操作时才会激活，可能错过第一次请求
3. **路径匹配限制**：使用 `ignore` 库，仅支持 gitignore 风格模式

### 改进建议

1. **文件监听**：添加文件系统监听，自动重新加载修改的技能
2. **热重载 API**：提供 `/reload-skills` 命令手动刷新
3. **条件技能预加载**：分析对话历史，预加载可能相关的条件技能
4. **性能监控**：添加技能加载耗时监控，识别慢来源
5. **技能依赖**：支持技能之间的依赖声明和自动加载
6. **版本兼容性**：技能声明兼容的 CLI 版本，避免不兼容技能加载

### 测试要点

- 多来源技能去重
- 符号链接处理
- 条件技能激活逻辑
- 动态技能发现
- 错误处理（损坏文件、权限问题）
- Bare 模式行为
- Plugin-only 锁定
