# FILE `plugins/ralph-wiggum/scripts/setup-ralph-loop.sh` 研究文档

## 场景与职责
`setup-ralph-loop.sh` 是 Ralph Wiggum 插件的会话级初始化脚本，职责是把一次 `/ralph-loop` 调用转换为可持续循环的状态文件，并在当前会话内激活 Stop Hook 控制流。

它处在如下链路中：
1. 用户执行 `/ralph-loop ...`。
2. `plugins/ralph-wiggum/commands/ralph-loop.md` 仅允许执行该脚本，并透传 `$ARGUMENTS`（`plugins/ralph-wiggum/commands/ralph-loop.md:4,13`）。
3. 本脚本写入 `.claude/ralph-loop.local.md`（`plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:140-150`）。
4. Stop Hook 在用户尝试退出时读取该状态文件并决定“放行退出”或“阻断并继续循环”（`plugins/ralph-wiggum/hooks/stop-hook.sh:13-174`，`plugins/ralph-wiggum/hooks/hooks.json:4-10`）。
5. `/cancel-ralph` 可以手动删除该状态文件，终止循环（`plugins/ralph-wiggum/commands/cancel-ralph.md:3,11-18`）。

结论：该脚本是 Ralph 运行时的“单一初始化入口”，同时定义了 Stop Hook 的状态文件协议基线。

## 功能点目的
脚本核心功能可拆为 5 组：
1. CLI 参数解析与帮助输出。
- 支持 `-h/--help`、`--max-iterations <n>`、`--completion-promise <text>`，其余 token 视作 prompt 组成部分（`setup-ralph-loop.sh:14-110`）。
2. 输入有效性约束。
- `--max-iterations` 必须为非负整数（`^[0-9]+$`）（`setup-ralph-loop.sh:61-85`）。
- `--completion-promise` 必须带值（`setup-ralph-loop.sh:87-103`）。
- prompt 不能为空（`setup-ralph-loop.sh:115-128`）。
3. 生成循环状态文件。
- 目标文件：`.claude/ralph-loop.local.md`。
- frontmatter 字段：`active`、`iteration`、`max_iterations`、`completion_promise`、`started_at`（`setup-ralph-loop.sh:140-147`）。
- body 写入原始 prompt（`setup-ralph-loop.sh:149`）。
4. 启动后行为提示。
- 输出迭代信息、完成承诺约束、监控命令，帮助操作者理解“同 prompt 迭代”机制（`setup-ralph-loop.sh:153-203`）。
5. 与 Stop Hook 协议对齐。
- `iteration/max_iterations/completion_promise` 字段格式直接被 `stop-hook.sh` 解析消费（`stop-hook.sh:21-25,50-55,115-127`）。

## 具体技术实现（关键流程/数据结构/协议/命令）
### 1) 关键流程
A. 参数处理流程
1. 初始化默认值：`MAX_ITERATIONS=0`（无限循环）、`COMPLETION_PROMISE="null"`（未启用承诺）（`setup-ralph-loop.sh:10-11`）。
2. `while + case` 扫描参数（`setup-ralph-loop.sh:14-110`）：
- 选项型参数消费两个位置（选项名 + 值）。
- 非选项参数累积进数组 `PROMPT_PARTS`。
3. `PROMPT="${PROMPT_PARTS[*]}"` 组装 prompt（`setup-ralph-loop.sh:113`）。
4. 空 prompt 失败快返并提示示例（`setup-ralph-loop.sh:115-128`）。

B. 状态文件落盘流程
1. `mkdir -p .claude` 保证目录存在（`setup-ralph-loop.sh:131`）。
2. `completion_promise` 做“null 或双引号字符串”二分（`setup-ralph-loop.sh:133-138`）。
3. 通过 heredoc 一次性写入 `.claude/ralph-loop.local.md`（`setup-ralph-loop.sh:140-150`）。

C. 运行提示流程
1. 输出循环激活信息，动态展示最大迭代值/承诺值（`setup-ralph-loop.sh:153-170`）。
2. 回显原始 prompt（`setup-ralph-loop.sh:172-176`）。
3. 若配置了承诺，输出严格规则（`setup-ralph-loop.sh:178-203`）。

### 2) 数据结构
核心数据结构是 markdown + YAML frontmatter 状态文件：`.claude/ralph-loop.local.md`。

典型结构（由 `setup-ralph-loop.sh:140-150` 生成）：
```markdown
---
active: true
iteration: 1
max_iterations: 20
completion_promise: "DONE"
started_at: "2026-03-20T12:31:37Z"
---

<原始 prompt 文本>
```

字段消费关系：
- `iteration` / `max_iterations`：Stop Hook 数值判断是否到达上限（`stop-hook.sh:50-55`）。
- `completion_promise`：Stop Hook 从最后 assistant 文本提取 `<promise>...</promise>` 后做字面值比较（`stop-hook.sh:115-127`）。
- body prompt：Stop Hook 在继续循环时作为 `reason` 回注（`stop-hook.sh:133-137,167-173`）。

### 3) 协议与命令
A. 命令协议
- Slash command 入口声明：`argument-hint` 为 `PROMPT [--max-iterations N] [--completion-promise TEXT]`（`commands/ralph-loop.md:3`）。
- 调用协议：`"${CLAUDE_PLUGIN_ROOT}/scripts/setup-ralph-loop.sh" $ARGUMENTS`（`commands/ralph-loop.md:13`）。

B. Stop Hook 协议
- Hook 注册：Stop 事件执行 `hooks/stop-hook.sh`（`hooks/hooks.json:4-10`）。
- Hook 输入：从 stdin 读取 JSON，使用 `jq -r '.transcript_path'` 获取 transcript 路径（`stop-hook.sh:9-10,57-59`）。
- Hook 输出：当需继续循环时输出 `{"decision":"block","reason":<prompt>,"systemMessage":...}`（`stop-hook.sh:167-174`）。

C. 工具命令实现
`setup-ralph-loop.sh` 使用的外部命令：`mkdir`、`date`、`cat`、`echo`（`setup-ralph-loop.sh:131,140-203`）。
`stop-hook.sh` 额外使用：`jq`、`grep`、`sed`、`awk`、`perl`、`mv`、`rm`（`stop-hook.sh:21-25,57-174`）。

### 4) 本次验证记录
1. 语法检查通过：
- `bash -n plugins/ralph-wiggum/scripts/setup-ralph-loop.sh`
- `bash -n plugins/ralph-wiggum/hooks/stop-hook.sh`

2. 隔离目录试跑（`--completion-promise 'DONE "NOW"'`）复现到状态落盘行为：
- 生成内容：`completion_promise: "DONE "NOW""`。
- 该字符串并未转义内部 `"`，从 YAML 规范角度不安全，后续如果切换为严格 YAML 解析器会失败。

## 关键代码路径与文件引用
主路径（运行时）：
1. `plugins/ralph-wiggum/commands/ralph-loop.md:1-18`
2. `plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:1-203`
3. `plugins/ralph-wiggum/hooks/hooks.json:1-15`
4. `plugins/ralph-wiggum/hooks/stop-hook.sh:1-177`
5. `plugins/ralph-wiggum/commands/cancel-ralph.md:1-18`

文档与说明路径（非运行时但影响可用性认知）：
1. `plugins/ralph-wiggum/README.md:13-134`
2. `plugins/ralph-wiggum/commands/help.md:34-70`
3. `plugins/README.md:26`
4. `plugins/plugin-dev/skills/plugin-settings/references/real-world-examples.md:129-264`

关键实现片段定位：
1. 参数解析：`setup-ralph-loop.sh:14-110`
2. prompt 校验：`setup-ralph-loop.sh:115-128`
3. 状态文件写入：`setup-ralph-loop.sh:140-150`
4. 激活与承诺提示：`setup-ralph-loop.sh:153-203`
5. 停止拦截与继续循环：`stop-hook.sh:50-55,130-174`

## 依赖与外部交互
### 1) 运行时依赖
1. Shell 环境：`bash`（`set -euo pipefail`）。
2. CLI 工具链：`date/jq/grep/sed/awk/perl/mv/rm/test` 等（主要在 stop hook）。
3. Claude Hook API：Stop Hook 通过 stdin 输入获取 `transcript_path`，并以 JSON 输出 block 决策（`stop-hook.sh:9-10,57-59,167-174`）。

### 2) 文件系统交互
1. 写入：`.claude/ralph-loop.local.md`（setup 初始化）。
2. 读取+更新+删除：同一文件由 stop-hook/cancel 命令共享。
3. 读取：Hook 读取 transcript JSONL（路径来自 hook input）。

### 3) 配置与权限约束
1. `commands/ralph-loop.md` 的 `allowed-tools` 把执行权限限制在 setup 脚本路径（`ralph-loop.md:4`）。
2. `commands/cancel-ralph.md` 只允许 `test/rm/read` 状态文件相关操作（`cancel-ralph.md:3`）。
3. 插件级元数据在 `.claude-plugin/plugin.json`（版本、描述、作者）（`plugin.json:1-8`）。

### 4) 测试覆盖现状
- 仓库内未检索到针对 `setup-ralph-loop.sh`/`stop-hook.sh` 的自动化测试（无对应 `test/spec` 用例入口）。
- 当前可见保障主要是脚本内参数校验、hook 运行时容错、以及人工/手动验证。

## 风险、边界与改进建议
1. `completion_promise` YAML 转义不完整（高优先级）。
- 现状：仅做外层双引号包裹（`setup-ralph-loop.sh:133-136`）。
- 风险：当承诺文本含 `"`、反斜杠、换行时，状态文件 YAML 语义不稳定。
- 建议：用可靠序列化（例如 `python -c 'import json,yaml'` 或 `jq -Rn` + 明确转义策略）统一生成 YAML-safe 字面量。

2. 未知选项被吞入 prompt（中优先级）。
- 现状：`case *` 直接进 `PROMPT_PARTS`（`setup-ralph-loop.sh:104-108`）。
- 风险：用户拼错参数（如 `--max-iteratons`）不会报错，只会变成 prompt 文本，导致意图失效。
- 建议：对 `-*` 且未识别参数显式报错，非 `-` 前缀 token 才作为 prompt。

3. 文案与能力不一致（中优先级）。
- 现状：setup/help 文案有“cannot be stopped manually”倾向（`setup-ralph-loop.sh:50,166-167`），但插件存在 `/cancel-ralph`（`cancel-ralph.md:7-18`）。
- 影响：用户对停止路径的心智模型不一致。
- 建议：统一文案为“默认会持续循环，除非达到 max/promise 或执行 `/cancel-ralph`”。

4. 帮助文档状态文件路径错误（中优先级）。
- 现状：`help.md` 写成 `.claude/.ralph-loop.local.md`（`help.md:49,69`）。
- 实现：真实路径是 `.claude/ralph-loop.local.md`（setup/stop/cancel 全一致）。
- 建议：修正文档，避免误导排障。

5. 并发与原子性边界（中低优先级）。
- `setup` 写文件使用重定向直写，若多会话并发启动同一项目循环，后写会覆盖先写。
- `stop-hook` 更新 `iteration` 已采用 temp + `mv`，但无锁机制（`stop-hook.sh:154-156`）。
- 建议：必要时引入 lock 文件或会话维度 state（例如按 session id 分文件）。

6. 自动化回归缺失（中优先级）。
- 建议新增 shell 测试矩阵：
  - 参数错误分支（缺值/非法数字/空 prompt）。
  - promise 匹配与不匹配。
  - `max_iterations` 达标退出。
  - transcript 缺失与 JSON 异常容错路径。

