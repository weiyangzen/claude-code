# .ops/generate_daily_research_todo.sh 研究

## 场景与职责
该脚本负责把全量研究蓝图（`Docs/researches/blueprint_checklist.md`）投影为“当天可执行视图”（`Docs/researches/todos_YYYYMMDD.md`）。

它是研究流水线中面向执行与跟踪的日报生成器：给出 done/pending 快照，并列出当前 pending 项清单，便于人工查看与自动日志记录。

## 功能点目的
1. 按日期生成当天 todo 文件名，沉淀每日快照。
2. 在 checklist 缺失时自动补建（调用 `.ops/generate_research_blueprint_checklist.sh`）。
3. 统计总完成数、总待办数、目录待办数、文件待办数。
4. 输出标准化 Markdown，供人读与脚本消费（`research_guard.sh` 会记录其输出路径）。

## 具体技术实现（关键流程/数据结构/协议/命令）
1. 路径与日期协议：
   - `DATE_TAG="$(date +%Y%m%d)"`
   - 输出文件：`Docs/researches/todos_${DATE_TAG}.md`
2. checklist 自愈：
   - 若 `blueprint_checklist.md` 不存在，执行 `bash .ops/generate_research_blueprint_checklist.sh`。
3. 统计实现：
   - `pending_total`：匹配 `^- \[ \] \[(DIR|FILE)\] `。
   - `done_total`：匹配 `^- \[[xX]\] \[(DIR|FILE)\] `。
   - `dir_pending`：匹配 `^- \[ \] \[DIR\] `。
   - `file_pending`：匹配 `^- \[ \] \[FILE\] `。
   - 统计命令统一为 `rg ... | wc -l | tr -d ' '`。
4. Markdown 渲染协议：
   - 头部：项目名、生成时间、数据来源。
   - `## Snapshot`：四类计数。
   - `## Pending Items`：
     - 若 `pending_total=0`，写 `- [x] No pending research items.`。
     - 否则原样摘录 checklist 中 pending 行。
5. 输出接口：
   - stdout 第一行输出生成摘要。
   - stdout 最后一行输出 todo 文件绝对路径，供上游脚本 `tail -n 1` 获取。

## 关键代码路径与文件引用
- `.ops/generate_daily_research_todo.sh:4-7`：根路径、checklist 路径、日期标签、todo 输出路径。
- `.ops/generate_daily_research_todo.sh:11-13`：checklist 缺失时调用蓝图生成脚本。
- `.ops/generate_daily_research_todo.sh:15-18`：四类统计计算。
- `.ops/generate_daily_research_todo.sh:20-39`：Markdown 内容生成与 pending 明细输出。
- `.ops/generate_daily_research_todo.sh:41-42`：生成摘要与路径输出。
- `.ops/research_guard.sh:172-173`：调用该脚本并记录 `today_todo`。

## 依赖与外部交互
1. 命令依赖：`bash`、`date`、`rg`、`wc`、`tr`。
2. 下游脚本依赖：`.ops/generate_research_blueprint_checklist.sh`（checklist 缺失时触发）。
3. 文件读写：
   - 读：`Docs/researches/blueprint_checklist.md`
   - 写：`Docs/researches/todos_YYYYMMDD.md`
4. 上下游关系：
   - 上游：人工执行、`research_guard.sh` 周期执行。
   - 下游：`todos_*.md` 作为每日研究执行列表与审计快照。

## 风险、边界与改进建议
1. 风险：统计完全依赖行格式正则，若 checklist 格式被人工改动，计数会失真。
2. 风险：脚本不加锁；与 checklist 重建并发时，todo 可能读取到中间态。
3. 边界：脚本仅生成“当天”文件，不会回填历史日期。
4. 边界：默认仅展示 pending 项，不显示 done 项明细。
5. 建议：生成时引入临时文件 + 原子替换，避免写入中断导致 todo 半成品。
6. 建议：与 `research_guard.sh` 共用锁文件，降低并发读写竞争。
7. 建议：除 Markdown 外可增设 machine-readable 输出（如 JSON），便于后续仪表盘集成。
