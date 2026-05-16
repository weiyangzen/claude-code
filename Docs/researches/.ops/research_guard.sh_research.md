# .ops/research_guard.sh 研究

## 场景与职责
`.ops/research_guard.sh` 是整条研究自动化链路的主调度器，负责将“蓝图清单中的待研究项”转化为可执行任务，并以无人值守方式驱动 `codex --yolo exec` 完成文档产出、checklist 回填、todo 刷新与提交。

它位于 `.ops` 体系中心，向上承接人工/cron 触发，向下串联 checklist 生成、todo 生成、任务提示词构建、执行超时控制、日志状态记录以及可选自动清理与自动推送。

## 功能点目的
1. 并发安全执行：通过 `flock` 和可选 tmux worker 包装，避免重复运行。
2. 自动选题：从 `Docs/researches/blueprint_checklist.md` 抓取首个 pending 项。
3. 任务协议生成：按 `DIR`、`FILE`、`FILE_BATCH` 三种场景拼装强约束中文 prompt。
4. 执行控制：调用 `codex --yolo exec`，可配置超时。
5. 运行后收敛：自动 checkpoint commit；可选尝试 auto-push 与冲突自动处理。
6. 生命周期状态记录：通过 `.cron/research_guard.state`、`.cron/research_guard.log` 跟踪运行结果。

## 具体技术实现（关键流程/数据结构/协议/命令）
### 1) 启动与环境注入
1. 使用严格模式：`set -euo pipefail`。
2. 在脚本开头扩展 `PATH`，优先注入 codex 所在路径，降低 cron 非交互环境找不到命令的风险。
3. 定义关键环境变量：
   - `CODEX_BIN`：codex 可执行路径（可覆盖）。
   - `AUTO_CLEANUP_ON_COMPLETE`：完成后自动清理 cron 开关。
   - `CODEX_EXEC_TIMEOUT_SECONDS`：任务执行超时秒数（默认 5400）。
   - `AUTO_PUSH_ON_CHECKPOINT`：checkpoint 后是否尝试 push。
   - `MAX_BATCH_BYTES`：文件批次上限（默认 102400）。
   - `TMUX_WRAP_ENABLED`、`TMUX_SESSION_NAME`、`RESEARCH_GUARD_WORKER`：tmux 包装控制。

### 2) tmux 包装与双层锁
1. 非 worker 且 `TMUX_WRAP_ENABLED=1` 时优先进入 tmux 包装逻辑。
2. 先用 `flock -n 8` 快速判重，避免重复投递 tmux 窗口。
3. 若会话存在则 `tmux new-window`，否则 `tmux new-session`。
4. 投递成功后主进程直接退出，由 tmux worker 执行真实任务。
5. 若 tmux 不可用/创建失败，退化为本地执行并写 warn 日志。

随后使用 `flock -n 9` 对同一锁文件做执行互斥，防止真实执行阶段并发重入。

### 3) 前置校验与预生成
1. 校验 codex 可执行：优先 `CODEX_BIN`，否则回退 `command -v codex`。
2. 若 codex 不可用，写状态 `failed_no_codex` 并记录错误日志后退出。
3. 强制调用 `.ops/generate_research_blueprint_checklist.sh`；失败写 `failed_blueprint`。
4. 调用 `.ops/generate_daily_research_todo.sh` 并记录其最后一行路径到日志字段 `today_todo`。

### 4) 目标项解析与完成态分支
1. 通过 `rg -n '^- \[ \] \[(DIR|FILE)\] ' "$CHECKLIST_FILE" | head -n 1` 选择首个 pending。
2. 若无 pending：
   - `state=completed`
   - `block_count=0`
   - 记录 `all research checklist items completed`
   - 若 `AUTO_CLEANUP_ON_COMPLETE=1`，执行 `.ops/cleanup_research_cron.sh --execute`。
3. 若有 pending：
   - 解析 `LINE_NO`、`TARGET_TYPE`、`TARGET_PATH`。
   - 进入 DIR/FILE 两类任务拼装分支。

### 5) 任务拼装协议
#### DIR 分支
1. 目标报告路径：
   - 根目录 `.` -> `Docs/researches/current_folder_research.md`
   - 其他目录 -> `Docs/researches/<dir>/current_folder_research.md`
2. 生成 heredoc `TASK`，强制要求：
   - 深入阅读上下文依赖。
   - 产出固定章节文档。
   - 将 checklist 指定行从 `[ ]` 改 `[x]`。
   - 运行 `bash .ops/generate_daily_research_todo.sh`。
   - 执行 `git add`/`git commit`。
   - 使用 codex exec 非 REPL。

#### FILE 分支（含批处理）
1. 从当前目标文件所在目录起步，遍历 checklist 中同目录连续 pending 文件。
2. 累加每个候选文件大小（`wc -c`），控制总量不超过 `MAX_BATCH_BYTES`。
3. 特殊规则：若批次第一个文件单体超上限，允许单文件批次继续（避免永久阻塞）。
4. 若批次数量 `<=1`：走单文件 prompt。
5. 若批次数量 `>1`：走 `FILE_BATCH` prompt，列出每个文件对应 checklist 行号、研究文档输出路径、字节数。

### 6) 执行与超时
1. 设置状态：`running_exec`。
2. 若 `CODEX_EXEC_TIMEOUT_SECONDS > 0`，使用 `timeout` 包裹执行：
   - `timeout <sec> "$CODEX_BIN" --yolo exec "$TASK"`
3. 否则直接执行 codex。
4. 返回码分流：
   - `0` -> `exec_completed`
   - `124` -> `exec_timeout`
   - 其他 -> `exec_failed`

### 7) 运行后 checkpoint 与可选 auto-push
1. 若存在工作区或暂存区变更，先 `git add -A`。
2. 若暂存区非空，提交 `chore(research): checkpoint <timestamp>`。
3. `AUTO_PUSH_ON_CHECKPOINT=1` 时调用 `auto_push_with_conflict_resolution()`。

`auto_push_with_conflict_resolution()` 核心协议：
1. 先尝试普通 `git push`。
2. 无 upstream 时尝试 `git push --set-upstream origin <branch>`。
3. 最多 3 次重试：`fetch` + `rebase remote`。
4. rebase 冲突时对冲突文件执行 `git checkout --ours`，再 `git add` + `rebase --continue`。
5. 无法收敛时 `rebase --abort` 并记录 warning。

## 关键代码路径与文件引用
- `.ops/research_guard.sh:4-21`：环境变量、路径、开关参数定义。
- `.ops/research_guard.sh:30-59`：tmux 包装分流与失败回退。
- `.ops/research_guard.sh:148-152`：执行阶段全局互斥锁。
- `.ops/research_guard.sh:156-164`：codex 可执行校验与降级处理。
- `.ops/research_guard.sh:166-173`：前置生成 blueprint + todo。
- `.ops/research_guard.sh:175-190`：首个 pending 行解析。
- `.ops/research_guard.sh:195-229`：DIR 任务 prompt 模板。
- `.ops/research_guard.sh:231-347`：FILE/FILE_BATCH 批次构造与 prompt 模板。
- `.ops/research_guard.sh:353-358`：`codex --yolo exec` 执行与 timeout 包裹。
- `.ops/research_guard.sh:360-373`：变更检测与 checkpoint commit。
- `.ops/research_guard.sh:375-384`：状态机收尾与结果日志。
- `.ops/research_guard.sh:61-146`：auto-push 自动冲突处理函数。
- `.ops/generate_research_blueprint_checklist.sh`：被该脚本前置调用。
- `.ops/generate_daily_research_todo.sh`：被该脚本前置调用并记录输出。
- `.ops/cleanup_research_cron.sh`：完成态可选调用。

## 依赖与外部交互
1. 本地命令依赖：`bash`、`rg`、`sed`、`wc`、`date`、`timeout`、`flock`、`git`。
2. 可选依赖：`tmux`（用于后台 worker 委托）。
3. 执行器依赖：`codex` CLI（默认路径可由 `CODEX_BIN` 覆盖）。
4. 文件交互：
   - 读写 `Docs/researches/blueprint_checklist.md`
   - 读写 `Docs/researches/todos_YYYYMMDD.md`
   - 写 `.cron/research_guard.log`
   - 写 `.cron/research_guard.state`
   - 写 `.cron/research_guard.block_count`
5. 系统外部交互：
   - `tmux` 会话/窗口创建。
   - `/tmp` 锁文件占用。
   - `git` 提交与可选 push（远端交互）。
6. 调用关系：
   - 上游：人工、cron 调度。
   - 下游：`.ops` 其他脚本与 codex 执行流程。

## 风险、边界与改进建议
1. 风险：`git add -A` 范围过大，可能将研究任务之外的改动打包进 checkpoint。
2. 风险：auto-push 冲突处理固定 `--ours`，会优先本地版本，存在覆盖远端语义的可能。
3. 风险：prompt 通过“checklist 第 N 行”定位，若并发重建 checklist，可能发生行号漂移。
4. 风险：tmux 包装 + 双锁机制复杂，问题定位需同时检查 tmux、锁、状态文件和日志。
5. 风险：脚本使用绝对 PATH 注入，迁移到其他机器时可能失效或误导定位。
6. 边界：这是流程编排脚本，不验证研究文档质量真实性；质量主要依赖 prompt 约束与人工复核。
7. 边界：无内建单元测试；行为正确性主要靠线上运行日志反馈。
8. 建议：默认提交范围改为白名单（`Docs/researches` 与当前目标文件），避免误提交。
9. 建议：用 `kind:path` 精确键更新 checklist，替代“按行号勾选”。
10. 建议：将日志改为结构化字段（JSONL），便于后续自动告警与分析。
11. 建议：给 `.ops` 增加 shellcheck + Bats 测试，覆盖超时、无 codex、批次边界、completed 清理分支。
12. 建议：将 PATH 注入与机器相关路径参数化，避免硬编码环境耦合。
