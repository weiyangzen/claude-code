# FILE `scripts/edit-issue-labels.sh` 研究文档

## 场景与职责

`scripts/edit-issue-labels.sh` 是 issue triage 流程的标签执行器。它把模型决策转换成可执行的 `gh issue edit` 命令，并在执行前进行“标签存在性过滤”，减少误打不存在标签导致的失败。

主要由 `.claude/commands/triage-issue.md` 调用，通常运行在 `.github/workflows/claude-issue-triage.yml` 中。

## 功能点目的

1. 解析 `--issue`、`--add-label`、`--remove-label` 参数。
2. 校验 issue 编号合法性。
3. 拉取仓库有效标签列表。
4. 仅保留“仓库真实存在”的 add/remove 标签。
5. 执行标签编辑并输出变更摘要。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 参数解析与输入校验

- 允许参数：
  - `--issue <number>`
  - `--add-label <name>`（可多次）
  - `--remove-label <name>`（可多次）
- 约束：
  - `ISSUE` 必填且必须是数字。
  - `ADD_LABELS` 与 `REMOVE_LABELS` 不能同时为空。

不满足条件时直接退出（`exit 1`）。

### 2) 标签白名单过滤

- 通过 `gh label list --limit 500 --json name --jq '.[].name'` 获取仓库标签全集。
- 对 `ADD_LABELS`/`REMOVE_LABELS` 分别执行 `grep -qxF` 精确匹配。
- 不存在的标签被静默丢弃，不会下发到编辑命令。
- 如果过滤后 add/remove 都为空，脚本 `exit 0`（无操作成功返回）。

### 3) 命令拼装执行

- 构建 `GH_ARGS=("issue" "edit" "$ISSUE")`。
- 依次追加 `--add-label` 与 `--remove-label` 参数。
- 最终执行 `gh "${GH_ARGS[@]}"`。
- 成功后输出：`Added: ...` / `Removed: ...`。

### 4) 与 triage 指令协议

`.claude/commands/triage-issue.md` 明确要求：

- 先用 `./scripts/gh.sh label list` 获取可用标签。
- 再调用该脚本做最终写操作。

因此此脚本承担了执行层的最后保护网，即使上游给出无效标签，也不会造成失败或污染。

## 关键代码路径与文件引用

- 主实现：`scripts/edit-issue-labels.sh`
- 调用命令定义：`.claude/commands/triage-issue.md`
- 触发 workflow：`.github/workflows/claude-issue-triage.yml`
- 读操作封装：`scripts/gh.sh`

关键代码段：

- 参数解析 `while/case`。
- `VALID_LABELS` 拉取与精确过滤。
- `GH_ARGS` 动态组装与执行。

## 依赖与外部交互

1. 运行依赖
- Bash。
- GitHub CLI `gh`（需认证）。

2. 外部命令
- `gh label list --json name`
- `gh issue edit <issue> --add-label ... --remove-label ...`

3. 环境上下文
- 仓库作用域通常由 workflow 注入 `GH_REPO`（在 `claude-issue-triage.yml` 中设置）。
- 若未设置 `GH_REPO`，`gh` 将依赖当前目录 git remote 解析仓库。

4. 权限需求
- 需要 issue 写权限（GitHub Actions 里一般对应 `issues: write`）。

## 风险、边界与改进建议

1. 错误信息不足
- 参数错误直接 `exit 1`，无可读错误提示。
- 建议补充 usage 与具体失败原因，降低排障成本。

2. 标签上限固定
- `gh label list --limit 500` 在极端标签量仓库可能截断。
- 建议支持分页拉取或提高上限并记录截断告警。

3. 冲突输入未显式处理
- 同一标签同时 add/remove 时，最终行为依赖 `gh issue edit` 处理顺序。
- 建议在脚本内预先冲突消解并输出提示。

4. 去重缺失
- 多次传入相同标签会重复附加参数。
- 建议在过滤后对 add/remove 数组去重。

5. 仓库作用域隐式
- 脚本本身未强制 owner/repo 格式校验。
- 建议与 `gh.sh` 一样增加 `GH_REPO` 格式校验，避免误操作到错误仓库。
