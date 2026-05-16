# .ops/cleanup_research_cron.sh 研究

## 场景与职责
该脚本用于研究流程收尾阶段，目标是从当前用户 crontab 中移除本仓库研究自动化相关条目，避免在全部研究完成后继续触发无效任务。

它属于 `.ops` 自动化链路的清理器，通常由 `.ops/research_guard.sh` 在 `AUTO_CLEANUP_ON_COMPLETE=1` 时调用，也支持人工手动执行。

## 功能点目的
1. 对执行入口做保护：必须传入 `--execute` 参数，避免误触发。
2. 从 crontab 读取现有任务，过滤掉指向当前仓库 `.ops/generate_daily_research_todo.sh` 和 `.ops/research_guard.sh` 的行。
3. 回写过滤后的 crontab，并记录状态与日志。
4. 在 `.cron/research_cleanup.state` 与 `.cron/research_cleanup.log` 留存审计痕迹。

## 具体技术实现（关键流程/数据结构/协议/命令）
1. 初始化运行路径与日志文件：
   - `REPO_ROOT`：通过 `$(cd "$(dirname "$0")/.." && pwd)` 计算仓库根路径。
   - `LOG_DIR`：固定为 `${REPO_ROOT}/.cron`。
   - `STATE_FILE` 与 `LOG_FILE` 位于 `.cron` 目录。
2. 参数协议：
   - 仅接受 `--execute`，否则打印 `usage` 并 `exit 1`。
3. crontab 清理流程：
   - `current_cron="$(crontab -l 2>/dev/null || true)"` 读取当前 crontab，允许“无 crontab”场景。
   - `rg -v "${REPO_ROOT}/\\.ops/(generate_daily_research_todo\\.sh|research_guard\\.sh)"` 过滤目标条目。
   - `sed '/^\s*$/d' | crontab -` 去除空白行后整体回写。
4. 结果落盘：
   - `STATE_FILE` 写入 `done <timestamp>`。
   - `LOG_FILE` 增加 `removed research cron entries for <repo>`。
   - stdout 输出 `cleanup complete: <repo>`。

该脚本没有复杂数据结构，核心是“字符串过滤 + 全量回写”协议。

## 关键代码路径与文件引用
- `.ops/cleanup_research_cron.sh:4-7`：路径与日志/状态文件定义。
- `.ops/cleanup_research_cron.sh:14-17`：强制 `--execute` 入口保护。
- `.ops/cleanup_research_cron.sh:19-22`：`crontab -l` 读取、`rg -v` 过滤、`crontab -` 回写。
- `.ops/cleanup_research_cron.sh:24-26`：状态写入、日志记录、完成输出。
- `.ops/research_guard.sh:180-182`：研究完成后可选自动调用清理脚本。

## 依赖与外部交互
1. 命令依赖：`bash`、`crontab`、`rg`、`sed`、`date`。
2. 文件系统交互：
   - 写 `.cron/research_cleanup.state`
   - 写 `.cron/research_cleanup.log`
3. 系统级外部交互：
   - 直接读写“当前用户”crontab（不是仓库内文件），属于全局副作用操作。
4. 上下游关系：
   - 上游：人工执行，或 `.ops/research_guard.sh` 自动触发。
   - 下游：用户级 cron 调度配置被修改。

## 风险、边界与改进建议
1. 风险：过滤规则基于路径子串，若其他 cron 命令行恰好包含相同字符串，可能被误删。
2. 风险：`sed '/^\s*$/d'` 会删除所有空白行，可能改变用户原有 crontab 的格式习惯。
3. 边界：脚本只清理“当前仓库绝对路径”对应的两个脚本，不处理其他仓库或别名命令。
4. 边界：未检测 `crontab` 命令是否存在；极简环境可能执行失败。
5. 建议：改为先写临时文件并 `crontab <tmp>`，失败时保留原文件快照，提升可回滚性。
6. 建议：增加 `--dry-run` 模式，先展示将移除的行，再由操作者确认。
7. 建议：过滤时可增加更严格锚点（如完整命令 token 边界），降低误匹配概率。
