# 研究文档：src/utils/undercover.ts

## 场景与职责

本模块实现 **Undercover Mode（卧底模式）**，是 Anthropic 内部员工（`USER_TYPE === 'ant'`）向公共/开源仓库贡献代码时的安全护栏。其核心目标是：防止模型在生成的 commit message、PR 标题/描述中泄露 Anthropic 内部信息，包括但不限于：

- 内部模型代号（动物名，如 Capybara、Tengu 等）；
- 未发布的模型版本号；
- 内部项目名、Slack 频道、短链接（如 `go/cc`）；
- 任何暗示“我是 AI”或“我是 Claude Code”的表述；
- `Co-Authored-By` 等归因信息。

模块通过向模型追加严格的系统级指令，并要求其以普通人类开发者的口吻撰写提交信息，来实现“卧底”效果。

## 功能点目的

| 导出符号 | 目的 |
|---------|------|
| `isUndercover()` | 判断当前是否处于卧底模式。 |
| `getUndercoverInstructions()` | 返回注入到模型 prompt 中的完整卧底指令文本。 |
| `shouldShowUndercoverAutoNotice()` | 判断是否需要向用户展示一次性的自动卧底模式说明弹窗。 |

## 具体技术实现

### 1. 模式激活逻辑

`isUndercover()` 的行为：

- **外部构建**（`USER_TYPE !== 'ant'`）：始终返回 `false`。由于 `USER_TYPE` 是构建时 `--define`，bundler 会死码消除（dead-code eliminate）所有 `ant` 分支，外部二进制中该函数退化为 trivial return。
- **内部构建**（`USER_TYPE === 'ant'`）：
  - 若环境变量 `CLAUDE_CODE_UNDERCOVER=1`，**强制开启**（即使当前仓库被识别为内部仓库）。
  - 否则进入 **Auto 模式**：仅当 `getRepoClassCached() === 'internal'` 时关闭；其他情况（`external`、`none`、尚未检测）均**默认开启**。

> 设计原则：**没有 force-OFF 开关**。只要系统不确定当前仓库是内部仓库，就保持 undercover，以最大化防止信息泄露。

### 2. 仓库分类缓存

- `getRepoClassCached()` 来自 `src/utils/commitAttribution.ts`。
- 该函数在 `setup.ts` 启动阶段预计算，结果缓存在模块级变量中。
- 分类基于远程仓库 URL 是否匹配 `INTERNAL_MODEL_REPOS` allowlist。

### 3. 指令文本

`getUndercoverInstructions()` 返回一段硬编码的 Markdown 格式系统指令，包含：

- 明确的 `## UNDERCOVER MODE — CRITICAL` 标题；
- `NEVER include` 列表（禁止项）；
- `GOOD` / `BAD` 示例对比；
- 要求模型“像普通人类开发者一样写 commit message”。

该指令被注入到与 commit/PR 相关的 prompt 中（见调用方列表）。

### 4. 自动提示弹窗

`shouldShowUndercoverAutoNotice()` 控制一次性弹窗：

- 仅对 `ant` 用户生效。
- 若用户通过环境变量强制开启（`CLAUDE_CODE_UNDERCOVER=1`），不弹窗（假设用户已知情）。
- 若当前不在 undercover 模式，不弹窗。
- 若用户之前已看过（`getGlobalConfig().hasSeenUndercoverAutoNotice === true`），不弹窗。
- 否则返回 `true`，由 UI 组件在 mount 时展示并将标志位持久化到全局配置。

## 关键代码路径与文件引用

- **主实现**：`src/utils/undercover.ts`（89 行）
- **仓库分类**：`src/utils/commitAttribution.ts`（`getRepoClassCached`、`INTERNAL_MODEL_REPOS`）
- **全局配置**：`src/utils/config.ts`（`getGlobalConfig`、`hasSeenUndercoverAutoNotice`）
- **调用方（commit 命令）**：`src/commands/commit.ts`
- **调用方（commit-push-pr 命令）**：`src/commands/commit-push-pr.ts`
- **调用方（Bash 提示注入）**：`src/tools/BashTool/prompt.ts`
- **调用方（归因系统）**：`src/utils/attribution.ts`
- **调用方（系统提示常量）**：`src/constants/prompts.ts`
- **调用方（UI 提示）**：`src/components/PromptInput/PromptInputFooter.tsx`

## 依赖与外部交互

- **`src/utils/commitAttribution.js`**：`getRepoClassCached`。
- **`src/utils/config.js`**：`getGlobalConfig`。
- **`src/utils/envUtils.js`**：`isEnvTruthy`。
- 无外部 npm 依赖。

## 风险、边界与改进建议

### 风险

1. **仓库分类缓存的时滞性**：`getRepoClassCached()` 在进程启动时计算一次。若用户在会话中途 `cd` 到另一个项目、或通过 `git remote set-url` 切换 remote，分类结果不会更新，可能导致在公共仓库中意外关闭 undercover，或在内部仓库中不必要地开启。
2. **指令文本的绕过可能性**：模型可能因指令冲突（如用户明确要求“在 commit message 中提及你是 Claude”）而部分忽略 undercover 指令。虽然系统提示权重较高，但无法 100% 保证模型不会泄露。
3. **硬编码敏感词列表的维护成本**：指令中的示例（如 `capybara`、`tengu`、`claude-opus-4-6`）是静态写死的。随着新模型代号的出现，需要持续更新源码并重新发布构建。

### 边界

- **仅影响 commit/PR 相关 prompt**：`getUndercoverInstructions()` 只在调用方显式注入时生效。普通的对话、代码编辑、文件读取等操作不受该模式影响。
- **无运行时 force-OFF**：这是安全设计的一部分，但也意味着内部员工在特殊场景下（如明确的内部私有子目录）无法便捷地临时关闭 undercover。
- **构建时 DCE 依赖**：外部构建的安全性完全依赖 bundler 正确执行死码消除。若构建配置变更导致 DCE 失效，内部指令文本可能意外泄露到外部二进制中。

### 改进建议

1. **动态仓库重检测**：在 `getUndercoverInstructions()` 或 `isUndercover()` 中增加一个可选的“强制刷新”路径，当检测到 CWD 或 git remote 发生变化时重新调用 `getRepoClassCached` 的底层逻辑。
2. **敏感词列表外置化**：将禁止提及的模型代号、项目名称、短链接模式迁移到一个内部配置文件中（如 `.claude/undercover-denylist.json`），通过 GrowthBook 或本地配置热更新，减少每次新增代号都要发版的频率。
3. **增加后处理过滤器**：除了向模型注入指令外，还可在 commit/PR 文本生成后增加一层正则/LLM 后处理扫描，自动检测并高亮潜在泄露内容，给用户一个二次确认的机会。
4. **更细粒度的触发条件**：当前 undercover 是全局开关。可考虑根据目标 remote 的可见性（public vs private GitHub repo）动态调整指令强度，而非简单的 on/off。
