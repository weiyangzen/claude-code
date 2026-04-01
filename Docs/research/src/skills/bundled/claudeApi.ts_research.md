# claudeApi.ts 研究文档

## 场景与职责

`claudeApi.ts` 实现了 `/claude-api` 内置技能，用于在用户请求与 Claude API、Anthropic SDK 或 Agent SDK 相关的问题时，动态注入对应语言的官方文档片段作为系统提示。该技能通过检测项目使用的编程语言，按需内联相关 Markdown 文档，帮助模型给出准确、最新的 API 使用建议。

## 功能点目的

1. **语言自动检测**：扫描当前工作目录的文件特征（扩展名、配置文件），判断项目主要语言（Python/TypeScript/Java/Go/Ruby/C#/PHP/cURL）。
2. **文档按需内联**：根据检测到的语言，从 `claudeApiContent.ts` 中筛选对应语言的文档和共享文档，拼接进系统提示。
3. **变量模板替换**：将 `{{VAR}}` 占位符替换为运行时模型 ID（如 `claude-sonnet-4-6`）。
4. **阅读指南生成**：提供按任务类型（单文本分类、流式、工具使用、批处理、Prompt Caching 等）快速定位文档的指南。

## 具体技术实现

### 关键流程

- `registerClaudeApiSkill()` → `registerBundledSkill({ name: 'claude-api', ... })`
- `getPromptForCommand(args)` 入口：
  1. 动态 `import('./claudeApiContent.js')` 实现懒加载（247KB 的 Markdown 字符串仅在调用时进入内存）
  2. `detectLanguage()` 扫描 `cwd` 目录条目
  3. `buildPrompt(lang, args, content)` 组装 prompt

### 数据结构

```ts
type DetectedLanguage =
  | 'python' | 'typescript' | 'java' | 'go'
  | 'ruby' | 'csharp' | 'php' | 'curl'

const LANGUAGE_INDICATORS: Record<DetectedLanguage, string[]> = {
  python: ['.py', 'requirements.txt', 'pyproject.toml', 'setup.py', 'Pipfile'],
  typescript: ['.ts', '.tsx', 'tsconfig.json', 'package.json'],
  java: ['.java', 'pom.xml', 'build.gradle'],
  go: ['.go', 'go.mod'],
  ruby: ['.rb', 'Gemfile'],
  csharp: ['.cs', '.csproj'],
  php: ['.php', 'composer.json'],
  curl: [],
}
```

### 文档处理函数

- `processContent(md, content)`：
  1. 循环去除 HTML 注释 `<!-- ... -->`
  2. 将 `{{(\w+)}}` 替换为 `SKILL_MODEL_VARS` 中对应值
- `buildInlineReference(filePaths, content)`：将文档包装为 `<doc path="...">...</doc>` 的 XML 片段，方便模型引用来源路径。

### Prompt 结构

1. `SKILL_PROMPT` 基础内容（截断至 `## Reading Guide` 之前）
2. `INLINE_READING_GUIDE`（按任务类型的快速索引）
3. `## Included Documentation`（内联的 `<doc>` 片段）
4. `## When to Use WebFetch` 与 `## Common Pitfalls`（保留原 prompt 尾部）
5. `## User Request`（用户输入）

## 关键代码路径与文件引用

- 源文件：`src/skills/bundled/claudeApi.ts`
- 文档内容文件：`src/skills/bundled/claudeApiContent.ts`
- 注册入口：`src/skills/bundled/index.ts`
- 核心注册器：`src/skills/bundledSkills.ts`
- 依赖的 cwd 工具：`src/utils/cwd.ts`（`getCwd`）
- 实际 Markdown 文档目录：`src/skills/bundled/claude-api/`（各语言子目录 + `shared/`）

## 依赖与外部交互

| 依赖 | 作用 |
|------|------|
| `claudeApiContent.ts` | 提供所有内联的 Markdown 文档字符串和模型变量 |
| `getCwd()` | 获取当前工作目录用于语言检测 |
| `fs/promises.readdir` | 扫描目录条目 |
| `registerBundledSkill` | 注册技能 |

- **无外部网络调用**：所有文档在构建时通过 Bun 的 text loader 内联进二进制。
- **懒加载设计**：`claudeApiContent.js` 在 `getPromptForCommand` 内部动态 import，避免启动时加载 247KB 文档字符串。

## 风险、边界与改进建议

1. **边界：语言检测的歧义性**：
   - `package.json` 同时出现在 `typescript` 的 indicators 中，若一个仓库同时包含 `.py` 与 `package.json`，按遍历顺序 `python` 先被检测（因为 `Object.entries` 顺序），可能导致误判。
   - 建议：为常见多语言仓库增加权重或更细粒度检测（如同时存在 `go.mod` 与 `package.json` 时如何决策）。
2. **边界：curl 语言无检测指标**：`curl` 永远不会被自动检测，只能走 "未检测到语言" 的全文档 fallback 路径。
3. **风险：模型变量与硬编码不同步**：注释明确提到更新 `SKILL_MODEL_VARS` 后，仍需手动更新 `claude-api/SKILL.md` 和 `shared/models.md` 中的定价表，存在遗漏风险。
4. **改进建议**：
   - 将 `LANGUAGE_INDICATORS` 的检测逻辑改为按文件数量/权重投票，而非第一个匹配即返回。
   - 考虑在 prompt 中显式声明文档版本日期，帮助用户判断建议是否可能过时。
   - 增加对 `bun`/`deno` 等新兴运行时配置文件的检测支持。
