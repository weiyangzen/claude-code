# 研究文档: src/utils/plugins/validatePlugin.ts

## 1. 场景与职责

`validatePlugin.ts` 是 Claude Code 插件系统的验证引擎，负责对插件相关的各类配置文件进行全面的静态分析和验证。它是 `claude plugin validate` CLI 命令的核心实现，也是插件加载前的重要防线。

### 核心职责
- **文件验证**: 验证 plugin.json、marketplace.json、hooks.json 的结构合法性
- **内容验证**: 验证 skill、agent、command 等 Markdown 文件的 frontmatter 格式
- **安全检查**: 检测路径遍历攻击、重复插件名、版本不匹配等问题
- **开发者反馈**: 提供详细的错误和警告信息，帮助插件作者修正问题

### 验证范围
| 文件类型 | 验证内容 |
|---------|---------|
| plugin.json | Schema 合规性、kebab-case 命名、路径安全、字段建议 |
| marketplace.json | Schema 合规性、插件源路径、重复名称检测、版本一致性 |
| hooks.json | Schema 合规性（硬错误，影响插件加载） |
| .md 文件 (skill/agent/command) | YAML frontmatter 语法、字段类型、必需字段 |

## 2. 功能点目的

### 2.1 Plugin Manifest 验证 (`validatePluginManifest`)

**目的**: 确保 plugin.json 符合 PluginManifestSchema 规范。

**验证流程**:
1. **文件读取**: 处理 ENOENT、EISDIR、权限错误
2. **JSON 解析**: 捕获语法错误
3. **路径遍历预检**: 在 Schema 验证前检查 `commands`、`agents`、`skills` 字段中的路径
4. **字段剥离**: 移除属于 marketplace-only 的字段（`category`、`source`、`tags`、`strict`、`id`），避免 Schema 严格模式报错
5. **Schema 验证**: 使用 `.strict()` 模式捕获未知字段（开发者工具需要严格反馈）
6. **后验证检查**:
   - 名称是否为 kebab-case（Claude.ai 市场同步要求）
   - 是否缺少 version、description、author 字段（警告级别）

**关键设计**: 运行时加载器对未知字段宽松处理（剥离），但验证工具严格处理（报错），平衡了兼容性和开发者体验。

### 2.2 Marketplace Manifest 验证 (`validateMarketplaceManifest`)

**目的**: 确保 marketplace.json 符合 PluginMarketplaceSchema 规范。

**验证流程**:
1. **文件读取**: 同 plugin.json
2. **JSON 解析**: 同 plugin.json
3. **路径遍历预检**: 
   - 检查 `plugins[].source` 字符串路径
   - 检查 `plugins[].source.path`（git-subdir 类型）
   - 提供针对性提示：说明路径解析相对于市场根目录而非 marketplace.json
4. **Schema 验证**: 外层和 plugins 数组元素均使用 `.strict()`
5. **后验证检查**:
   - 空插件列表警告
   - 重复插件名检测
   - **版本一致性检查**: 对比本地源插件的 entry.version 与 plugin.json 中的 version

**版本不匹配警告**:
```typescript
if (manifestVersion && manifestVersion !== entry.version) {
  warnings.push({
    path: `plugins[${i}].version`,
    message: `Entry declares version "${entry.version}" but ${entry.source}/.claude-plugin/plugin.json says "${manifestVersion}"...`
  })
}
```

### 2.3 组件文件验证 (`validateComponentFile`)

**目的**: 验证 Markdown 文件的 YAML frontmatter。

**验证内容**:
- **Frontmatter 存在性**: 无 frontmatter 时发出警告（建议添加）
- **YAML 语法**: 解析失败时硬错误
- **数据类型**: 必须是对象映射（非数组、非 null）
- **字段验证**:
  - `description`: 必须是标量（字符串/数字/布尔/null），数组/对象非法
  - `name`: 如果存在必须是字符串
  - `allowed-tools`: 字符串或字符串数组
  - `shell`: 必须是 'bash' 或 'powershell'（大小写不敏感）

**运行时差异**: 运行时加载器对 frontmatter 错误宽松处理（静默忽略），验证工具严格处理（报错）。

### 2.4 Hooks 验证 (`validateHooksJson`)

**目的**: 验证 hooks/hooks.json 的合法性。

**重要性**: hooks.json 在运行时使用 `.parse()` 而非 `.safeParse()`，解析失败会导致整个插件加载失败。因此验证工具必须严格检查。

**处理 ENOENT**: hooks 是可选的，文件不存在时返回成功。

### 2.5 插件内容批量验证 (`validatePluginContents`)

**目的**: 扫描插件目录中的所有组件文件并验证。

**扫描范围**:
- `skills/`: 查找 `<name>/SKILL.md` 格式（仅一级子目录）
- `agents/`: 递归查找所有 `.md` 文件
- `commands/`: 递归查找所有 `.md` 文件
- `hooks/hooks.json`: 验证 hooks 配置

**错误处理**: 单个文件错误不影响其他文件验证。

### 2.6 自动类型检测 (`validateManifest`)

**目的**: 根据文件路径或内容自动判断验证类型。

**检测逻辑**:
1. 显式文件名: `plugin.json` → plugin, `marketplace.json` → marketplace
2. 目录位置: `.claude-plugin/` 目录内 → 可能是 plugin
3. 内容启发式: 包含 `plugins` 数组 → marketplace
4. 默认回退: plugin

## 3. 具体技术实现

### 3.1 路径遍历检测

```typescript
function checkPathTraversal(
  p: string,
  field: string,
  errors: ValidationError[],
  hint?: string,
): void {
  if (p.includes('..')) {
    errors.push({
      path: field,
      message: hint
        ? `Path contains "..": ${p}. ${hint}`
        : `Path contains ".." which could be a path traversal attempt: ${p}`,
    })
  }
}
```

**市场源路径提示生成**:
```typescript
function marketplaceSourceHint(p: string): string {
  const stripped = p.replace(/^(\.\.\/)+/, '')
  const corrected = stripped !== p ? `./${stripped}` : './plugins/my-plugin'
  return (
    'Plugin source paths are resolved relative to the marketplace root... ' +
    `Use "${corrected}" instead of "${p}".`
  )
}
```

### 3.2 Zod 错误格式化

```typescript
function formatZodErrors(zodError: z.ZodError): ValidationError[] {
  return zodError.issues.map(error => ({
    path: error.path.join('.') || 'root',
    message: error.message,
    code: error.code,
  }))
}
```

### 3.3 Markdown 文件收集

```typescript
async function collectMarkdown(
  dir: string,
  isSkillsDir: boolean,
): Promise<string[]> {
  // Skills: 仅一级子目录，查找 SKILL.md
  if (isSkillsDir) {
    return entries
      .filter(e => e.isDirectory())
      .map(e => path.join(dir, e.name, 'SKILL.md'))
  }
  // Commands/Agents: 递归查找所有 .md
  // ...
}
```

### 3.4 类型定义

```typescript
export type ValidationResult = {
  success: boolean
  errors: ValidationError[]
  warnings: ValidationWarning[]
  filePath: string
  fileType: 'plugin' | 'marketplace' | 'skill' | 'agent' | 'command' | 'hooks'
}

export type ValidationError = {
  path: string      // 字段路径，如 "commands[0]" 或 "root"
  message: string   // 人类可读的错误描述
  code?: string     // 错误代码（如 Zod 错误码）
}

export type ValidationWarning = {
  path: string
  message: string
}
```

## 4. 关键代码路径与文件引用

### 4.1 核心依赖

| 依赖文件 | 用途 |
|---------|------|
| `src/utils/plugins/schemas.ts` | Schema 定义（PluginManifestSchema, PluginMarketplaceSchema 等） |
| `src/utils/errors.ts` | 错误处理工具（errorMessage, getErrnoCode, isENOENT） |
| `src/utils/frontmatterParser.ts` | Frontmatter 解析（FRONTMATTER_REGEX） |
| `src/utils/slowOperations.ts` | JSON 解析（jsonParse） |
| `src/utils/yaml.ts` | YAML 解析（parseYaml） |

### 4.2 被调用方

| 调用方 | 用途 |
|-------|------|
| `src/cli/handlers/plugins.ts` | `claude plugin validate` 命令实现 |
| `src/commands/plugin/ValidatePlugin.tsx` | 交互式验证 UI |

### 4.3 调用关系图

```
validateManifest
├── validatePluginManifest
│   └── checkPathTraversal (commands, agents, skills)
│   └── PluginManifestSchema().strict().safeParse()
├── validateMarketplaceManifest
│   └── checkPathTraversal (plugin sources)
│   └── PluginMarketplaceSchema().strict().safeParse()
│   └── 版本一致性检查 (对比 plugin.json)
└── validatePluginContents
    ├── collectMarkdown (skills/agents/commands)
    ├── validateComponentFile (每个 .md 文件)
    │   └── parseYaml (frontmatter)
    └── validateHooksJson
        └── PluginHooksSchema().safeParse()
```

## 5. 依赖与外部交互

### 5.1 输入依赖

1. **文件系统**: 读取待验证的配置文件
2. **schemas.ts**: Schema 定义用于验证
3. **frontmatterParser.ts**: Frontmatter 提取逻辑
4. **yaml.ts**: YAML 解析
5. **slowOperations.ts**: 安全 JSON 解析

### 5.2 输出消费

1. **CLI 处理器**: 格式化验证结果输出给用户
2. **IDE 集成**: 可能的 LSP 诊断信息来源
3. **CI/CD**: 插件发布前的自动化检查

## 6. 风险、边界与改进建议

### 6.1 风险分析

| 风险 | 描述 | 缓解措施 |
|-----|------|---------|
| 验证-运行时不一致 | 验证工具使用 `.strict()`，运行时使用默认模式（宽松） | 文档说明；marketplace-only 字段预剥离 |
| 路径遍历漏检 | 复杂的对象路径可能绕过 `..` 检测 | 预检覆盖常见字段，Schema 作为后备 |
| 版本检查性能 | 每个本地插件都读取 plugin.json | 仅检查显式声明版本的条目 |
| 符号链接遍历 | `collectMarkdown` 可能跟随符号链接 | 使用 `withFileTypes` 但未显式处理 symlink |

### 6.2 边界情况

1. **空插件目录**: `validatePluginContents` 返回空数组（无错误）
2. **混合大小写 SKILL.md**: 正则 `/^skill\.md$/i` 大小写不敏感匹配
3. ** dangling symlinks**: `collectMarkdown` 中 ENOENT 被静默忽略
4. **非 UTF-8 文件**: 依赖 Node.js 读取，可能产生乱码

### 6.3 改进建议

1. **增强验证**:
   - 增加对 `mcpServers` 和 `lspServers` 配置的深度验证
   - 验证插件依赖的循环引用
   - 检查插件名称与目录名的一致性

2. **性能优化**:
   - 对大型市场使用并行验证
   - 缓存已解析的 plugin.json 避免重复读取

3. **开发者体验**:
   - 提供自动修复建议（如自动转换名称到 kebab-case）
   - 增加 JSON Schema 导出供 IDE 使用
   - 验证报告导出为 JSON/SARIF 格式

4. **安全增强**:
   - 验证文件权限（避免世界可写配置文件）
   - 检查敏感信息泄露（如硬编码 token）

5. **代码重构**:
   - 将验证逻辑拆分为独立模块：
     - `validators/manifest.ts`
     - `validators/marketplace.ts`
     - `validators/component.ts`
     - `validators/hooks.ts`
