# loadOutputStylesDir.ts 研究文档

## 1. 场景与职责

### 1.1 功能定位

`loadOutputStylesDir.ts` 是 Claude Code CLI 中负责**加载用户自定义输出样式**的核心模块。输出样式（Output Style）是一种允许用户自定义 Claude 响应行为的配置机制，通过 Markdown 文件定义不同的系统提示词（system prompt）和行为模式。

### 1.2 业务场景

该模块服务于以下业务场景：

1. **自定义输出风格**：用户可以通过在项目中创建 `.claude/output-styles/*.md` 文件来定义自定义的输出风格
2. **用户级全局样式**：用户可以在 `~/.claude/output-styles/` 目录下创建全局可用的输出样式
3. **样式继承与覆盖**：项目级样式覆盖用户级样式，形成层级化的配置体系
4. **与内置样式集成**：自定义样式与内置样式（Default、Explanatory、Learning）合并，供用户选择

### 1.3 在架构中的位置

```
┌─────────────────────────────────────────────────────────────┐
│                    Output Style 系统                         │
├─────────────────────────────────────────────────────────────┤
│  UI 层: OutputStylePicker.tsx                                │
│       └── 调用 getAllOutputStyles()                          │
├─────────────────────────────────────────────────────────────┤
│  聚合层: constants/outputStyles.ts                           │
│       ├── getAllOutputStyles()                               │
│       ├── getOutputStyleConfig()                             │
│       └── 合并内置 + 目录 + 插件样式                          │
├─────────────────────────────────────────────────────────────┤
│  目录加载层: outputStyles/loadOutputStylesDir.ts  ◄── 本文件 │
│       └── getOutputStyleDirStyles()                          │
│       └── clearOutputStyleCaches()                           │
├─────────────────────────────────────────────────────────────┤
│  插件加载层: utils/plugins/loadPluginOutputStyles.ts         │
│       └── loadPluginOutputStyles()                           │
├─────────────────────────────────────────────────────────────┤
│  基础设施层: utils/markdownConfigLoader.ts                   │
│       └── loadMarkdownFilesForSubdir()                       │
└─────────────────────────────────────────────────────────────┘
```

## 2. 功能点目的

### 2.1 核心功能

| 功能 | 说明 |
|------|------|
| `getOutputStyleDirStyles()` | 从文件系统加载所有自定义输出样式（项目级 + 用户级） |
| `clearOutputStyleCaches()` | 清除所有输出样式相关的缓存，用于配置刷新 |

### 2.2 样式来源与优先级

该模块处理两种来源的样式文件：

1. **项目级样式**（`projectSettings`）：从 `cwd` 向上遍历到 git root 的所有 `.claude/output-styles/*.md` 文件
2. **用户级样式**（`userSettings`）：`~/.claude/output-styles/*.md` 文件
3. **托管级样式**（`policySettings`）：通过托管配置下发的样式

**优先级规则**（由 `constants/outputStyles.ts` 中的 `getAllOutputStyles` 实现）：
- 低优先级：内置样式（built-in）
- 中优先级：插件样式（plugin）
- 较高优先级：托管样式（policySettings）
- 高优先级：用户样式（userSettings）
- 最高优先级：项目样式（projectSettings）

### 2.3 Frontmatter 支持

每个输出样式 Markdown 文件支持以下 frontmatter 字段：

```yaml
---
name: "自定义样式名称"           # 可选，默认使用文件名
description: "样式描述"         # 可选，默认从内容提取
keep-coding-instructions: true  # 可选，是否保留编码指令
---
```

**注意**：`force-for-plugin` 字段在此模块中会被检测并警告（仅适用于插件输出样式）。

## 3. 具体技术实现

### 3.1 核心数据结构

```typescript
// 输出样式配置（定义在 constants/outputStyles.ts）
type OutputStyleConfig = {
  name: string
  description: string
  prompt: string
  source: SettingSource | 'built-in' | 'plugin'
  keepCodingInstructions?: boolean
  forceForPlugin?: boolean  // 仅插件样式使用
}

// Markdown 文件解析结果（定义在 markdownConfigLoader.ts）
type MarkdownFile = {
  filePath: string
  baseDir: string
  frontmatter: FrontmatterData
  content: string
  source: SettingSource
}
```

### 3.2 关键流程

#### 3.2.1 样式加载流程

```
getOutputStyleDirStyles(cwd)
    │
    ▼
loadMarkdownFilesForSubdir('output-styles', cwd)
    │
    ├──► 扫描 managedDir (policySettings)
    ├──► 扫描 userDir (userSettings)      [条件启用]
    └──► 扫描 projectDirs (projectSettings) [条件启用]
            │
            └──► getProjectDirsUpToHome()
                    │
                    └──► 从 cwd 向上遍历到 git root
                    └──► 收集所有 .claude/output-styles/ 目录
    │
    ▼
去重（基于 inode）
    │
    ▼
解析每个 Markdown 文件
    │
    ▼
提取 frontmatter + content
    │
    ▼
构建 OutputStyleConfig 数组
```

#### 3.2.2 单个文件解析逻辑

```typescript
// 伪代码表示
function parseStyleFile(file): OutputStyleConfig | null {
  const fileName = basename(file.filePath)
  const styleName = fileName.replace(/\.md$/, '')
  
  // 1. 提取名称（frontmatter.name > 文件名）
  const name = frontmatter['name'] || styleName
  
  // 2. 提取描述（frontmatter.description > 内容第一行 > 默认描述）
  const description = coerceDescriptionToString(frontmatter['description']) 
    ?? extractDescriptionFromMarkdown(content, defaultDesc)
  
  // 3. 解析 keep-coding-instructions 标志
  const keepCodingInstructions = parseBooleanFlag(
    frontmatter['keep-coding-instructions']
  )
  
  // 4. 警告：force-for-plugin 在此不生效
  if (frontmatter['force-for-plugin'] !== undefined) {
    logForDebugging('警告：force-for-plugin 仅适用于插件输出样式')
  }
  
  // 5. 构建配置对象
  return {
    name,
    description,
    prompt: content.trim(),
    source: file.source,  // 'policySettings' | 'userSettings' | 'projectSettings'
    keepCodingInstructions,
  }
}
```

### 3.3 缓存机制

该模块使用 `lodash-es/memoize` 实现函数级缓存：

```typescript
// getOutputStyleDirStyles 被 memoize 包装
export const getOutputStyleDirStyles = memoize(async (cwd: string) => {
  // ...
})

// 缓存清除函数
export function clearOutputStyleCaches(): void {
  getOutputStyleDirStyles.cache?.clear?.()  // 清除本模块缓存
  loadMarkdownFilesForSubdir.cache?.clear?.()  // 清除底层缓存
  clearPluginOutputStyleCache()  // 清除插件样式缓存
}
```

**缓存策略**：
- 缓存键：`cwd` 参数值
- 缓存生命周期：进程级（内存中）
- 刷新机制：通过 `clearOutputStyleCaches()` 手动清除

### 3.4 错误处理

| 错误场景 | 处理方式 |
|---------|---------|
| 文件读取失败 | `logError(error)`，返回 `null`，过滤掉该样式 |
| 单个文件解析失败 | `try-catch` 捕获，`logError(error)`，继续处理其他文件 |
| 整体加载失败 | `logError(error)`，返回空数组 `[]` |

## 4. 关键代码路径与文件引用

### 4.1 直接依赖

| 文件 | 用途 |
|------|------|
| `src/constants/outputStyles.ts` | `OutputStyleConfig` 类型定义，聚合所有样式 |
| `src/utils/markdownConfigLoader.ts` | `loadMarkdownFilesForSubdir()`, `extractDescriptionFromMarkdown()` |
| `src/utils/frontmatterParser.ts` | `coerceDescriptionToString()`, `parseFrontmatter()` |
| `src/utils/debug.ts` | `logForDebugging()` 调试日志 |
| `src/utils/log.ts` | `logError()` 错误日志 |
| `src/utils/plugins/loadPluginOutputStyles.ts` | `clearPluginOutputStyleCache()` |

### 4.2 调用链

```
入口点
├── src/components/OutputStylePicker.tsx
│   └── getAllOutputStyles(getCwd())
│       └── 调用 getOutputStyleDirStyles()
│
├── src/constants/outputStyles.ts
│   ├── getAllOutputStyles()  [聚合]
│   ├── getOutputStyleConfig()  [获取当前生效样式]
│   └── clearAllOutputStylesCache()  [清除缓存]
│
└── src/constants/prompts.ts
    └── getOutputStyleConfig()  [在系统提示词中使用]
```

### 4.3 目录扫描范围

```typescript
// 由 markdownConfigLoader.ts 中的 getProjectDirsUpToHome 实现
// 扫描路径示例（cwd = /home/user/project/subdir）：
[
  '/home/user/project/subdir/.claude/output-styles',   // 最近（最高优先级）
  '/home/user/project/.claude/output-styles',          // 父目录
  // 停止于 git root
]

// 用户目录（始终检查）
'/home/user/.claude/output-styles'

// 托管目录（始终检查）
'{managedPath}/.claude/output-styles'
```

## 5. 依赖与外部交互

### 5.1 运行时依赖

```typescript
// 外部库
import memoize from 'lodash-es/memoize.js'  // 缓存
import { basename } from 'path'              // 路径处理

// 项目内部模块
import type { OutputStyleConfig } from '../constants/outputStyles.js'
import { logForDebugging } from '../utils/debug.js'
import { coerceDescriptionToString } from '../utils/frontmatterParser.js'
import { logError } from '../utils/log.js'
import {
  extractDescriptionFromMarkdown,
  loadMarkdownFilesForSubdir,
} from '../utils/markdownConfigLoader.js'
import { clearPluginOutputStyleCache } from '../utils/plugins/loadPluginOutputStyles.js'
```

### 5.2 配置开关

样式加载受以下配置影响（在 `markdownConfigLoader.ts` 中检查）：

```typescript
// 用户设置是否启用
isSettingSourceEnabled('userSettings')

// 项目设置是否启用
isSettingSourceEnabled('projectSettings')

// 插件限制检查
isRestrictedToPluginOnly('agents')  // 影响 agents 加载，但不直接影响 output-styles
```

### 5.3 与插件系统的交互

虽然本模块专注于目录样式加载，但通过 `clearOutputStyleCaches()` 与插件系统联动：

```
用户执行 /plugins 更新插件
    │
    ▼
触发 clearOutputStyleCaches()
    │
    ├──► 清除 getOutputStyleDirStyles 缓存
    ├──► 清除 loadMarkdownFilesForSubdir 缓存
    └──► 清除 loadPluginOutputStyles 缓存
                │
                └──► 插件样式重新加载
```

## 6. 风险、边界与改进建议

### 6.1 已知风险

| 风险 | 严重程度 | 说明 |
|------|---------|------|
| 缓存不一致 | 中 | 文件系统变更后缓存未自动刷新，需手动调用 `clearOutputStyleCaches()` |
| 命名冲突 | 低 | 不同来源的样式使用相同 name 时，高优先级会覆盖低优先级，但无警告 |
| 文件解析失败静默 | 低 | 单个文件解析失败仅记录日志，用户可能感知不到 |
| force-for-plugin 误用 | 低 | 用户在目录样式中设置 `force-for-plugin` 会收到警告，但无实际效果 |

### 6.2 边界情况

1. **空 frontmatter**：正常处理，使用默认值
2. **缺失 description**：从 Markdown 内容第一行提取，或 fallback 到默认描述
3. **无效 keep-coding-instructions**：非布尔/字符串值会被解析为 `undefined`
4. **符号链接**：在底层 `loadMarkdownFilesForSubdir` 中通过 inode 去重处理
5. **Worktree 场景**：底层逻辑会检测 git worktree 并正确处理主仓库的样式

### 6.3 改进建议

#### 6.3.1 短期改进

1. **添加文件监听刷新**
   ```typescript
   // 建议：在开发模式下监听文件变更自动刷新缓存
   if (process.env.NODE_ENV === 'development') {
     watchConfigDirs('output-styles', () => clearOutputStyleCaches())
   }
   ```

2. **增强错误报告**
   - 当前：单个文件错误仅记录到 debug 日志
   - 建议：汇总加载错误，在 UI 中提示用户哪些文件加载失败

3. **命名冲突警告**
   - 当不同来源的样式使用相同 name 时，在 debug 日志中记录覆盖关系

#### 6.3.2 中长期改进

1. **Schema 验证**
   - 当前：frontmatter 字段无严格校验，任意类型值可能导致意外行为
   - 建议：使用 Zod 等库对 frontmatter 进行 Schema 验证

2. **性能优化**
   - 当前：每次调用都重新读取所有文件（虽然有缓存）
   - 建议：考虑使用文件系统监听 + 增量更新，而非全量重载

3. **与插件样式统一**
   - 当前：目录样式和插件样式有重复解析逻辑
   - 建议：抽象统一的 `OutputStyleParser` 类，供两者复用

### 6.4 测试建议

该模块目前**无直接单元测试**，建议补充：

1. **正常路径测试**：
   - 从多种来源加载样式文件
   - 验证 frontmatter 解析正确性
   - 验证优先级覆盖逻辑

2. **异常路径测试**：
   - 损坏的 Markdown 文件
   - 无效的 frontmatter YAML
   - 不可读的目录/文件

3. **缓存测试**：
   - 验证 memoize 缓存生效
   - 验证 `clearOutputStyleCaches()` 正确清除缓存

---

*文档生成时间：2026-04-01*
*研究范围：代码、配置、依赖关系*
*executor=kimi; model=k2p5*
