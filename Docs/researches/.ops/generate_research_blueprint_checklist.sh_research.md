# .ops/generate_research_blueprint_checklist.sh 研究

## 场景与职责
该脚本是研究体系的“蓝图构建器”，负责扫描仓库目录/文件并生成统一 checklist（`Docs/researches/blueprint_checklist.md`），作为后续研究执行和 todo 生成的唯一来源。

核心职责是“重建结构但保留进度”：仓库结构每次可重扫，已完成项通过状态映射继承，不因重建而丢失。

## 功能点目的
1. 建立全量研究对象清单（目录 + 文件），并排除运行时产物目录。
2. 保留已有勾选状态，避免重复研究。
3. 输出规范化 Markdown，供人工审阅与 `research_guard.sh` 自动选题。
4. 输出统计信息（目录数、文件数、pending、done），用于日志与监控。

## 具体技术实现（关键流程/数据结构/协议/命令）
1. 路径初始化：
   - `CHECKLIST_FILE="$REPO_ROOT/Docs/researches/blueprint_checklist.md"`。
2. 状态继承数据结构：
   - 使用 Bash 关联数组 `declare -A STATUS`。
   - 读取旧 checklist 并用正则 `^- \[([xX\ ])\] \[(DIR|FILE)\] (.+)$` 解析。
   - 键格式：`"${kind}:${path}"`，值为 `x` 或空格。
3. 仓库扫描策略：
   - 通过 `find .` 两次扫描目录与文件。
   - 使用 `-prune` 排除：`.git/`、`.cron/`、`Docs/researches/`。
   - 通过 `sed 's|^\./||'` 去前缀，再 `LC_ALL=C sort -u` 稳定排序。
   - 目录列表额外 `awk 'NF==0{print "."; next} {print}'` 把根目录映射为 `.`。
4. Markdown 输出协议：
   - 固定头部（项目、生成时间、排除说明、图例）。
   - 分段输出 `## Directories` 与 `## Files`。
   - 每项格式：`- [ ] [DIR|FILE] path` 或 `- [x] ...`。
5. 统计实现：
   - 生成完成后再次用 `rg + wc` 统计 pending/done。
   - stdout 输出摘要 `generated ... (dirs=, files=, pending=, done=)`。

## 关键代码路径与文件引用
- `.ops/generate_research_blueprint_checklist.sh:10-25`：旧 checklist 解析与 `STATUS` 映射回填。
- `.ops/generate_research_blueprint_checklist.sh:27-34`：目录扫描、排除规则、根目录标准化。
- `.ops/generate_research_blueprint_checklist.sh:36-42`：文件扫描与排序。
- `.ops/generate_research_blueprint_checklist.sh:44-66`：Markdown checklist 重建与状态继承。
- `.ops/generate_research_blueprint_checklist.sh:68-73`：结果统计与摘要输出。
- `.ops/generate_daily_research_todo.sh:11-13`：当 checklist 不存在时调用本脚本。
- `.ops/research_guard.sh:166-170`：每次守护执行前强制调用本脚本。

## 依赖与外部交互
1. 命令依赖：`bash`、`find`、`sed`、`awk`、`sort`、`rg`、`wc`、`tr`、`date`。
2. 文件交互：
   - 读旧文件：`Docs/researches/blueprint_checklist.md`（可选）。
   - 写新文件：`Docs/researches/blueprint_checklist.md`（覆盖写）。
3. 上下游关系：
   - 上游：人工调用、`generate_daily_research_todo.sh`、`research_guard.sh`。
   - 下游：`research_guard.sh` 从该 checklist 抽取首个 pending 项；todo 生成从该文件汇总统计。
4. 无网络依赖：全部操作在本地文件系统与 shell 命令完成。

## 风险、边界与改进建议
1. 风险：覆盖写 checklist 时未加文件锁，与并发执行脚本可能产生竞态。
2. 风险：扫描基于文件系统视图，不遵循 `.gitignore`，会纳入如 `__pycache__` 等噪声文件。
3. 风险：路径匹配与键生成对“包含换行符的文件名”不友好（虽少见但理论存在）。
4. 边界：仅排除 `.git/.cron/Docs/researches` 三类路径，其他大文件或生成目录不会自动忽略。
5. 边界：状态继承依赖 `kind:path` 文本一致；路径重命名后会变成新 pending 项。
6. 建议：增加可配置 ignore 列表（例如读取 `.gitignore` 或单独 `.ops/research-ignore`）。
7. 建议：使用临时文件 + `mv` 原子替换 + 文件锁，降低并发写入损坏风险。
8. 建议：把统计与扫描逻辑拆为可测试函数，补充 shell 单测（如 Bats）。
