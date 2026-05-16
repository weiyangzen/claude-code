# FILE `.claude/commands/triage-issue.md` 研究文档

## 场景与职责

`.claude/commands/triage-issue.md` 定义仓库级 `/triage-issue` 命令，负责对 GitHub Issue 做自动化标签治理。该命令强调“只改标签，不发评论”（`.claude/commands/triage-issue.md:8,66`），定位是“分诊策略编排层”，不是标签脚本实现层。

其核心职责：

- 统一新 Issue 的分类、技术域、重复项与生命周期标签判定。
- 在新增评论事件中，仅维护生命周期标签（如 `needs-info`、`needs-repro`、`stale`、`autoclose`）的增删。
- 把执行面限制在 `scripts/gh.sh`（只读）和 `scripts/edit-issue-labels.sh`（写标签）两个脚本。

主要调用方是 workflow：

- `.github/workflows/claude-issue-triage.yml:1-38`
- 触发事件：`issues.opened` 与 `issue_comment.created`（`:3-6`）
- 评论事件过滤：仅非 PR、非 Bot 评论（`:12-15`）
- 调用 prompt：`/triage-issue REPO: ... ISSUE_NUMBER: ... EVENT: ...`（`:35`）

## 功能点目的

1. 新 Issue 入站标准化
- 自动判定是否属于 Claude Code 范畴，并统一类型/领域标签，降低人工初筛成本。

2. 生命周期标签治理
- 对 bug 类 issue 施加或移除 `needs-repro` / `needs-info`，为后续自动关闭流程提供结构化信号。

3. 评论触发后的“状态恢复”
- 新人类评论出现时，及时移除 `stale` / `autoclose` 等标签，避免错误自动关闭。

4. 控制误操作面
- 通过 allowed-tools + 脚本白名单减少错误写操作范围；命令文本也明确禁止评论发布。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) frontmatter 与工具边界

frontmatter：

```yaml
allowed-tools: Bash(./scripts/gh.sh:*),Bash(./scripts/edit-issue-labels.sh:*)
description: Triage GitHub issues by analyzing and applying labels
```

含义：

- 读取仅能通过 `gh.sh` 的白名单命令完成。
- 写操作仅能走 `edit-issue-labels.sh`，避免直接 `gh issue comment`、`gh issue edit` 任意字段操作。

### 2) 输入协议与上下文

命令从 `$ARGUMENTS` 接收上下文（`.claude/commands/triage-issue.md:10-13`），workflow 传入键值对：

- `REPO`
- `ISSUE_NUMBER`
- `EVENT`（`issues` 或 `issue_comment`）

workflow 还设置 `GH_REPO=${{ github.repository }}`（`.github/workflows/claude-issue-triage.yml:31`），使 `scripts/gh.sh` 能固定在当前仓库执行。

### 3) 读取层：`scripts/gh.sh` 的协议约束

`triage-issue` 文本列出了允许的 `gh.sh` 子命令（`.claude/commands/triage-issue.md:15-22`），并与脚本实现一致：

- 允许子命令：`label list`, `issue view`, `issue list`, `search issues`（`scripts/gh.sh:29-36`）
- flags 白名单：`--comments --state --limit --label`（`scripts/gh.sh:23-24,57-60`）
- `issue view` 参数必须是数字（`:84-89`）
- `search issues` 禁止 `repo:/org:/user:` qualifier（`:76-83`）

这构成 triage 阶段的“只读信息面”边界。

### 4) 写入层：`scripts/edit-issue-labels.sh`

命令最终通过：

```bash
./scripts/edit-issue-labels.sh --issue ISSUE_NUMBER --add-label ... --remove-label ...
```

脚本关键行为：

- 参数解析 `--issue` / `--add-label` / `--remove-label`（`scripts/edit-issue-labels.sh:13-32`）
- 强校验 issue 编号合法性与“至少有一个变更”条件（`:34-45`）
- 读取仓库真实标签全集 `gh label list --limit 500`（`:47-49`）
- 过滤不存在标签，避免模型幻觉标签写入（`:50-67`）
- 仅对过滤后的标签执行 `gh issue edit`（`:69-80`）

### 5) 事件分支策略（命令正文）

#### A. `EVENT=issues` 分支

1. 先判断是否属于 Claude Code 问题域；非目标产品打 `invalid` 并停止（`.claude/commands/triage-issue.md:31-34`）。
2. 评估类别标签与平台技术域（`:35-38`）。
3. 可通过 `search issues` 查重，且只标记开放 issue 的重复（`:38`）。
4. 生命周期标签仅针对 bug：
   - `needs-repro`（复现路径不足）
   - `needs-info`（推进所需信息不足）
   详见 `:40-49,68-69`。

#### B. `EVENT=issue_comment` 分支

1. 若有新的人类评论，移除 `stale`/`autoclose`（`:55-57`）。
2. 若补齐信息，移除 `needs-repro`/`needs-info`（`:58-59`）。
3. 可按需要新增生命周期标签，但禁止改类别标签（`:60-63`）。
4. `+1/me too/emoji` 不算补充信息（`:61`）。

### 6) 与生命周期自动化的闭环关系

`/triage-issue` 不是孤立命令，它与以下链路耦合：

1. 生命周期配置单一事实源
- `scripts/issue-lifecycle.ts:3-34` 定义 `invalid/needs-repro/needs-info/stale/autoclose` 的超时天数和提示语。

2. 标签触发评论
- `.github/workflows/issue-lifecycle-comment.yml:1-27` 在 `issues.labeled` 时调用 `scripts/lifecycle-comment.ts`。
- `lifecycle-comment.ts` 会按标签生成“X 天后自动关闭”提示评论（`scripts/lifecycle-comment.ts:19-27`）。

3. 定时关闭执行
- `.github/workflows/sweep.yml:1-30` 调 `scripts/sweep.ts`。
- `sweep.ts` 按 `issue-lifecycle.ts` 的天数关闭超时 issue，同时检查“标签后有人类评论则跳过关闭”（`scripts/sweep.ts:95-149`）。
- 代码注释明确把 triage 的标签移除机制视作第一道防线，sweep 的人类评论检查是兜底（`scripts/sweep.ts:124-127`）。

## 关键代码路径与文件引用

- 命令定义
- `.claude/commands/triage-issue.md:1-70`

- 调用方 workflow
- `.github/workflows/claude-issue-triage.yml:1-38`

- 读取白名单脚本
- `scripts/gh.sh:1-96`

- 标签写入脚本
- `scripts/edit-issue-labels.sh:1-87`

- 生命周期配置与后处理
- `scripts/issue-lifecycle.ts:3-38`
- `.github/workflows/issue-lifecycle-comment.yml:1-27`
- `scripts/lifecycle-comment.ts:19-27`
- `.github/workflows/sweep.yml:1-30`
- `scripts/sweep.ts:92-154`

- 相关研究汇总（辅助）
- `Docs/researches/scripts/current_folder_research.md:8,24-27,159-164`
- `Docs/researches/current_folder_research.md:61-63,147`

- 测试现状
- 仓库未发现 triage 命令和标签脚本的专用单测；当前靠脚本参数校验与线上 workflow 行为保障。

## 依赖与外部交互

1. 运行依赖
- GitHub Actions + `anthropics/claude-code-action@v1`（`.github/workflows/claude-issue-triage.yml:28`）
- `gh` CLI（`gh.sh` 与 `edit-issue-labels.sh`）
- Bun（`lifecycle-comment.ts`、`sweep.ts`）

2. 关键环境变量
- `GH_TOKEN` / `GITHUB_TOKEN`
- `GH_REPO` / `GITHUB_REPOSITORY`
- `GITHUB_REPOSITORY_OWNER` + `GITHUB_REPOSITORY_NAME`（sweep 用）
- `LABEL` + `ISSUE_NUMBER`（lifecycle-comment 用）

3. 外部交互
- GitHub REST API（通过 `gh` 或 `fetch`）
- triage 本身不直接访问第三方统计系统

4. workflow 控制面
- triage workflow 并发键为 `issue-triage-<issue_number>`（`.github/workflows/claude-issue-triage.yml:15-17`），防止同一 issue 上并发分诊冲突。

## 风险、边界与改进建议

1. 风险：命令策略依赖模型判定，强规则有限
- 现状：例如“是否属于 Claude Code”“是否该加 needs-info”主要靠自然语言规则。
- 影响：边缘案例一致性不足。
- 建议：将高风险判定抽到脚本化规则（例如产品域白名单、最小信息字段检查），命令只负责调度。

2. 风险：标签过滤是静默容错
- 现状：`edit-issue-labels.sh` 会过滤不存在标签并继续（`scripts/edit-issue-labels.sh:50-67`）。
- 影响：命令可能“看起来成功但未实际变更”。
- 建议：对被过滤标签输出 warning，并在日志中标注“ignored labels”。

3. 风险：查重仅靠策略约束，不是硬约束
- 现状：命令要求只标记 open duplicates（`.claude/commands/triage-issue.md:38`），脚本层没有强制校验该规则。
- 建议：增加脚本级校验或辅助函数，确保 duplicate 标记不会指向 closed issue。

4. 风险：`allowed_non_write_users: "*"` 的触发面较宽
- triage workflow 同样使用该配置（`.github/workflows/claude-issue-triage.yml:34`）。
- 建议：收敛触发主体，或在 CI 检查中将风险配置从“提醒”提升为“阻断”。

5. 边界：triage 不负责评论沟通
- 命令明确禁止评论；用户沟通依赖 lifecycle-comment/sweep 的标准模板与人工跟进。

6. 改进建议（测试与可观测性）
- 为 `edit-issue-labels.sh` 添加参数与过滤行为测试。
- 为 triage 流程补充 replay 测试样本（new issue/comment 两类输入，验证标签结果）。
- 在 workflow 输出中记录最终 add/remove 标签集合，提升审计可追踪性。
