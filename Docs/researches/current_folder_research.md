# DIR `.` 研究文档

## 场景与职责

本次目标对象是仓库根目录 `.`，它不是单一运行时应用源码，而是一个“Claude Code 生态资产仓库”的聚合层。其核心职责可归纳为四类：

1. 插件市场与分发入口
- 通过 `.claude-plugin/marketplace.json` 声明可分发插件集合（`plugins/*`）。
- 根 README 负责引导安装与官方文档链接，`plugins/README.md` 负责插件目录导航。

2. GitHub 议题治理自动化中枢
- `.github/workflows/*.yml` 将 Issue/Comment/Schedule 事件路由到 Claude Code Action、Bun/TS 脚本、GitHub Script。
- `.claude/commands/*.md` 定义内部 slash-command（如 `/dedupe`、`/triage-issue`）并调用 `scripts/*`。

3. 插件运行样例与可执行能力承载
- `plugins/` 提供 commands/agents/skills/hooks 的参考实现；其中 `hookify`、`security-guidance`、`ralph-wiggum` 含可执行脚本。
- 该仓库承担“模板与能力示例”职能，而非 Claude Code 主程序实现仓库。

4. 研究流程自动化
- `.ops` + `Docs/researches` + `.cron` 构成研究巡检闭环：生成蓝图、选取待研究项、驱动 codex exec、更新 todo/checklist。

补充观察（结构层）：当前仓库（排除 `.git/.cron/Docs/researches`）文件以 `md/json/sh/yml` 为主，体现“文档+配置+脚本编排”的仓库属性。

## 功能点目的

按根目录视角，关键功能点目的如下：

1. 插件市场编排
- 目标：把多个插件作为可发现、可安装、可复用能力包统一暴露。
- 证据：`.claude-plugin/marketplace.json` 内 `plugins[].source` 指向 `./plugins/*`。

2. Issue 生命周期治理
- 目标：在 GitHub Issues 上实现“分流、去重、打标、超时处理、自动关闭、锁定、统计上报”。
- 证据：
  - `claude-issue-triage.yml` 触发 `/triage-issue ...`。
  - `claude-dedupe-issues.yml` 触发 `/dedupe ...`。
  - `sweep.yml`、`auto-close-duplicates.yml` 由定时任务执行脚本。

3. Claude 行为治理（Hook 体系）
- 目标：在 Claude 工具调用前后/停止时注入策略，做安全提醒、流程约束、迭代控制。
- 代表实现：
  - `hookify`：用户规则驱动（`.claude/hookify.*.local.md`）。
  - `security-guidance`：编辑敏感模式时拦截并提示。
  - `ralph-wiggum`：Stop Hook 阻断退出并回灌原始 prompt。

4. 开发环境约束（DevContainer）
- 目标：提供可复现 CLI 环境，并通过 iptables/ipset 限制网络出口。
- 证据：`.devcontainer/devcontainer.json` 在 `postStartCommand` 执行 `init-firewall.sh`。

5. 研究任务自动排产
- 目标：把“目录/文件级研究”任务化、可追踪、可日更。
- 证据：`.ops/research_guard.sh` 读取 checklist 首个未完成项，生成报告路径并调用 `codex --yolo exec`。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) Issue 事件主流程（调用方/被调用方）

1. `issues.opened` 触发并行链路：
- `.github/workflows/claude-issue-triage.yml`
  - 调用 `anthropics/claude-code-action@v1`
  - prompt: `/triage-issue REPO: ... ISSUE_NUMBER: ... EVENT: ...`
  - 被调用方：`.claude/commands/triage-issue.md` -> `scripts/gh.sh` + `scripts/edit-issue-labels.sh`
- `.github/workflows/claude-dedupe-issues.yml`
  - prompt: `/dedupe owner/repo/issues/N`
  - 被调用方：`.claude/commands/dedupe.md` -> `scripts/gh.sh` + `scripts/comment-on-duplicates.sh`
- `.github/workflows/issue-opened-dispatch.yml`
  - 使用 `gh api repos/{target}/dispatches` 做跨仓事件分发。

2. 定时治理：
- `sweep.yml` -> `bun run scripts/sweep.ts`
- `auto-close-duplicates.yml` -> `bun run scripts/auto-close-duplicates.ts`
- `lock-closed-issues.yml` -> `actions/github-script@v7`

3. 生命周期提示：
- `issue-lifecycle-comment.yml` -> `scripts/lifecycle-comment.ts`
- `scripts/lifecycle-comment.ts` 读取 `scripts/issue-lifecycle.ts` 中 `lifecycle[]` 配置。

### 2) 关键数据结构

1. Issue 生命周期配置（`scripts/issue-lifecycle.ts`）
- `lifecycle`：数组项包含 `label / days / reason / nudge`。
- 当前策略含 `invalid(3d) / needs-repro(7d) / needs-info(7d) / stale(14d) / autoclose(14d)`。
- `STALE_UPVOTE_THRESHOLD = 10` 作为保留活跃 issue 的阈值。

2. Hookify 规则模型（`plugins/hookify/core/config_loader.py`）
- `Condition { field, operator, pattern }`
- `Rule { name, enabled, event, conditions, action, tool_matcher, message }`
- 规则文件：`.claude/hookify.*.local.md`（YAML frontmatter + markdown message）。

3. 安全提醒规则集（`plugins/security-guidance/hooks/security_reminder_hook.py`）
- `SECURITY_PATTERNS[]` 同时支持路径匹配与子串匹配。
- 每条规则含 `ruleName` 和 `reminder`，按 session 维度去重提示（state file 在 `~/.claude/`）。

### 3) Hook 协议与阻断机制

1. Hookify 的协议输出（`plugins/hookify/core/rule_engine.py`）
- `PreToolUse/PostToolUse` 阻断格式：
  - `hookSpecificOutput.permissionDecision = "deny"`
  - 携带 `systemMessage`
- `Stop` 阻断格式：
  - `decision = "block"`
  - `reason` 作为阻断原因/回灌内容

2. Security Guidance 的阻断
- 在 `PreToolUse` 场景，脚本用 `sys.exit(2)` 阻断工具调用并向 stderr 输出提醒。

3. Ralph Loop 的 Stop Hook
- 状态文件：`.claude/ralph-loop.local.md`（frontmatter含 `iteration/max_iterations/completion_promise`）。
- Stop Hook 从 `transcript_path` 抽取最后 assistant 消息，匹配 `<promise>...</promise>`；未完成则 `decision:block` 并把原 prompt 回灌。

### 4) 命令级安全收敛（GitHub CLI 包装）

`scripts/gh.sh` 通过白名单约束可执行子命令与参数：
- 允许子命令仅 `issue view/list`, `search issues`, `label list`。
- 允许参数仅 `--comments --state --limit --label`。
- 搜索 query 禁止注入 `repo:/org:/user:` 限定词。
- `issue view` 强制 issue number 为纯数字。

### 5) DevContainer 网络控制实现

- `.devcontainer/devcontainer.json` 通过 `postStartCommand` 执行 `init-firewall.sh`。
- `init-firewall.sh`：
  - 清空规则后恢复 Docker DNS 必要项。
  - 通过 `ipset allowed-domains` 维护允许域/IP。
  - 从 `https://api.github.com/meta` 拉取网段并聚合。
  - 默认 `INPUT/FORWARD/OUTPUT` 均 DROP，再按白名单放行。

### 6) 研究自动化流程

- `.ops/generate_research_blueprint_checklist.sh` 扫描目录/文件，保留已勾选状态。
- `.ops/generate_daily_research_todo.sh` 输出当天快照与 pending 列表。
- `.ops/research_guard.sh`：
  - 读取首个 pending 项 -> 生成 DIR/FILE 对应报告路径
  - 构建固定提示词 -> 调用 `codex --yolo exec`
  - 失败/超时写 state，必要时做 checkpoint commit。

## 关键代码路径与文件引用

以下为根目录级“调用链关键点”索引（调用方 -> 被调用方）：

1. 市场与插件索引
- `.claude-plugin/marketplace.json` -> `plugins/*`
- `README.md` -> `plugins/README.md`

2. Issue 去重/分流链
- `.github/workflows/claude-dedupe-issues.yml` -> `.claude/commands/dedupe.md` -> `scripts/gh.sh` + `scripts/comment-on-duplicates.sh`
- `.github/workflows/claude-issue-triage.yml` -> `.claude/commands/triage-issue.md` -> `scripts/gh.sh` + `scripts/edit-issue-labels.sh`

3. 生命周期治理链
- `.github/workflows/sweep.yml` -> `scripts/sweep.ts` -> `scripts/issue-lifecycle.ts`
- `.github/workflows/issue-lifecycle-comment.yml` -> `scripts/lifecycle-comment.ts` -> `scripts/issue-lifecycle.ts`
- `.github/workflows/auto-close-duplicates.yml` -> `scripts/auto-close-duplicates.ts`
- `.github/workflows/backfill-duplicate-comments.yml` -> `scripts/backfill-duplicate-comments.ts`

4. Hook 执行链
- `plugins/hookify/hooks/hooks.json` -> `plugins/hookify/hooks/*.py` -> `plugins/hookify/core/*.py` -> `.claude/hookify.*.local.md`
- `plugins/security-guidance/hooks/hooks.json` -> `plugins/security-guidance/hooks/security_reminder_hook.py`
- `plugins/ralph-wiggum/hooks/hooks.json` -> `plugins/ralph-wiggum/hooks/stop-hook.sh` -> `.claude/ralph-loop.local.md`

5. 研发环境链
- `.devcontainer/devcontainer.json` -> `.devcontainer/Dockerfile` + `.devcontainer/init-firewall.sh`
- `Script/run_devcontainer_claude_code.ps1` -> `devcontainer up` -> container 内 `claude`

6. 研究流程链
- `.ops/research_guard.sh` -> `.ops/generate_research_blueprint_checklist.sh` + `.ops/generate_daily_research_todo.sh` -> `Docs/researches/*`

## 依赖与外部交互

1. 本地依赖工具
- Shell 工具链：`bash`, `jq`, `curl`, `grep/sed/awk`, `perl`
- GitHub 相关：`gh`
- JS 运行：`bun`（workflow 执行 TS 脚本）
- Python 运行：`python3`（hookify/security hooks）
- 网络控制：`iptables`, `ipset`, `dig`, `aggregate`

2. 外部 API/服务
- GitHub REST: `https://api.github.com/*`
- Statsig 事件上报: `https://events.statsigapi.net/v1/log_event`
- Claude Code Action: `anthropics/claude-code-action@v1`
- HackerOne 漏洞报告入口（SECURITY.md）

3. 凭据与环境变量交互
- workflow 常见变量：`GITHUB_TOKEN`, `ANTHROPIC_API_KEY`, `STATSIG_API_KEY`
- hook 与插件：`CLAUDE_PLUGIN_ROOT`, `ENABLE_SECURITY_REMINDER`
- 研究任务：`CODEX_BIN`, `AUTO_PUSH_ON_CHECKPOINT` 等。

4. 测试与验证现状
- 仓库缺少常规单元测试目录（`__tests__/`, `*.spec.*` 等）；当前验证手段主要是：
  - workflow 运行时验证
  - 命令/脚本自检
  - plugin-dev 技能中提供的脚本化校验（如 `test-hook.sh`）。

## 风险、边界与改进建议

1. 风险：部分脚本仓库目标硬编码
- 现状：`scripts/comment-on-duplicates.sh`、`scripts/backfill-duplicate-comments.ts` 固定 `anthropics/claude-code`。
- 影响：复用于 fork 或其他仓库时易误操作。
- 建议：统一改为 `GH_REPO/GITHUB_REPOSITORY` 或 workflow 输入参数注入。

2. 风险：Hookify 前置解析器为手写 YAML 解析
- 现状：`plugins/hookify/core/config_loader.py` 使用定制 parser。
- 影响：复杂 YAML 场景易出现边界解析偏差。
- 建议：引入安全 YAML 解析库或 JSON schema 约束并加样例回归测试。

3. 风险：缺乏自动化测试基线
- 现状：脚本与 hook 逻辑覆盖广，但测试目录几乎为空。
- 影响：workflow/脚本回归风险高，依赖线上触发暴露问题。
- 建议：至少补充 `scripts/` 与 `plugins/hookify/security-guidance/ralph` 的 smoke tests。

4. 风险：`allowed_non_write_users: "*"` 的误用面
- 现状：部分 workflow 使用该配置；虽有 `non-write-users-check.yml` 守护，但属于事后提醒。
- 影响：配置漂移时可能放大自动化执行面。
- 建议：增加 PR 必须审批规则或在守护 workflow 中改为 fail-fast 策略。

5. 风险：Ralph 无限迭代边界
- 现状：`--max-iterations 0` + 无 completion promise 会无限循环。
- 影响：可能造成资源占用与误用。
- 建议：默认设置安全上限（例如 20），并把“无限模式”改为显式 opt-in。

6. 风险：security-guidance 基于子串匹配易误报
- 现状：对 `pickle`, `eval(` 等做直接子串命中并阻断。
- 影响：误报会降低开发效率。
- 建议：路径+语义联合判断（上下文 AST/语言判定）并提供“本次放行”机制。

7. 风险：DevContainer 防火墙脚本对外部网络/域名解析强依赖
- 现状：初始化阶段需要拉取 GitHub meta 与多域名 DNS 解析。
- 影响：网络抖动会导致容器启动失败。
- 建议：加入可缓存策略与 fallback，允许受控离线模式。

8. 边界说明
- 本仓库重点在插件资产、治理脚本与工作流编排，不等价于 Claude Code 内核代码仓。
- 因此“功能实现研究”主要落在：workflow 调度、脚本协议、hook 机制、插件结构，而非 CLI 主引擎内部算法。
