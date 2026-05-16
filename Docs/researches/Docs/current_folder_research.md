# DIR `Docs` 研究文档

## 场景与职责

`Docs/` 在当前仓库不是产品说明文档中心，而是“研究流程产物根目录”。当前有效内容集中在 `Docs/researches/`，承担以下职责：

1. 持久化研究任务状态
- `Docs/researches/blueprint_checklist.md`：全仓目录/文件研究蓝图与完成状态。
- `Docs/researches/todos_YYYYMMDD.md`：每日待办快照。

2. 沉淀研究报告正文
- 目录研究：`Docs/researches/<target-dir>/current_folder_research.md`
- 文件研究：`Docs/researches/<target-dir>/<filename>_research.md`
- 根对象研究：`Docs/researches/current_folder_research.md`

3. 作为自动化调度的状态源与落盘目标
- `.ops/research_guard.sh` 读取 checklist 首个未完成项，并将输出写入 `Docs/researches/*`。
- `.ops/generate_daily_research_todo.sh` 从 checklist 派生当日 todo。

调用关系（上游/下游）：
- 上游调用方：人工执行 `.ops` 脚本、cron/tmux worker（由 `research_guard.sh` 驱动）。
- 下游消费者：后续研究任务调度器本身（继续读取 checklist）、开发者审阅报告、git 历史审计。

## 功能点目的

### 1. 研究进度单一事实源
`blueprint_checklist.md` 用统一格式 `- [ ] [DIR|FILE] path` 保存状态，使“是否已研究”可机器解析且可人工审阅。

### 2. 每日执行面板
`todos_YYYYMMDD.md` 将 checklist 投影成当天快照，展示 Done/Pending 统计及全部未完成项，便于当天执行与复盘。

### 3. 研究报告归档
`Docs/researches/` 下按目标路径镜像目录结构，保证“目标对象 -> 报告文件”可逆映射，减少跨目录查找成本。

### 4. 自动化可重入与可恢复
即便中断，后续运行也可基于 checklist 中的 `[x]/[ ]` 状态继续推进，不需重新扫描人工记录。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 关键流程

1. 蓝图生成
- 命令：`bash .ops/generate_research_blueprint_checklist.sh`
- 行为：扫描仓库目录/文件，排除 `.git/`、`.cron/` 与 `Docs/researches/`，输出到 `Docs/researches/blueprint_checklist.md`。

2. 每日 todo 生成
- 命令：`bash .ops/generate_daily_research_todo.sh`
- 行为：读取 checklist，统计 done/pending，输出 `Docs/researches/todos_$(date +%Y%m%d).md`。

3. 研究任务执行与落盘
- 命令链：`.ops/research_guard.sh` -> `codex --yolo exec "<任务提示词>"`
- 行为：解析 checklist 首个 pending 项，计算报告路径并要求写入 `Docs/researches/<...>`，然后勾选对应 checklist 项。

### 2) 关键数据结构

1. Checklist 条目协议
- 正则：`^- \[([xX\ ])\] \[(DIR|FILE)\] (.+)$`
- 键模型：`kind:path`（例如 `DIR:Docs`、`FILE:README.md`）
- 作用：在重建 checklist 时保留历史勾选状态。

2. Todo 快照结构
- 头部：`Project` / `Generated at` / `Source`
- 统计：`Done`、`Pending`、`Pending Dirs`、`Pending Files`
- 明细：直接复用 checklist 中所有 pending 条目。

3. 报告命名协议（由 `research_guard.sh` 约束）
- `DIR` 目标：`current_folder_research.md`
- `FILE` 目标：`<原文件名>_research.md`

### 3) 关键命令与实现细节

- 目录准备：`mkdir -p "$REPO_ROOT/Docs/researches"`
- 状态统计：`rg` + `wc -l`
- 排序稳定：`LC_ALL=C sort -u`
- 流程容错：`|| true` 用于避免统计命令在空结果时中断。

## 关键代码路径与文件引用

- `Docs/researches/blueprint_checklist.md`
- `Docs/researches/todos_20260319.md`
- `Docs/researches/current_folder_research.md`
- `Docs/researches/2026-03-19-repo-research.md`
- `Docs/researches/.ops/current_folder_research.md`
- `.ops/generate_research_blueprint_checklist.sh`
- `.ops/generate_daily_research_todo.sh`
- `.ops/research_guard.sh`
- `.ops/cleanup_research_cron.sh`

关键调用路径（调用方 -> 被调用方 -> 产物）：
- `research_guard.sh` -> `generate_research_blueprint_checklist.sh` -> `Docs/researches/blueprint_checklist.md`
- `research_guard.sh` -> `generate_daily_research_todo.sh` -> `Docs/researches/todos_YYYYMMDD.md`
- `research_guard.sh` -> `codex --yolo exec` -> `Docs/researches/<target>/..._research.md`

## 依赖与外部交互

### 本地依赖
- Shell/文本工具：`bash`、`find`、`sed`、`awk`、`rg`、`wc`、`date`
- 版本控制：`git`
- 任务执行器：`codex` CLI
- 调度增强：`tmux`、`flock`、`timeout`（在 guard 场景使用）

### 外部交互
- `Docs/` 本身不直接请求外部 API；其外部交互由上游 `.ops/research_guard.sh` 发起（调用 codex）。
- 研究流程日志与状态写入 `.cron/*`，不写入 `Docs/`，二者共同组成“产物 + 运行态”双通道。

### 配置耦合
- 路径耦合：`.ops` 脚本硬编码 `Docs/researches` 作为输出根。
- 扫描边界：`Docs/researches` 被显式排除，防止将生成产物再次纳入研究对象，避免递归膨胀。

## 风险、边界与改进建议

### 风险

1. 路径强耦合风险
- 问题：多个脚本硬编码 `Docs/researches`；目录改名会导致链路整体失效。
- 建议：抽象 `RESEARCH_ROOT` 环境变量并统一默认值。

2. 行号驱动勾选的漂移风险
- 问题：任务提示要求“修改第 N 行”，若 checklist 在执行期间被并发改写，可能错勾。
- 建议：改为基于 `DIR:<path>/FILE:<path>` 键值匹配更新。

3. 格式脆弱性
- 问题：checklist/todo 依赖固定 markdown 正则格式，手工编辑很容易破坏解析。
- 建议：增加格式校验脚本（例如 pre-commit 或 CI lint）。

4. 产物快速增长
- 问题：`Docs/researches` 随研究推进持续膨胀，阅读成本和 diff 噪音会上升。
- 建议：增加索引页、按日期归档策略，必要时拆分长期归档目录。

5. 测试空白
- 问题：未见对 `Docs` 产物格式的自动化测试。
- 建议：为 `.ops` 脚本新增最小 smoke test，验证必需文件生成与字段完整性。

### 边界

- `Docs/` 当前仅覆盖研究流程文档，不承担用户产品文档（安装、教程）主入口职责。
- `Docs/researches` 属于“流程产物”，权威性取决于脚本执行时上下文和提示词约束，不等价于源码规范文档。
