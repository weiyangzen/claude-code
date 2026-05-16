# .ops 目录研究

## 场景与职责

`.ops/` 是本仓库的“研究任务编排层”，核心目标是把“全仓逐项研究并沉淀文档”流程自动化、可重入化、可观测化。它不承载业务代码，而是承载一条运维型流水线：

1. 生成研究蓝图清单（目录/文件全集，带完成状态）。
2. 生成当日待办快照（仅 pending 项）。
3. 选择首个未完成项并驱动 `codex --yolo exec` 自动执行研究。
4. 记录状态与日志，必要时清理 cron 条目。

职责分层如下：

- `generate_research_blueprint_checklist.sh`：清单生成与状态保留。
- `generate_daily_research_todo.sh`：日报待办视图生成。
- `research_guard.sh`：主调度器（选题、提示词拼装、调用 codex、状态机与锁控制）。
- `cleanup_research_cron.sh`：收尾清理（移除研究相关 crontab 条目）。

调用关系（上游/下游）概览：

- 上游调用方：人工执行、cron 任务、tmux 包装后的 worker 进程。
- `.ops` 内部调用：`research_guard.sh -> generate_research_blueprint_checklist.sh -> generate_daily_research_todo.sh`，完成后可选调用 `cleanup_research_cron.sh`。
- 下游被调用方：`codex` CLI、`git`、`tmux`、`flock`、`timeout`、`crontab`、`rg`、`find`、`sed`、`awk`、`wc`、`date`。
- 数据落点：`Docs/researches/*.md` 与 `.cron/research_guard.*`。

## 功能点目的

### 1) 研究蓝图构建

脚本：`.ops/generate_research_blueprint_checklist.sh`

目的：

- 对仓库（排除 `.git/.cron/Docs/researches`）做目录与文件全集扫描。
- 生成 `Docs/researches/blueprint_checklist.md`，每项采用 `- [ ] [DIR|FILE] path`。
- 若 checklist 已存在，读取历史勾选状态并按 `kind:path` 映射回填，避免重复劳动。

### 2) 当日 TODO 快照

脚本：`.ops/generate_daily_research_todo.sh`

目的：

- 基于 checklist 生成当天快照 `Docs/researches/todos_YYYYMMDD.md`。
- 输出 Done/Pending/目录 pending/文件 pending 统计。
- 只列 pending 项，作为人工或自动任务的可读输入。

### 3) 研究守护执行

脚本：`.ops/research_guard.sh`

目的：

- 从 checklist 选取首个未完成项，拼装统一任务提示词。
- 调用 `codex --yolo exec`（非 REPL）执行实质研究。
- 通过锁、tmux worker、状态文件和日志保证并发安全与可恢复。
- 在运行后执行 checkpoint commit（可选 auto-push）。

### 4) 研究 cron 清理

脚本：`.ops/cleanup_research_cron.sh`

目的：

- 从用户 crontab 中移除指向该仓库 `.ops/generate_daily_research_todo.sh` 与 `.ops/research_guard.sh` 的条目。
- 将清理结果写入 `.cron/research_cleanup.*` 便于审计。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. checklist 生成算法

`generate_research_blueprint_checklist.sh` 的关键步骤：

1. 读取旧 checklist，正则匹配 `^- \[([xX\ ])\] \[(DIR|FILE)\] (.+)$`。
2. 将已有状态写入 `STATUS["${kind}:${path}"]`（Bash 关联数组）。
3. `find` 扫描目录和文件，`sed` 去掉 `./` 前缀，`sort -u` 稳定排序。
4. 重建 markdown，按 `DIR` 与 `FILE` 两段输出，保留历史勾选。
5. 用 `rg + wc` 计算 done/pending 统计并打印。

这保证了“结构重建 + 状态继承”，即使仓库结构变化，已研究项也不会丢失。

### B. todo 生成算法

`generate_daily_research_todo.sh` 逻辑：

1. 若 checklist 不存在，先触发 blueprint 生成。
2. 用 `rg` 分别统计：
- 总 pending：`^- \[ \] \[(DIR|FILE)\]`
- 总 done：`^- \[[xX]\] \[(DIR|FILE)\]`
- 目录 pending：`^- \[ \] \[DIR\]`
- 文件 pending：`^- \[ \] \[FILE\]`
3. 渲染 `todos_YYYYMMDD.md`，pending 为 0 时写固定项 `No pending research items.`。

### C. research_guard 主流程

`research_guard.sh` 可分为 8 个阶段：

1. 运行环境初始化：
- 显式扩展 `PATH`，定位 `codex` 可执行文件。
- 准备 `.cron` 日志、状态、阻塞计数文件。

2. tmux 包装分流（默认启用）：
- `TMUX_WRAP_ENABLED=1` 且非 worker 时，优先把任务投递到 tmux 会话窗口。
- 使用 `flock` 控制只投递一次，避免重复启动。

3. 互斥锁：
- 对 `/tmp/${PROJECT}_research_guard.lock` 加独占锁，避免并发守护器重入。

4. 前置生成：
- 强制执行 blueprint 生成；失败则 `failed_blueprint`。
- 执行 daily todo 生成，并把 todo 路径写入日志。

5. 目标选择：
- 从 checklist 抓取首个 pending 行（`head -n 1`）。
- 解析 `LINE_NO / TARGET_TYPE / TARGET_PATH`。
- 计算报告输出路径：
  - DIR -> `Docs/researches/<path>/current_folder_research.md`
  - FILE -> `Docs/researches/<dir>/<basename>_research.md`

6. 任务协议拼装：
- 用 heredoc 生成中文强约束 prompt，固定要求：
  - 深入阅读上下文依赖
  - 输出指定章节
  - 勾选 checklist 对应行
  - 运行 todo 生成脚本
  - 执行 git add/commit
  - 必须使用 `codex exec` 非 REPL

7. 执行与超时：
- 若 `CODEX_EXEC_TIMEOUT_SECONDS>0`，通过 `timeout` 包裹：
  `timeout <sec> codex --yolo exec "$TASK"`
- 返回码 0/124/其他分别写入 `exec_completed/exec_timeout/exec_failed`。

8. 运行后 checkpoint：
- 若存在变更：`git add -A` -> commit `chore(research): checkpoint ...`。
- 当 `AUTO_PUSH_ON_CHECKPOINT=1`，进入自动 push + 冲突处理流程。

### D. auto push 冲突处理协议

`auto_push_with_conflict_resolution()` 设计要点：

1. 首次 `git push` 成功即返回。
2. 若无 upstream，尝试 `git push --set-upstream origin <branch>`。
3. 最多重试 3 次：`fetch` + `rebase remote`。
4. rebase 冲突时，对冲突文件执行 `git checkout --ours` 并 `git add`，随后 `rebase --continue`。
5. 若流程无法收敛，`rebase --abort` 并记录警告。

该策略是“本地优先（ours）”自动解冲突，能提升无人值守成功率，但也带来覆盖远端修改的风险。

### E. cron 清理协议

`cleanup_research_cron.sh --execute` 执行时：

1. 读取当前 crontab（允许为空）。
2. 用 `rg -v` 过滤掉指向当前仓库 `.ops/generate_daily_research_todo.sh|research_guard.sh` 的条目。
3. `sed '/^\s*$/d' | crontab -` 回写。
4. 写入 `.cron/research_cleanup.state` 与 `.cron/research_cleanup.log`。

## 关键代码路径与文件引用

### .ops 目录内关键文件

- `.ops/research_guard.sh`
- `.ops/generate_research_blueprint_checklist.sh`
- `.ops/generate_daily_research_todo.sh`
- `.ops/cleanup_research_cron.sh`

### 研究产物与状态文件

- `Docs/researches/blueprint_checklist.md`
- `Docs/researches/todos_YYYYMMDD.md`
- `Docs/researches/<target>/current_folder_research.md`
- `Docs/researches/<dir>/<file>_research.md`
- `.cron/research_guard.log`
- `.cron/research_guard.state`
- `.cron/research_guard.block_count`

### 关键实现锚点（按职责）

- 目标抽取与报告路径映射：`research_guard.sh` 中 `PENDING_LINE`、`TARGET_TYPE/TARGET_PATH`、`REPORT_PATH` 计算段。
- 提示词协议与 `codex --yolo exec` 调用：`research_guard.sh` 中 heredoc `TASK` 与 `timeout ... codex --yolo exec` 段。
- checklist 状态保留：`generate_research_blueprint_checklist.sh` 中 `declare -A STATUS` 与 regex 回填段。
- runtime 目录排除：`generate_research_blueprint_checklist.sh` 的 `find ... -prune` 段（排除 `.git/.cron/Docs/researches`）。
- cron 移除匹配：`cleanup_research_cron.sh` 中 `rg -v "${REPO_ROOT}/\.ops/(...)"` 段。

## 依赖与外部交互

### 本地命令依赖

- 必需：`bash`, `find`, `sed`, `awk`, `rg`, `wc`, `date`, `git`。
- 守护增强：`flock`, `timeout`, `tmux`（可选但默认启用路径）。
- 清理能力：`crontab`。
- 任务执行器：`codex` CLI（默认路径可被 `CODEX_BIN` 覆盖）。

### 配置与环境变量

`research_guard.sh` 支持如下开关：

- `CODEX_BIN`
- `CODEX_EXEC_TIMEOUT_SECONDS`（默认 5400）
- `AUTO_CLEANUP_ON_COMPLETE`（默认 0）
- `AUTO_PUSH_ON_CHECKPOINT`（默认 0）
- `TMUX_WRAP_ENABLED`（默认 1）
- `TMUX_SESSION_NAME`（默认项目名）
- `RESEARCH_GUARD_WORKER`（内部 worker 标记）

### 文档与上下文依赖

- `.ops` 本身没有 README/注释文档，行为规范主要通过：
- `research_guard.sh` 内置 prompt（即流程“协议文档”）。
- `Docs/researches/blueprint_checklist.md` 与 `todos_*.md`（运行结果文档）。
- 已有研究文档 `Docs/researches/current_folder_research.md`（仓库全局层对 `.ops` 的上层说明）。

### 测试现状

- 未发现 `.ops` 目录的自动化测试（如 Bats/Shell test）与 CI 校验（如 shellcheck workflow）。
- 当前质量保障主要依赖：
- Bash 严格模式 `set -euo pipefail`
- 运行时日志/状态文件
- 实际执行后的 git/checklist/todo 输出验证

## 风险、边界与改进建议

### 主要风险

1. 自动提交范围过宽
- `research_guard.sh` 在运行后使用 `git add -A`，可能把与研究无关的工作区修改一并提交。

2. 自动冲突处理偏激进
- auto-push 时固定 `checkout --ours`，可能覆盖远端新增改动，存在“静默丢失远端变更语义”的风险。

3. checklist 行号耦合
- prompt 里要求“修改第 N 行”，如果执行期间 checklist 被重建或外部并发修改，可能错改。

4. crontab 过滤精度依赖路径字符串
- 清理逻辑基于正则排除路径，若用户 cron 写法变体较大，可能漏删或误删相似条目。

5. tmux/lock 双层并发控制复杂
- tmux 分流 + flock 两层机制提升健壮性，但也增加排障复杂度；故障时需要同时检查 `/tmp` 锁和 tmux 会话。

### 边界条件

- 该体系默认在 Linux/macOS shell 环境，Windows 原生环境兼容性有限。
- `.ops` 只处理“研究文档生产”流程，不负责业务代码语义验证。
- 当 `codex` 不可用时流程不会硬失败退出码（记录状态后退出 0），属于“运维友好但告警弱化”。

### 改进建议

1. 缩小自动提交范围
- 将默认提交改为仅 `Docs/researches` 与目标文件白名单，避免误纳入其他改动。

2. 把“按行号勾选”改为“按 key 勾选”
- 使用 `DIR:<path>` / `FILE:<path>` 唯一键更新 checklist，避免行号漂移问题。

3. 增加最小测试资产
- 为四个 `.ops` 脚本增加 Bats 用例，覆盖：空 checklist、无 pending、tmux 不存在、codex 超时、cron 清理匹配。

4. 增加 shellcheck 与 CI 守护
- 在 workflow 中执行 shellcheck，尽早暴露 Bash 兼容性与引用问题。

5. 提升可观测性
- 结构化状态枚举与失败原因输出（例如 JSON 行日志），便于后续接入告警或面板。
