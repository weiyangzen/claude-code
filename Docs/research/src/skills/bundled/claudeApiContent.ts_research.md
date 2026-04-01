# claudeApiContent.ts 研究文档

## 场景与职责

`claudeApiContent.ts` 是 `/claude-api` 技能的**内容资产文件**，负责将所有语言相关的 Markdown 文档在构建时通过 Bun 的 text loader 内联为 TypeScript 字符串常量。它在运行时被 `claudeApi.ts` 懒加载，用于按需拼接系统提示。该文件本身不包含业务逻辑，是技能的数据层。

## 功能点目的

1. **构建时文档内联**：利用 Bun 的 `*.md` text loader，将 `claude-api/` 目录下的所有 Markdown 文件编译为 JS 字符串。
2. **运行时变量替换**：定义 `SKILL_MODEL_VARS`，为文档中的 `{{VAR}}` 占位符提供运行时替换值。
3. **统一内容导出**：以 `SKILL_PROMPT` 和 `SKILL_FILES` 两个导出对象，向 `claudeApi.ts` 提供标准化访问接口。

## 具体技术实现

### 关键数据结构

```ts
export const SKILL_MODEL_VARS = {
  OPUS_ID: 'claude-opus-4-6',
  OPUS_NAME: 'Claude Opus 4.6',
  SONNET_ID: 'claude-sonnet-4-6',
  SONNET_NAME: 'Claude Sonnet 4.6',
  HAIKU_ID: 'claude-haiku-4-5',
  HAIKU_NAME: 'Claude Haiku 4.5',
  PREV_SONNET_ID: 'claude-sonnet-4-5',
} satisfies Record<string, string>

export const SKILL_PROMPT: string = skillPrompt

export const SKILL_FILES: Record<string, string> = {
  'csharp/claude-api.md': csharpClaudeApi,
  'curl/examples.md': curlExamples,
  'go/claude-api.md': goClaudeApi,
  'java/claude-api.md': javaClaudeApi,
  'php/claude-api.md': phpClaudeApi,
  'python/agent-sdk/README.md': pythonAgentSdkReadme,
  'python/agent-sdk/patterns.md': pythonAgentSdkPatterns,
  'python/claude-api/README.md': pythonClaudeApiReadme,
  'python/claude-api/batches.md': pythonClaudeApiBatches,
  'python/claude-api/files-api.md': pythonClaudeApiFilesApi,
  'python/claude-api/streaming.md': pythonClaudeApiStreaming,
  'python/claude-api/tool-use.md': pythonClaudeApiToolUse,
  'ruby/claude-api.md': rubyClaudeApi,
  'shared/error-codes.md': sharedErrorCodes,
  'shared/live-sources.md': sharedLiveSources,
  'shared/models.md': sharedModels,
  'shared/prompt-caching.md': sharedPromptCaching,
  'shared/tool-use-concepts.md': sharedToolUseConcepts,
  'typescript/agent-sdk/README.md': typescriptAgentSdkReadme,
  'typescript/agent-sdk/patterns.md': typescriptAgentSdkPatterns,
  'typescript/claude-api/README.md': typescriptClaudeApiReadme,
  'typescript/claude-api/batches.md': typescriptClaudeApiBatches,
  'typescript/claude-api/files-api.md': typescriptClaudeApiFilesApi,
  'typescript/claude-api/streaming.md': typescriptClaudeApiStreaming,
  'typescript/claude-api/tool-use.md': typescriptClaudeApiToolUse,
}
```

### 导入的文档清单

- 主 Prompt：`claude-api/SKILL.md`
- 共享文档：`shared/error-codes.md`、`shared/live-sources.md`、`shared/models.md`、`shared/prompt-caching.md`、`shared/tool-use-concepts.md`
- 语言文档：C#、cURL、Go、Java、PHP、Python、Ruby、TypeScript 各语言的 README、streaming、batches、files-api、tool-use、agent-sdk 等

## 关键代码路径与文件引用

- 源文件：`src/skills/bundled/claudeApiContent.ts`
- 引用方：`src/skills/bundled/claudeApi.ts`
- 文档源目录：`src/skills/bundled/claude-api/`
- 构建工具：Bun 的 text loader（`*.md` 文件在 import 时作为字符串内联）

## 依赖与外部交互

- **纯静态数据文件**：除 Bun 构建时的 text loader 外，无运行时依赖。
- **无网络交互**：所有内容在编译期确定。
- **调用关系**：仅被 `claudeApi.ts` 通过动态 `import('./claudeApiContent.js')` 引用。

## 风险、边界与改进建议

1. **边界：文件大小**：注释说明该文件 bundle 后约 247KB，虽然采用懒加载，但仍会增加最终二进制体积。若文档持续扩充，需考虑按语言拆分为多个 chunk。
2. **风险：模型 ID 同步遗漏**：文件顶部注释明确提醒更新 `SKILL_MODEL_VARS` 后，还要手动更新 `SKILL.md` 和 `shared/models.md` 中的硬编码表格。此过程无自动化校验，容易遗漏。
3. **风险：文档版本漂移**：`shared/live-sources.md` 提供了官方文档的 WebFetch URL，但内置的 Markdown 文档版本取决于构建时的快照，可能滞后于线上最新版本。
4. **改进建议**：
   - 引入构建时脚本，自动扫描 `claude-api/` 目录生成 `SKILL_FILES` 映射，减少新增文档时的人工编辑。
   - 在 CI 中增加校验：检查 `SKILL_MODEL_VARS` 中的 ID 是否与 `SKILL.md`/`models.md` 中的表格一致。
   - 考虑将大型共享文档（如 `models.md`）按语言拆分，进一步降低首次加载的内存占用。
