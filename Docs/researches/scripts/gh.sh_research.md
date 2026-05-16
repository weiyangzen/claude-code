# FILE `scripts/gh.sh` 研究文档

## 场景与职责

`scripts/gh.sh` 是仓库内 AI 自动化访问 GitHub 的安全网关。它不是功能脚本，而是“受限代理层”：把 `gh` CLI 能力收敛到少量只读命令和少量参数，防止 agent 执行任意 gh 操作。

该脚本被 `.claude/commands/triage-issue.md` 与 `.claude/commands/dedupe.md` 明确指定为唯一 GitHub 查询入口。

## 功能点目的

1. 强制仓库作用域在当前仓库（`owner/repo`）内。
2. 白名单限制允许的二级子命令。
3. 白名单限制允许的 flags。
4. 对 `search issues` 做防逃逸检查（禁止 `repo:`/`org:`/`user:`）。
5. 对命令参数形态做额外校验（如 `issue view` 只能一个数字编号）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 仓库作用域绑定

- 读取 `GH_REPO`，缺失时回退 `GITHUB_REPOSITORY`。
- 校验必须是 `owner/repo` 双段格式。
- 失败即报错退出。
- 成功后 `export GH_REPO="$REPO"`，并设置 `GH_HOST=github.com`。

### 2) 命令白名单

仅允许以下组合：

- `issue view`
- `issue list`
- `search issues`
- `label list`

任意其他子命令直接拒绝。

### 3) 参数白名单

允许 flags：

- `--comments`
- `--state`
- `--limit`
- `--label`

其中 `--state` / `--limit` / `--label` 属于“带值参数”，脚本通过 `skip_next` 逻辑把后继值加入 flags 数组。

### 4) 命令分支校验

- `search issues`：
  - 读取 query（第一个 positional）。
  - 拒绝 query 包含 `repo:`/`org:`/`user:`。
  - 运行 `gh search issues <query> --repo <REPO> ...`。
- `issue view`：
  - 必须且仅能有一个数字 issue 编号。
- `issue list` / `label list`：
  - 不允许 positional 参数。

### 5) 安全价值

该 wrapper 把 agent 的 GitHub 访问能力收敛为“只读检索”，从协议层避免：

- 任意仓库访问。
- 任意资源类型访问。
- 任意 flags 携带。

## 关键代码路径与文件引用

- 主实现：`scripts/gh.sh`
- 调用方：
  - `.claude/commands/triage-issue.md`
  - `.claude/commands/dedupe.md`
- 写操作配套脚本：
  - `scripts/edit-issue-labels.sh`
  - `scripts/comment-on-duplicates.sh`
- 触发 workflow：
  - `.github/workflows/claude-issue-triage.yml`
  - `.github/workflows/claude-dedupe-issues.yml`

重点实现段：

- `REPO` 格式校验。
- `CMD` 白名单 case。
- `ALLOWED_FLAGS` 与 `FLAGS_WITH_VALUES` 校验。
- `search issues` 逃逸词过滤。

## 依赖与外部交互

1. 运行依赖
- Bash。
- `gh` CLI。
- `tr`（大小写归一化）。

2. 环境变量
- `GH_REPO` / `GITHUB_REPOSITORY`（至少一个，格式 `owner/repo`）。

3. 外部交互
- 最终调用 GitHub CLI：`gh issue ...`、`gh search issues ...`、`gh label list ...`。

4. 权限模型
- 读取 issue/label/search 一般依赖 `GITHUB_TOKEN`（workflow 环境）或本地 `gh auth` 登录状态。

## 风险、边界与改进建议

1. query 参数容错有限
- `search issues` 只取第一个 positional，额外 positional 未显式报错。
- 建议强制仅允许一个 query positional，超出即失败。

2. flags 值完整性未校验
- 对 `--limit` 这类带值 flag，若缺值或误传可能被错误解析。
- 建议显式检查 `skip_next` 在循环结束时是否仍为 true。

3. 功能覆盖有限
- 当前不允许 `--json` 等输出控制参数，调用方很难拿结构化结果。
- 建议在安全前提下增加少量结构化输出白名单。

4. 报错可观测性可提升
- 目前错误信息主要说明“什么不允许”，缺少完整 usage 回显。
- 建议统一输出 usage 模板，减少误用成本。

5. 读写职责分离依赖人为遵守
- wrapper 只约束自身调用，不能阻止其他脚本直接执行原生 `gh`。
- 建议在命令模板与 CI 规则中进一步强制。
