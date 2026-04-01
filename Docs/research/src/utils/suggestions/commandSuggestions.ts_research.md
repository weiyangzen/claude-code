# commandSuggestions.ts 深度研究文档

## 场景与职责

`commandSuggestions.ts` 是 Claude Code CLI 的**命令自动补全核心引擎**，负责处理用户输入斜杠命令 (`/command`) 时的智能提示功能。该模块在终端交互中扮演关键角色：

1. **命令发现**：当用户输入 `/` 或部分命令名时，提供可用的斜杠命令列表
2. **模糊搜索**：基于 Fuse.js 实现智能模糊匹配，支持命令名、别名、描述的搜索
3. **输入中命令检测**：识别用户在输入中间位置输入的斜杠命令（如 `tell me about /com`）
4. **幽灵文本补全**：为部分输入的命令提供最佳匹配的后缀补全建议
5. **命令高亮**：在文本中定位所有斜杠命令位置，用于 UI 高亮显示

该模块是 REPL (Read-Eval-Print Loop) 交互体验的核心组件，直接影响用户发现和执行命令的效率。

## 功能点目的

### 1. 智能命令搜索 (`generateCommandSuggestions`)
- **目的**：根据用户输入返回排序后的命令建议列表
- **排序策略**：
  - 最近使用的命令（基于使用频率和时效性评分）
  - 内置命令（`local`, `local-jsx` 类型）
  - 用户设置命令（`userSettings`, `localSettings`）
  - 项目设置命令（`projectSettings`）
  - 策略设置命令（`policySettings`）
  - 其他命令（插件、MCP 等）

### 2. 模糊匹配引擎 (`getCommandFuse`)
- **目的**：使用 Fuse.js 实现高效的模糊搜索
- **权重配置**：
  - 命令名 (`commandName`): 权重 3（最高）
  - 命令部分 (`partKey`): 权重 2
  - 别名 (`aliasKey`): 权重 2
  - 描述 (`descriptionKey`): 权重 0.5
- **缓存机制**：基于命令数组身份（identity）缓存 Fuse 索引，避免每次输入重建

### 3. 输入中命令检测 (`findMidInputSlashCommand`)
- **目的**：检测用户在非开头位置输入的斜杠命令
- **匹配规则**：`/` 前必须是空白字符，后接可选的字母数字下划线冒号
- **性能优化**：避免使用 lookbehind 正则（JSC JIT 性能问题），改用捕获组

### 4. 幽灵文本补全 (`getBestCommandMatch`)
- **目的**：为部分输入的命令提供最佳前缀匹配，用于内联补全
- **匹配逻辑**：优先返回前缀匹配的结果（如 `com` → `commit`）

### 5. 命令应用 (`applyCommandSuggestion`)
- **目的**：处理用户选择建议后的输入更新和执行逻辑
- **自动执行**：无参数命令在选择后自动执行

### 6. 斜杠命令高亮 (`findSlashCommandPositions`)
- **目的**：在文本中定位所有斜杠命令位置，用于语法高亮
- **匹配规则**：要求 `/` 前有空白或位于字符串开头，避免匹配文件路径（如 `/usr/bin`）

## 具体技术实现

### 关键数据结构

```typescript
// 命令搜索项（Fuse 索引结构）
type CommandSearchItem = {
  descriptionKey: string[]      // 描述词列表（分词后）
  partKey: string[] | undefined // 命令名分割部分（如 ["git", "commit"]）
  commandName: string           // 完整命令名
  command: Command              // 原始命令对象
  aliasKey: string[] | undefined // 别名列表
}

// 输入中斜杠命令
type MidInputSlashCommand = {
  token: string           // 完整 token（如 "/com"）
  startPos: number        // "/" 的位置
  partialCommand: string  // 命令部分（如 "com"）
}
```

### 核心算法流程

#### 1. 命令建议生成流程
```
generateCommandSuggestions(input, commands)
├── 输入验证（必须以 "/" 开头，无参数）
├── 空查询处理（仅输入 "/"）
│   ├── 获取最近使用命令（基于 skillUsageTracking）
│   ├── 按来源分类排序命令
│   └── 返回分类后的建议列表
└── 非空查询处理
    ├── 查找隐藏命令的精确匹配（处理 OAuth 过期等场景）
    ├── Fuse 模糊搜索
    ├── 多维度排序（精确匹配 > 别名匹配 > 前缀匹配 > 模糊匹配）
    └── 合并隐藏命令结果（如需要）
```

#### 2. Fuse 索引构建
```typescript
function getCommandFuse(commands: Command[]): Fuse<CommandSearchItem> {
  // 缓存检查：基于数组身份（identity）比较
  if (fuseCache?.commands === commands) {
    return fuseCache.fuse
  }
  
  // 构建搜索数据
  const commandData = commands
    .filter(cmd => !cmd.isHidden)
    .map(cmd => ({
      descriptionKey: (cmd.description ?? '')
        .split(' ')
        .map(word => cleanWord(word))
        .filter(Boolean),
      partKey: commandName.split(SEPARATORS).filter(Boolean), // ["git", "commit"]
      commandName,
      command: cmd,
      aliasKey: cmd.aliases,
    }))
  
  // Fuse 配置：阈值 0.3（较严格），优先匹配开头
  const fuse = new Fuse(commandData, {
    includeScore: true,
    threshold: 0.3,
    location: 0,
    distance: 100,
    keys: [
      { name: 'commandName', weight: 3 },
      { name: 'partKey', weight: 2 },
      { name: 'aliasKey', weight: 2 },
      { name: 'descriptionKey', weight: 0.5 },
    ],
  })
}
```

#### 3. 排序优先级算法
```typescript
// 优先级从高到低：
1. 精确名称匹配 (aName === query)
2. 精确别名匹配 (aliases.some(alias => alias === query))
3. 前缀名称匹配 (aName.startsWith(query))
   - 前缀匹配中，优先较短的名称（更接近精确匹配）
4. 前缀别名匹配 (aliases.find(alias => alias.startsWith(query)))
   - 前缀别名匹配中，优先较短的别名
5. Fuse 分数 + 使用频率作为平局决胜
```

### 关键代码路径

| 功能 | 函数 | 行号 |
|------|------|------|
| 命令建议生成 | `generateCommandSuggestions` | 292-498 |
| Fuse 索引获取 | `getCommandFuse` | 30-80 |
| 输入中命令检测 | `findMidInputSlashCommand` | 114-154 |
| 最佳匹配获取 | `getBestCommandMatch` | 164-195 |
| 命令应用 | `applyCommandSuggestion` | 503-539 |
| 斜杠命令定位 | `findSlashCommandPositions` | 552-567 |
| 命令 ID 生成 | `getCommandId` | 233-244 |
| 别名匹配 | `findMatchedAlias` | 250-259 |
| 建议项创建 | `createCommandSuggestionItem` | 265-287 |

## 依赖与外部交互

### 直接依赖模块

```typescript
import Fuse from 'fuse.js'  // 模糊搜索库
import {
  type Command,
  formatDescriptionWithSource,
  getCommand,
  getCommandName,
} from '../../commands.js'  // 命令定义和工具函数
import type { SuggestionItem } from '../../components/PromptInput/PromptInputFooterSuggestions.js'  // UI 类型
import { getSkillUsageScore } from './skillUsageTracking.js'  // 技能使用评分
```

### 依赖详解

1. **fuse.js** (外部库)
   - 用途：高性能模糊搜索
   - 配置：阈值 0.3，支持多字段加权搜索

2. **commands.js** (内部模块)
   - `Command` 类型：命令对象结构
   - `getCommandName`: 获取命令显示名
   - `getCommand`: 根据名称获取命令对象
   - `formatDescriptionWithSource`: 格式化带来源的描述

3. **PromptInputFooterSuggestions.tsx** (UI 组件)
   - `SuggestionItem` 类型：建议项 UI 结构
   - 字段：`id`, `displayText`, `tag`, `description`, `metadata`, `color`

4. **skillUsageTracking.js** (内部模块)
   - `getSkillUsageScore`: 获取技能使用评分，用于最近使用排序

### 调用方

- `useTypeahead.tsx`: 类型提示钩子，调用 `generateCommandSuggestions`
- `PromptInput.tsx`: 输入处理，调用 `findMidInputSlashCommand`, `applyCommandSuggestion`
- `unifiedSuggestions.ts`: 统一建议处理

## 风险、边界与改进建议

### 已知风险

1. **缓存失效风险**
   - Fuse 缓存基于命令数组的 identity 比较
   - 如果命令对象被修改但未重建数组，缓存可能返回过期索引
   - **缓解**：命令数组在 REPL.tsx 中通过 memoization 保持稳定

2. **隐藏命令处理复杂性**
   - 代码需要处理隐藏命令（`isHidden`）的精确匹配场景
   - OAuth 过期或功能开关可能导致命令中途隐藏/显示
   - **当前处理**：隐藏命令的精确匹配被追加到 Fuse 结果前

3. **正则性能陷阱**
   - 避免使用 lookbehind `(?<=\s)`，因为它会 defeat YARR JIT
   - 改用捕获组并手动计算偏移量

### 边界情况

| 场景 | 处理方式 |
|------|----------|
| 仅输入 "/" | 显示最近使用 + 分类排序的命令列表 |
| 隐藏命令精确匹配 | 优先显示，但避免与可见命令重复 |
| 输入中斜杠命令 | 检测 `/` 前为空白字符的命令模式 |
| 光标在命令后空格 | 不显示幽灵文本（`cursorOffset > slashPos + 1 + fullCommand.length`） |
| 命令有别名 | 用户输入别名时显示别名在括号中 |
| 无匹配结果 | 返回空数组，UI 不显示建议 |

### 改进建议

1. **性能优化**
   - 考虑使用 Web Worker 处理 Fuse 搜索（大量命令时）
   - 对 `generateCommandSuggestions` 进行 memoization

2. **功能增强**
   - 支持命令描述的中文分词（当前仅按空格分割）
   - 添加命令使用频率的持久化缓存
   - 支持命令参数的智能提示

3. **代码质量**
   - 将排序逻辑提取为可配置的排序策略
   - 增加单元测试覆盖边界情况（隐藏命令、别名匹配等）
   - 考虑使用更类型安全的方式处理 `SuggestionItem.metadata`

4. **可访问性**
   - 为命令建议添加键盘导航的 ARIA 标签支持
   - 提供命令描述的语音朗读支持

### 测试要点

- 模糊搜索的准确性（特别是多词描述匹配）
- 最近使用命令的排序稳定性
- 隐藏命令在各种场景下的显示逻辑
- 输入中命令检测的边界情况
- 大命令集（1000+）的性能表现
