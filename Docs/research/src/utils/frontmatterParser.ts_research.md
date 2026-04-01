# 研究文档：src/utils/frontmatterParser.ts

## 场景与职责

`frontmatterParser.ts` 是 Claude Code 中所有 Markdown 配置文件（commands、skills、agents、CLAUDE.md、output-styles 等）的 **YAML frontmatter 解析器**。它不仅负责提取 `---` 包裹的元数据，还提供了大量针对 frontmatter 场景的专用辅助函数，如 brace expansion（大括号展开）、布尔值/整数值/描述字段的强制类型转换、shell 字段校验等。

该模块是配置加载 pipeline 的核心前置步骤，直接影响技能发现、权限控制、模型选择、hook 注册等高级功能。

## 功能点目的

| 导出项 | 目的 |
|--------|------|
| `parseFrontmatter(markdown, sourcePath?)` | 解析 markdown 中的 YAML frontmatter，失败时尝试自动引号修复。 |
| `FRONTMATTER_REGEX` | 匹配 `---\n...\n---\n` 的正则表达式。 |
| `FrontmatterData` | frontmatter 字段的类型定义（允许 `null` 和多种类型）。 |
| `ParsedMarkdown` | `{ frontmatter, content }` 返回类型。 |
| `splitPathInFrontmatter(input)` | 解析 frontmatter 中的路径字段，支持逗号分隔和 brace expansion。 |
| `parsePositiveIntFromFrontmatter(value)` | 从 frontmatter 中解析正整数。 |
| `coerceDescriptionToString(value, ...)` | 将 description 强制转换为字符串，过滤非法类型并记录警告。 |
| `parseBooleanFrontmatter(value)` | 仅当值为 literal `true` 或字符串 `"true"` 时返回 `true`。 |
| `parseShellFrontmatter(value, source)` | 解析并校验 `shell:` 字段，仅接受 `bash` 或 `powershell`。 |
| `FrontmatterShell` | `'bash' \| 'powershell'` 类型。 |

## 具体技术实现

### Frontmatter 解析流程

```ts
export function parseFrontmatter(markdown: string, sourcePath?: string): ParsedMarkdown {
  const match = markdown.match(FRONTMATTER_REGEX)
  if (!match) return { frontmatter: {}, content: markdown }

  const frontmatterText = match[1] || ''
  const content = markdown.slice(match[0].length)

  let frontmatter: FrontmatterData = {}
  try {
    const parsed = parseYaml(frontmatterText) as FrontmatterData | null
    if (parsed && typeof parsed === 'object' && !Array.isArray(parsed)) {
      frontmatter = parsed
    }
  } catch {
    // 第一次失败 → 尝试 quoteProblematicValues 修复
    try {
      const quotedText = quoteProblematicValues(frontmatterText)
      const parsed = parseYaml(quotedText) as FrontmatterData | null
      if (parsed && typeof parsed === 'object' && !Array.isArray(parsed)) {
        frontmatter = parsed
      }
    } catch (retryError) {
      logForDebugging(`Failed to parse YAML frontmatter...`, { level: 'warn' })
    }
  }

  return { frontmatter, content }
}
```

### 自动引号修复（`quoteProblematicValues`）

由于用户常在 frontmatter 中写 glob 模式（如 `src/*.{ts,tsx}`），这些模式包含 YAML 特殊字符（`{`, `}`, `*`, `: ` 等），会导致标准 YAML 解析失败。模块实现了轻量修复：

```ts
const YAML_SPECIAL_CHARS = /[{}[\]*&#!|>%@`]|: /
```

- 逐行扫描 `key: value` 格式。
- 跳过已带引号的值。
- 若值匹配 `YAML_SPECIAL_CHARS`，则用双引号包裹，并转义内部的双引号。

### 路径字段解析（`splitPathInFrontmatter`）

支持两种输入：
- 逗号分隔的字符串
- YAML 字符串数组

解析时会：
1. 按逗号分割，但**尊重大括号层级**（逗号在 `{...}` 内不视为分隔符）。
2. 对每个片段调用 `expandBraces` 递归展开大括号模式。

示例：
- `"a, src/*.{ts,tsx}"` → `["a", "src/*.ts", "src/*.tsx"]`
- `"{a,b}/{c,d}"` → `["a/c", "a/d", "b/c", "b/d"]`

### 专用字段解析函数

| 函数 | 行为 |
|------|------|
| `parsePositiveIntFromFrontmatter` | 接受 `number` 或 `string`，仅返回正整数，否则 `undefined`。 |
| `coerceDescriptionToString` | `string` 直接返回（trim）；`number`/`boolean` 强制转字符串；`array`/`object` 记录警告并返回 `null`。 |
| `parseBooleanFrontmatter` | 严格匹配 `true` 或 `"true"`，其他一律 `false`。 |
| `parseShellFrontmatter` | 仅接受 `bash`/`powershell`（大小写不敏感），其他值记录警告并返回 `undefined`（调用方默认 fallback 到 bash）。 |

## 关键代码路径与文件引用

### 调用方

| 文件 | 导入内容 | 说明 |
|------|----------|------|
| `src/utils/markdownConfigLoader.ts:17` | `parseFrontmatter`, `FrontmatterData` | 加载 commands/agents/skills/output-styles 等 markdown 配置。 |
| `src/utils/claudemd.ts:66` | `parseFrontmatter`, `splitPathInFrontmatter` | CLAUDE.md 和 memory 文件的 frontmatter 解析与路径匹配。 |
| `src/skills/loadSkillsDir.ts` | `FrontmatterData` 等 | 技能目录扫描与 frontmatter 校验。 |
| `src/memdir/memoryScan.ts` | `parseFrontmatter` | 内存文件扫描。 |
| `src/tools/AgentTool/loadAgentsDir.ts` | `parseFrontmatter` | Agent 定义加载。 |
| `src/tools/SkillTool/SkillTool.ts` | `FrontmatterData` | Skill 工具执行。 |
| `src/utils/promptShellExecution.ts` | `FrontmatterShell` (type) | 执行 `!` 代码块时选择 shell。 |
| `src/commands/security-review.ts` | `parseFrontmatter` | 安全审查命令配置。 |
| `src/utils/plugins/loadPluginCommands.ts` | `parseFrontmatter` | 插件命令加载。 |
| `src/utils/plugins/validatePlugin.ts` | `parseFrontmatter` | 插件校验。 |
| `src/utils/plugins/loadPluginAgents.ts` | `parseFrontmatter` | 插件 Agent 加载。 |
| `src/utils/plugins/loadPluginOutputStyles.ts` | `parseFrontmatter` | 插件输出样式加载。 |

### 被调用方

- `src/utils/debug.js`：`logForDebugging`
- `src/utils/yaml.js`：`parseYaml`
- `src/utils/settings/types.js`：`HooksSettings` (type)

## 依赖与外部交互

- 无网络依赖。
- 无持久化。
- 核心依赖内部的 `parseYaml`（`src/utils/yaml.js`），该模块可能是 `yaml` 第三方包的薄封装或自定义实现。

## 风险、边界与改进建议

### 风险

1. **引号修复的过度匹配**：`YAML_SPECIAL_CHARS` 中的 `: ` 模式虽然能避免大部分 key-value 混淆，但也可能误伤合法值（如 `"note: this is a value"` 会被引号包裹，这其实是正确的；但 `"https://example.com"` 不含 `: ` 所以安全）。
2. **brace expansion 的递归深度**：`expandBraces` 使用正则 `^([^{]*)\{([^}]+)\}(.*)$` 递归展开，若用户输入极度嵌套的大括号（如 `{{{{a,b}}}}`），可能导致栈溢出或性能骤降。
3. **YAML 解析失败的静默吞掉**：两次解析都失败后，仅通过 `logForDebugging` 输出 warn 级别日志，调用方得到的是空 frontmatter，可能导致技能/命令的关键配置（如 `allowed-tools`）被忽略。
4. **类型宽泛**：`FrontmatterData` 中大量字段是 `string | string[] | null`，调用方需要在多处做运行时类型守卫，增加了出错概率。

### 边界

- `FRONTMATTER_REGEX` 要求 frontmatter 必须严格以 `---` 开头并以 `---` 结束，不支持 `...` 结束符或 `+++` TOML frontmatter。
- `quoteProblematicValues` 只处理简单的一级 `key: value` 行，不处理嵌套对象或 block scalar（如 `|`、`>`）。
- `splitPathInFrontmatter` 只做字符串层面的 brace expansion，不做真正的文件系统 glob 解析。
- `parseBooleanFrontmatter` 是严格模式，不接受 `"yes"`、`"1"`、 `"on"` 等常见真值。

### 改进建议

1. **更健壮的 YAML 修复**：考虑使用专门的 YAML tolerant parser（如 `yaml` 包的 `JSON_SCHEMA` 或自定义 lexer），而非基于正则的引号修复，减少误匹配。
2. **brace expansion 深度限制**：在 `expandBraces` 中增加递归深度计数器，超过阈值（如 10）时抛出可读错误。
3. **解析失败显式报错**：将 frontmatter 解析失败从 warn 提升为 error，并向用户展示具体文件和行号，避免配置被静默忽略。
4. **FrontmatterData 细化**：对关键字段（如 `model`、`shell`、`context`、`effort`）使用更窄的联合类型或 branded type，在编译期捕获更多错误。
5. **测试覆盖**：补充对复杂嵌套 braces、非法 YAML、空 frontmatter、以及 `coerceDescriptionToString` 各种输入类型的单元测试。
