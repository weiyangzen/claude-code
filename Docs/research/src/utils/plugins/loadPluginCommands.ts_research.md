# loadPluginCommands.ts 深度研究文档

## 文件元数据
- **路径**: `src/utils/plugins/loadPluginCommands.ts`
- **大小**: 30,541 bytes
- **核心职责**: 插件命令和技能的加载、解析与注册

---

## 一、场景与职责

### 1.1 功能定位
本模块是 Claude Code 插件系统的**命令加载核心**，负责：
1. 从已启用的插件中加载 Markdown 格式的命令定义文件
2. 支持传统命令（commands/目录下的 .md 文件）和技能（skills/目录下的 SKILL.md）
3. 解析 frontmatter 元数据，构建可执行的 Command 对象
4. 处理变量替换（`${CLAUDE_PLUGIN_ROOT}`, `${CLAUDE_SKILL_DIR}`, `${user_config.X}` 等）
5. 支持内联命令内容（无需物理文件）

### 1.2 业务场景
- **插件初始化**: 系统启动时加载所有启用插件的命令
- **热重载**: 插件变更时重新加载命令
- **技能执行**: 用户调用 `/plugin:namespace:command` 格式的命令
- **Bare 模式**: `--bare` 标志跳过自动加载，仅加载显式指定的插件

---

## 二、功能点目的

### 2.1 命令命名空间系统
```typescript
// 命令命名格式: pluginName:namespace:commandName
// 示例: my-plugin:utils:formatter
```
- **Skill 文件**（SKILL.md）: 使用父目录名作为命令基础名
- **普通文件**（.md）: 使用文件名（不含扩展名）作为基础名
- **命名空间**: 基于文件相对于 commands/ 目录的路径构建

### 2.2 命令元数据解析（Frontmatter）
支持的 frontmatter 字段：
| 字段 | 类型 | 说明 |
|------|------|------|
| `description` | string | 命令描述 |
| `name` | string | 显示名称（userFacingName） |
| `model` | string | 指定模型（如 'claude-sonnet-4-20250514'） |
| `effort` | string/number | 努力级别（low/medium/high 或 1-5） |
| `allowed-tools` | string[] | 允许的工具列表 |
| `arguments` | string/string[] | 参数名列表 |
| `argument-hint` | string | 参数提示文本 |
| `when_to_use` | string | 使用场景说明 |
| `version` | string | 版本号 |
| `disable-model-invocation` | boolean | 禁止模型调用 |
| `user-invocable` | boolean | 用户是否可调用（默认 true） |
| `shell` | string | Shell 执行配置 |

### 2.3 变量替换系统
在 `getPromptForCommand` 中执行的变量替换链：

1. **参数替换**: `substituteArguments` - 替换 `${argName}` 为实际参数值
2. **插件变量**: `substitutePluginVariables` - 替换 `${CLAUDE_PLUGIN_ROOT}`, `${CLAUDE_PLUGIN_DATA}`
3. **用户配置**: `substituteUserConfigInContent` - 替换 `${user_config.X}`
4. **技能目录**: `${CLAUDE_SKILL_DIR}` - 替换为技能所在目录（仅技能模式）
5. **会话 ID**: `${CLAUDE_SESSION_ID}` - 替换为当前会话 ID
6. **Shell 执行**: `executeShellCommandsInPrompt` - 执行 `${...}` 中的 shell 命令

---

## 三、具体技术实现

### 3.1 核心数据类型

```typescript
// 插件 Markdown 文件表示
type PluginMarkdownFile = {
  filePath: string
  baseDir: string
  frontmatter: FrontmatterData
  content: string
}

// 加载配置
type LoadConfig = {
  isSkillMode: boolean  // true 表示从 skills/ 目录加载
}
```

### 3.2 关键流程

#### 3.2.1 命令加载流程（`getPluginCommands`）
```
getPluginCommands (memoized)
  └── 检查 bare 模式 → 可能返回空数组
  └── loadAllPluginsCacheOnly() → 获取启用的插件列表
  └── 并行处理每个插件
      ├── 创建 loadedPaths Set（去重）
      ├── 加载默认 commands 目录
      │   └── loadCommandsFromDirectory
      │       ├── collectMarkdownFiles → 递归收集 .md 文件
      │       ├── transformPluginSkillFiles → Skill 文件转换
      │       └── createPluginCommand → 创建 Command 对象
      ├── 加载自定义 commandsPaths
      │   ├── 支持目录（批量加载）
      │   └── 支持单个 .md 文件
      │   └── 支持 metadata 覆盖（object-mapping 格式）
      └── 加载内联命令（commandsMetadata 中的 content 字段）
  └── 合并所有命令并返回
```

#### 3.2.2 Skill 加载流程（`getPluginSkills`）
```
getPluginSkills (memoized)
  └── 检查 bare 模式
  └── loadAllPluginsCacheOnly()
  └── 并行处理每个插件
      ├── 加载默认 skillsPath
      │   └── loadSkillsFromDirectory
      │       ├── 尝试直接加载 skillsPath/SKILL.md
      │       └── 或扫描子目录查找 SKILL.md
      └── 加载自定义 skillsPaths
  └── 合并所有 Skills
```

### 3.3 关键函数实现

#### 3.3.1 `createPluginCommand` - 命令对象工厂
```typescript
function createPluginCommand(
  commandName: string,
  file: PluginMarkdownFile,
  sourceName: string,
  pluginManifest: PluginManifest,
  pluginPath: string,
  isSkill: boolean,
  config: LoadConfig = { isSkillMode: false },
): Command | null
```
构建的 Command 对象包含：
- **基础属性**: type, name, description, source, pluginInfo
- **执行属性**: getPromptForCommand（异步函数，返回 ContentBlockParam[]）
- **UI 属性**: userFacingName(), isHidden, progressMessage
- **模型配置**: model, effort, disableModelInvocation
- **权限配置**: allowedTools

#### 3.3.2 `transformPluginSkillFiles` - Skill 目录处理
处理包含 SKILL.md 的目录：
- 如果目录包含 SKILL.md，仅保留 SKILL.md 文件（忽略其他 .md）
- 如果存在多个 SKILL.md，使用第一个并记录警告
- 普通目录保留所有 .md 文件

### 3.4 缓存机制
- **`getPluginCommands`**: 使用 lodash memoize，缓存所有插件命令
- **`getPluginSkills`**: 使用 lodash memoize，缓存所有插件技能
- **缓存清除**: `clearPluginCommandCache()` 和 `clearPluginSkillsCache()`

---

## 四、关键代码路径与文件引用

### 4.1 入口点
| 函数 | 导出类型 | 调用方 |
|------|----------|--------|
| `getPluginCommands` | memoized async | `src/commands.ts`, `src/utils/plugins/pluginLoader.ts` |
| `getPluginSkills` | memoized async | `src/utils/plugins/pluginLoader.ts` |
| `clearPluginCommandCache` | function | `src/utils/plugins/cacheUtils.ts` |
| `clearPluginSkillsCache` | function | `src/utils/plugins/cacheUtils.ts` |

### 4.2 关键依赖
```typescript
// 核心依赖
import { loadAllPluginsCacheOnly } from './pluginLoader.js'
import { parseFrontmatter } from '../frontmatterParser.js'
import { walkPluginMarkdown } from './walkPluginMarkdown.js'
import { substitutePluginVariables, loadPluginOptions } from './pluginOptionsStorage.js'
import { executeShellCommandsInPrompt } from '../promptShellExecution.js'

// 类型定义
import type { Command } from '../../types/command.js'
import type { PluginManifest, CommandMetadata } from './schemas.js'
```

### 4.3 文件引用关系
```
loadPluginCommands.ts
  ├── pluginLoader.ts         # 加载插件列表
  ├── walkPluginMarkdown.ts   # 遍历插件目录
  ├── pluginOptionsStorage.ts # 插件选项存储
  ├── frontmatterParser.ts    # Frontmatter 解析
  ├── promptShellExecution.ts # Shell 命令执行
  ├── schemas.ts              # 类型定义
  └── ../../types/command.ts  # Command 类型
```

---

## 五、依赖与外部交互

### 5.1 上游依赖（被调用）
| 模块 | 用途 |
|------|------|
| `pluginLoader.ts` | 获取已启用的插件列表 |
| `walkPluginMarkdown.ts` | 递归遍历插件目录 |
| `frontmatterParser.ts` | 解析 Markdown frontmatter |
| `pluginOptionsStorage.ts` | 加载用户配置选项 |
| `promptShellExecution.ts` | 执行 prompt 中的 shell 命令 |
| `argumentSubstitution.ts` | 参数替换逻辑 |
| `fsOperations.ts` | 文件系统操作抽象 |

### 5.2 下游消费者（调用方）
| 模块 | 用途 |
|------|------|
| `src/commands.ts` | 合并所有命令（内置 + 插件） |
| `src/utils/plugins/pluginLoader.ts` | 加载插件组件 |
| `src/utils/plugins/cacheUtils.ts` | 缓存清除 |
| `src/utils/plugins/refresh.ts` | 热重载 |

### 5.3 配置交互
- **Settings**: 通过 `loadAllPluginsCacheOnly` 间接读取 `enabledPlugins`
- **Bare 模式**: 通过 `isBareMode()` 检查 `--bare` 标志
- **内联插件**: 通过 `getInlinePlugins()` 获取 `--plugin-dir` 指定的插件

---

## 六、风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 路径遍历风险
- **位置**: `loadLspServersFromManifest`（lspPluginIntegration.ts 有类似逻辑）
- **缓解**: 本模块通过 `isDuplicatePath` 和路径规范化防止重复加载

#### 6.1.2 缓存一致性问题
- **风险**: 文件系统变更后缓存可能过期
- **缓解**: 提供 `clearPluginCommandCache()` 和 `clearPluginSkillsCache()` 供外部调用

#### 6.1.3 Shell 命令注入
- **位置**: `executeShellCommandsInPrompt`
- **缓解**: 通过 `shell` 配置控制允许执行的命令

### 6.2 边界情况

| 场景 | 处理方式 |
|------|----------|
| 重复文件路径 | `loadedPaths` Set 去重，每插件独立 |
| 多个 SKILL.md | 使用第一个，记录警告日志 |
| 无效的 frontmatter | 记录错误，跳过该命令 |
| 内联内容无 source | 通过 metadata.content 加载 |
| Bare 模式 | 仅加载显式指定的内联插件 |

### 6.3 改进建议

#### 6.3.1 性能优化
1. **并行加载优化**: 当前已使用 `Promise.all` 并行处理插件，但每个插件内部的多个路径仍串行
2. **增量加载**: 支持仅加载变更的插件，而非全部重新加载
3. **文件监听**: 集成文件系统监听实现真正的热重载

#### 6.3.2 可维护性
1. **错误分类**: 当前错误统一为调试日志，建议区分用户可见错误和内部错误
2. **类型安全**: `PluginMarkdownFile` 与 `Command` 的映射关系可进一步类型化
3. **测试覆盖**: 复杂的变量替换链需要更多单元测试

#### 6.3.3 功能扩展
1. **命令别名**: 支持为命令定义多个别名
2. **条件加载**: 基于环境或平台条件加载命令
3. **命令分组**: 支持对命令进行逻辑分组，便于 UI 展示

### 6.4 技术债务
1. **DEPRECATED API**: 依赖 `getSettings_DEPRECATED`，需关注迁移计划
2. **平台判断**: `process.platform === 'win32'` 硬编码，建议抽象到工具模块
3. **魔术字符串**: `${CLAUDE_PLUGIN_ROOT}` 等变量名分散在多处

---

## 七、附录

### 7.1 命令命名示例
```
插件结构:
my-plugin/
  commands/
    build.md              → my-plugin:build
    utils/
      formatter.md        → my-plugin:utils:formatter
  skills/
    my-skill/
      SKILL.md            → my-plugin:my-skill
```

### 7.2 缓存清除触发点
- `src/utils/plugins/cacheUtils.ts:clearAllCaches()` - 全局缓存清除
- `src/hooks/useManagePlugins.ts` - 插件管理操作后
- `src/utils/plugins/refresh.ts` - 热重载时
