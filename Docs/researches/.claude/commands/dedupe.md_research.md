# FILE `.claude/commands/dedupe.md` 研究文档

## 场景与职责

`.claude/commands/dedupe.md` 定义仓库级 `/dedupe` 命令，用于“新建 Issue 后快速查重并回帖候选重复项”。它不直接实现查重算法，而是承担 Agent 编排职责：

- 约束可调用工具：仅 `./scripts/gh.sh` 与 `./scripts/comment-on-duplicates.sh`（`.claude/commands/dedupe.md:2`）
- 规定执行阶段：先判定是否跳过、再总结、再并行检索、再过滤、最后发表评论（`:10-17`）
- 明确输出动作：最多 3 个候选重复 issue，通过脚本统一回帖模板

调用场景主要来自 GitHub Actions：

- `.github/workflows/claude-dedupe-issues.yml:1-33`
- 触发事件：`issues.opened` + `workflow_dispatch`（`:3-11`）
- 触发 prompt：`/dedupe owner/repo/issues/NUMBER`（`:32`）

因此该命令是“issue 去重流水线”的提示层入口，而真正副作用由脚本和后续定时任务完成。

## 功能点目的

1. 降低重复 issue 对维护成本的影响
- 在 issue 打开早期给出候选重复项，减少后续人工分诊负担。

2. 建立自动关闭前的缓冲交互
- 评论模板包含 3 天窗口、反对方式（评论或 👎）说明，给提报者保留异议通道（`scripts/comment-on-duplicates.sh:92-95`）。

3. 维持低噪声输出
- 明确“最多 3 个候选重复 issue”（`.claude/commands/dedupe.md:6,16`；脚本也强校验最多 3 个，`scripts/comment-on-duplicates.sh:51-53`）。

4. 与后处理自动化形成闭环
- 去重评论会被 `scripts/auto-close-duplicates.ts` 检测并在条件满足时自动关闭 issue（`.github/workflows/auto-close-duplicates.yml:1-27`，`scripts/auto-close-duplicates.ts:164-271`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 前置权限与命令协议

frontmatter：

```yaml
allowed-tools: Bash(./scripts/gh.sh:*), Bash(./scripts/comment-on-duplicates.sh:*)
description: Find duplicate GitHub issues
```

这意味着命令执行面被限制在两类 shell 脚本：

- 只读访问与检索：`scripts/gh.sh`
- 写评论副作用：`scripts/comment-on-duplicates.sh`

### 2) 主流程（命令文本层）

命令规定的步骤（`.claude/commands/dedupe.md:10-17`）：

1. 先判断跳过条件：issue 已关闭、无需 dedupe、已存在先前重复评论。
2. 读取目标 issue 并产出摘要。
3. 启动 5 个并行 agent，用不同关键词/策略搜索。
4. 汇总并过滤误报；若无候选则停止。
5. 调用评论脚本写回候选重复项。

备注约束：

- 强制使用 `./scripts/gh.sh`，禁止直接 `gh` 或其它工具（`:21-27`）。
- 明确让 agent 先做 todo list（`:27`）。

### 3) 读取面脚本：`scripts/gh.sh`

`gh.sh` 是受限 `gh` 包装器：

- 仓库范围约束：必须有 `GH_REPO` 或 `GITHUB_REPOSITORY` 且为 `owner/repo` 格式（`scripts/gh.sh:16-21`）
- 仅允许子命令：`issue view/list`、`search issues`、`label list`（`:29-36`）
- 仅允许 flags：`--comments --state --limit --label`（`:23-24,57-60`）
- `issue view` 必须是纯数字 issue 编号（`:84-89`）
- `search issues` 禁止 `repo:/org:/user:` qualifier（`:76-83`），避免跨仓/跨组织检索扩权

该限制与 dedupe 命令“只能在当前仓库查重”的策略一致。

### 4) 写入面脚本：`scripts/comment-on-duplicates.sh`

核心逻辑：

- 参数解析：`--base-issue` + `--potential-duplicates ...`（`scripts/comment-on-duplicates.sh:13-32`）
- 强校验：base issue 和 duplicate issue 必须是数字且存在（`:34-75`）
- 上限控制：候选重复最多 3 个（`:51-53`）
- 评论模板：
  - `Found N possible duplicate issues:`
  - 枚举 issue URL
  - 3 天后自动关闭说明
  - 反对/申诉说明
  - Claude Code attribution（`:77-98`）

该模板是后续自动关闭逻辑的“文本协议”。

### 5) 后处理闭环：自动关闭与回填

1. 自动关闭（定时）
- workflow：`.github/workflows/auto-close-duplicates.yml:1-27`
- 脚本：`scripts/auto-close-duplicates.ts`
- 判定条件：
  - issue 打开超过 3 天（`scripts/auto-close-duplicates.ts:112-144`）
  - 存在 bot 的 dedupe 评论（`:164-172`）
  - dedupe 评论已超过 3 天（`:181-201`）
  - dedupe 评论后无新增评论（`:203-215`）
  - issue 作者未对 dedupe 评论点 👎（`:228-241`）
- 满足后：关闭 issue、打 `duplicate`、发送自动关闭评论（`:66-97,251-266`）

2. 历史回填
- workflow：`.github/workflows/backfill-duplicate-comments.yml:1-44`
- 脚本：`scripts/backfill-duplicate-comments.ts`
- 通过 dispatch 触发 `claude-dedupe-issues.yml` 对历史 issue 做补跑（`scripts/backfill-duplicate-comments.ts:47-70,183-203`）

### 6) 监控/统计外部链路

`claude-dedupe-issues.yml` 在命令执行后总是尝试上报 Statsig 事件（`.github/workflows/claude-dedupe-issues.yml:36-83`），事件名为 `github_duplicate_comment_added`。该统计与“是否真的发表评论”并未在 workflow 层做强耦合校验。

## 关键代码路径与文件引用

- 命令定义
- `.claude/commands/dedupe.md:1-27`

- 调用方（自动触发）
- `.github/workflows/claude-dedupe-issues.yml:1-35`

- 读取接口封装
- `scripts/gh.sh:1-96`

- 评论写回脚本
- `scripts/comment-on-duplicates.sh:1-100`

- 后处理自动关闭
- `.github/workflows/auto-close-duplicates.yml:1-30`
- `scripts/auto-close-duplicates.ts:49-63`
- `scripts/auto-close-duplicates.ts:164-271`

- 历史回填
- `.github/workflows/backfill-duplicate-comments.yml:1-44`
- `scripts/backfill-duplicate-comments.ts:47-70`
- `scripts/backfill-duplicate-comments.ts:160-203`

- 相关目录研究汇总（辅助）
- `Docs/researches/scripts/current_folder_research.md:109-164`
- `Docs/researches/current_folder_research.md:63-66,146`

- 测试现状
- 仓库未发现 `/dedupe` 对应专门自动化测试；当前可靠性主要依赖脚本参数校验与线上 workflow 行为。

## 依赖与外部交互

1. 运行依赖
- GitHub Actions + `anthropics/claude-code-action@v1`（`.github/workflows/claude-dedupe-issues.yml:26`）
- `gh` CLI（被 `scripts/gh.sh` 与评论脚本调用）
- Bun（auto-close/backfill 的 TS 脚本运行时）

2. 关键环境变量
- `GH_TOKEN`/`GITHUB_TOKEN`（workflow 提供）
- `GITHUB_REPOSITORY`（Actions 默认变量，供 `scripts/gh.sh` 兜底）
- `STATSIG_API_KEY`（可选统计上报）

3. 外部系统交互
- GitHub REST API（通过 `gh` 或 fetch）
- Statsig Events API（`https://events.statsigapi.net/v1/log_event`）

4. 权限交互
- dedupe workflow 权限为 `contents: read` + `issues: write`（`.github/workflows/claude-dedupe-issues.yml:17-19`）
- workflow 允许 `allowed_non_write_users: "*"`（`:31`）扩大触发主体范围

## 风险、边界与改进建议

1. 风险：命令步骤编号与语义存在不一致
- 现状：第 3 步写“use summary from #1”，但摘要实际来自第 2 步。
- 影响：执行代理可能误读输入来源。
- 建议：修正文案为“using the summary from #2”。

2. 风险：`comment-on-duplicates.sh` 仓库硬编码
- 现状：`REPO="anthropics/claude-code"`（`scripts/comment-on-duplicates.sh:9`）。
- 影响：在 fork 或复用仓库中可能错误评论到主仓库。
- 建议：优先读取 `GH_REPO/GITHUB_REPOSITORY`，仅无值时回退默认。

3. 风险：文本协议脆弱
- 现状：`auto-close-duplicates.ts` 依赖评论正文包含“Found/possible duplicate”等文本（`scripts/auto-close-duplicates.ts:166-169`）。
- 影响：若评论模板改写，自动关闭可能失效或误判。
- 建议：在评论中加入机器可读 marker（如 HTML 注释），关闭脚本按 marker 解析。

4. 风险：统计事件可能高估
- 现状：workflow 在 `if: always()` 下上报“duplicate_comment_added”（`.github/workflows/claude-dedupe-issues.yml:36-83`），未验证评论是否真正发布。
- 建议：用脚本输出或 GitHub API 结果作为上报条件。

5. 风险：并行 agent 依赖执行器能力
- 现状：命令强要求并行 agent，但 frontmatter 只声明 Bash 工具。
- 影响：跨执行环境时可能出现能力不一致。
- 建议：补充对执行器能力前提说明，或提供“无 agent 并行能力时”的降级策略。

6. 边界：去重结果是“候选”不是判定
- 命令与脚本都只给“possible duplicate”并提供 3 天申诉窗口，不等于立即关闭；最终关闭由定时任务和用户反馈共同决定。

7. 改进建议（测试）
- 为 `scripts/comment-on-duplicates.sh` 增加参数和模板契约测试。
- 为 `scripts/auto-close-duplicates.ts` 增加评论解析与关闭判定单测。
- 为 workflow 增加 dry-run 验证作业，避免线上回归后才暴露问题。
