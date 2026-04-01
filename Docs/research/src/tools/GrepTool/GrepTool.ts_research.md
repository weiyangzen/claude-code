# GrepTool.ts 深度研究文档

## 场景与职责

GrepTool 是 Claude Code 的核心搜索工具，基于 ripgrep (rg) 提供高性能正则表达式搜索能力。它是 Agent 与代码库交互的主要入口之一，用于：

1. **代码搜索**：在文件内容中搜索正则表达式模式
2. **文件发现**：查找包含特定模式的文件
3. **统计分析**：统计匹配项数量
4. **代码导航**：支持上下文行展示，帮助理解代码结构

该工具被设计为只读工具，具有并发安全性，是 Agent 工具链中最基础且高频使用的工具之一。

## 功能点目的

### 核心功能

| 功能 | 目的 |
|------|------|
| 正则搜索 | 基于 ripgrep 的高性能内容搜索 |
| 文件过滤 | 通过 glob/type 参数限定搜索范围 |
| 输出模式 | 支持 content/files_with_matches/count 三种模式 |
| 上下文展示 | 支持 -A/-B/-C 参数展示匹配行上下文 |
| 分页控制 | head_limit/offset 实现结果分页 |
| 权限检查 | 集成文件系统权限系统 |

### 输出模式详解

1. **content 模式**：返回匹配行的实际内容，包含行号
2. **files_with_matches 模式**（默认）：仅返回匹配的文件路径列表，按修改时间排序
3. **count 模式**：返回每个文件的匹配数量统计

## 具体技术实现

### 输入 Schema 定义

```typescript
const inputSchema = lazySchema(() =>
  z.strictObject({
    pattern: z.string(),           // 正则表达式模式
    path: z.string().optional(),   // 搜索路径，默认当前工作目录
    glob: z.string().optional(),   // glob 过滤模式
    output_mode: z.enum(['content', 'files_with_matches', 'count']).optional(),
    '-B': semanticNumber(z.number().optional()),  // 前置上下文行数
    '-A': semanticNumber(z.number().optional()),  // 后置上下文行数
    '-C': semanticNumber(z.number().optional()),  // 双向上下文行数
    context: semanticNumber(z.number().optional()),
    '-n': semanticBoolean(z.boolean().optional()), // 显示行号，默认 true
    '-i': semanticBoolean(z.boolean().optional()), // 忽略大小写
    type: z.string().optional(),   // 文件类型（rg --type）
    head_limit: semanticNumber(z.number().optional()), // 结果限制，默认 250
    offset: semanticNumber(z.number().optional()),     // 跳过前 N 条
    multiline: semanticBoolean(z.boolean().optional()), // 多行模式
  }),
)
```

### 关键数据结构

```typescript
// 输出数据结构
const outputSchema = lazySchema(() =>
  z.object({
    mode: z.enum(['content', 'files_with_matches', 'count']).optional(),
    numFiles: z.number(),
    filenames: z.array(z.string()),
    content: z.string().optional(),
    numLines: z.number().optional(),    // content 模式专用
    numMatches: z.number().optional(),  // count 模式专用
    appliedLimit: z.number().optional(),
    appliedOffset: z.number().optional(),
  }),
)
```

### 核心流程

#### 1. 工具构建（buildTool）

```typescript
export const GrepTool = buildTool({
  name: GREP_TOOL_NAME,  // 'Grep'
  searchHint: 'search file contents with regex (ripgrep)',
  maxResultSizeChars: 20_000,  // 20KB 持久化阈值
  strict: true,
  // ... 各生命周期方法
})
```

#### 2. 输入验证（validateInput）

- 检查指定路径是否存在
- 对 UNC 路径进行安全检查（防止 NTLM 凭证泄漏）
- 路径不存在时提供智能建议（基于当前工作目录）

#### 3. 权限检查（checkPermissions）

```typescript
async checkPermissions(input, context): Promise<PermissionDecision> {
  const appState = context.getAppState()
  return checkReadPermissionForTool(
    GrepTool,
    input,
    appState.toolPermissionContext,
  )
}
```

权限检查委托给 `checkReadPermissionForTool`，支持：
- 读取权限规则匹配
- 工作目录边界检查
- UNC 路径防护
- 危险文件检测

#### 4. 核心调用流程（call 方法）

**ripgrep 参数构建**：

```typescript
const args = ['--hidden']

// 1. 排除版本控制目录
for (const dir of VCS_DIRECTORIES_TO_EXCLUDE) {
  args.push('--glob', `!${dir}`)
}

// 2. 限制行长度（防止 base64/压缩内容）
args.push('--max-columns', '500')

// 3. 多行模式
if (multiline) {
  args.push('-U', '--multiline-dotall')
}

// 4. 大小写敏感
if (case_insensitive) {
  args.push('-i')
}

// 5. 输出模式
if (output_mode === 'files_with_matches') {
  args.push('-l')
} else if (output_mode === 'count') {
  args.push('-c')
}

// 6. 行号
if (show_line_numbers && output_mode === 'content') {
  args.push('-n')
}

// 7. 上下文行
if (context !== undefined) {
  args.push('-C', context.toString())
}

// 8. 模式处理（以 - 开头的模式使用 -e 参数）
if (pattern.startsWith('-')) {
  args.push('-e', pattern)
} else {
  args.push(pattern)
}

// 9. 文件类型过滤
if (type) {
  args.push('--type', type)
}

// 10. glob 过滤
if (glob) {
  // 支持逗号和空格分隔，保留花括号模式
  const globPatterns = parseGlobPatterns(glob)
  for (const globPattern of globPatterns) {
    args.push('--glob', globPattern)
  }
}

// 11. 忽略模式（来自权限系统的 deny 规则）
const ignorePatterns = getFileReadIgnorePatterns(appState.toolPermissionContext)
for (const ignorePattern of ignorePatterns) {
  const rgIgnorePattern = ignorePattern.startsWith('/')
    ? `!${ignorePattern}`
    : `!**/${ignorePattern}`
  args.push('--glob', rgIgnorePattern)
}

// 12. 孤儿插件缓存排除
for (const exclusion of await getGlobExclusionsForPluginCache(absolutePath)) {
  args.push('--glob', exclusion)
}
```

**执行搜索**：

```typescript
const results = await ripGrep(args, absolutePath, abortController.signal)
```

**结果处理**：

- **content 模式**：直接返回匹配行，转换为相对路径
- **count 模式**：解析 `文件路径:计数` 格式，统计总数
- **files_with_matches 模式**：按修改时间排序，应用分页限制

#### 5. 分页实现（applyHeadLimit）

```typescript
function applyHeadLimit<T>(
  items: T[],
  limit: number | undefined,
  offset: number = 0,
): { items: T[]; appliedLimit: number | undefined } {
  // limit === 0 表示无限制
  if (limit === 0) {
    return { items: items.slice(offset), appliedLimit: undefined }
  }
  const effectiveLimit = limit ?? DEFAULT_HEAD_LIMIT  // 默认 250
  const sliced = items.slice(offset, offset + effectiveLimit)
  const wasTruncated = items.length - offset > effectiveLimit
  return {
    items: sliced,
    appliedLimit: wasTruncated ? effectiveLimit : undefined,
  }
}
```

### 关键常量

```typescript
// 版本控制目录排除列表
const VCS_DIRECTORIES_TO_EXCLUDE = [
  '.git', '.svn', '.hg', '.bzr', '.jj', '.sl'
] as const

// 默认结果限制
const DEFAULT_HEAD_LIMIT = 250
```

## 关键代码路径与文件引用

### 内部依赖

| 文件 | 用途 |
|------|------|
| `src/Tool.ts` | Tool 类型定义、buildTool 工厂函数 |
| `src/utils/ripgrep.ts` | ripgrep 执行封装（ripGrep 函数） |
| `src/utils/permissions/filesystem.ts` | 文件读取权限检查 |
| `src/utils/permissions/shellRuleMatching.ts` | 通配符模式匹配 |
| `src/utils/plugins/orphanedPluginFilter.ts` | 孤儿插件缓存排除 |
| `src/utils/semanticNumber.ts` | 语义化数字（支持字符串数字） |
| `src/utils/semanticBoolean.ts` | 语义化布尔（支持 "true"/"false"） |
| `src/utils/path.ts` | 路径展开、相对路径转换 |
| `src/utils/file.ts` | 文件工具函数 |
| `src/utils/cwd.ts` | 当前工作目录获取 |

### UI 组件

| 文件 | 用途 |
|------|------|
| `src/tools/GrepTool/UI.tsx` | 工具使用消息、结果渲染 |
| `src/tools/GrepTool/prompt.ts` | 工具描述、名称常量 |

### 调用方

- `src/tools.ts` - 工具注册
- `src/tools/GlobTool/GlobTool.ts` - GlobTool 复用 UI 组件
- `src/tools/AgentTool/built-in/*.ts` - Agent 工具集成
- `src/services/compact/*.ts` - 压缩服务
- `src/services/extractMemories/*.ts` - 记忆提取

## 依赖与外部交互

### 外部依赖

1. **ripgrep (rg)**：高性能正则表达式搜索工具
   - 支持三种运行模式：system（系统安装）、builtin（内置二进制）、embedded（Bun 内置）
   - 通过 `src/utils/ripgrep.ts` 封装调用

2. **zod/v4**：Schema 验证库
   - 用于输入/输出数据结构验证

### 权限系统集成

GrepTool 深度集成权限系统：

```
┌─────────────────┐
│   GrepTool.call │
└────────┬────────┘
         │
         ▼
┌─────────────────────────┐
│ checkReadPermissionForTool│
└────────┬────────────────┘
         │
    ┌────┴────┐
    ▼         ▼
┌───────┐  ┌─────────────┐
│deny规则│  │工作目录检查  │
└───────┘  └─────────────┘
```

权限检查流程：
1. 检查 UNC 路径（防御深度）
2. 检查可疑 Windows 路径模式
3. 检查 Read-specific deny 规则
4. 检查 Read-specific ask 规则
5. 检查 Edit 权限（Edit 权限隐含 Read 权限）
6. 检查工作目录边界
7. 检查内部可读路径
8. 检查 allow 规则

## 风险、边界与改进建议

### 已知风险

1. **超时风险**
   - WSL 环境下默认 60 秒超时，其他环境 20 秒
   - 大范围搜索可能超时，返回 `RipgrepTimeoutError`
   - 风险：模型可能误判为"无匹配"而非"搜索未完成"

2. **结果截断**
   - 默认 250 条结果限制，防止上下文膨胀
   - 需要配合 `offset` 参数实现分页
   - 风险：用户可能遗漏重要匹配

3. **行长度限制**
   - `--max-columns 500` 截断长行
   - 风险：base64/压缩内容可能显示不完整

4. **权限绕过风险**
   - 已针对 UNC 路径、Windows 短文件名、路径遍历等进行防护
   - 通过 `checkReadPermissionForTool` 进行多层检查

### 边界情况

1. **空结果处理**
   - ripgrep 退出码 1 表示"无匹配"，视为正常空结果
   - 与真正的执行错误（退出码 2+）区分处理

2. **文件并发修改**
   - `files_with_matches` 模式下，文件可能在 ripgrep 扫描和 stat 之间被删除
   - 使用 `Promise.allSettled` 处理，失败时按 mtime 0 排序

3. **测试环境**
   - `NODE_ENV === 'test'` 时按文件名排序，确保结果确定性

### 改进建议

1. **性能优化**
   - 考虑对频繁搜索的路径实现缓存机制
   - 对于大仓库，考虑使用 ripgrep 的 `--max-count` 提前终止

2. **用户体验**
   - 在结果截断时提供更明显的提示
   - 考虑添加搜索结果预览功能

3. **安全性增强**
   - 定期审计权限检查逻辑
   - 考虑添加搜索范围警告（如搜索根目录）

4. **可观测性**
   - 添加搜索性能指标收集
   - 记录常见搜索模式用于优化建议
