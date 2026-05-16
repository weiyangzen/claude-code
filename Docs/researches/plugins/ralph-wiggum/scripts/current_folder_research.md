# DIR `plugins/ralph-wiggum/scripts` 研究文档

## 场景与职责

`plugins/ralph-wiggum/scripts` 当前只有一个脚本：`setup-ralph-loop.sh`。它是 Ralph 插件的“启动器”，负责把一次 `/ralph-loop` 命令调用转换成可被 Stop Hook 消费的会话状态文件。

在插件整体链路中的位置：

1. 用户执行 `/ralph-loop`（`plugins/ralph-wiggum/commands/ralph-loop.md:1-18`）。
2. 命令层仅允许执行 `scripts/setup-ralph-loop.sh`，并透传参数（`plugins/ralph-wiggum/commands/ralph-loop.md:4,12-14`）。
3. setup 脚本写入 `.claude/ralph-loop.local.md`（`plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:140-150`）。
4. Stop Hook 在会话退出时读取该状态并决定“放行退出”或“阻断并继续下一轮”（`plugins/ralph-wiggum/hooks/stop-hook.sh:13-18,165-174`）。

因此该目录的核心职责不是“执行循环”，而是“初始化循环状态与行为约束”。

## 功能点目的

### 1) 参数解析与输入校验

- 支持 `--max-iterations`（数值）与 `--completion-promise`（文本）以及自由位置的 prompt 分词输入（`setup-ralph-loop.sh:14-110`）。
- 对缺参、类型错误、空 prompt 给出明确错误提示并退出（`setup-ralph-loop.sh:62-127`）。
- 提供 `--help` 内嵌帮助，降低误用成本（`setup-ralph-loop.sh:16-59`）。

目的：在循环开始前把“终止边界”和“完成信号”定义清楚，避免进入不可控循环。

### 2) 状态文件初始化

- 创建 `.claude` 目录并写入 `.claude/ralph-loop.local.md`（`setup-ralph-loop.sh:131,140-150`）。
- frontmatter 中写入：`active`、`iteration`、`max_iterations`、`completion_promise`、`started_at`（`setup-ralph-loop.sh:141-147`）。
- body 写入原始 prompt（`setup-ralph-loop.sh:149`）。

目的：为 Stop Hook 提供单一真实状态源（single source of truth）。

### 3) 运行期行为提醒与约束输出

- 输出当前迭代、最大迭代、completion promise 说明（`setup-ralph-loop.sh:153-170`）。
- 当设置 completion promise 时，输出严格承诺规则（`setup-ralph-loop.sh:178-203`）。

目的：将“不要伪造完成承诺”的策略从文档层前置到运行时提示层，减少人机协同偏差。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 关键流程

1. 初始化默认值：`MAX_ITERATIONS=0`（无限）、`COMPLETION_PROMISE="null"`（`setup-ralph-loop.sh:9-11`）。
2. 遍历 CLI 参数，区分选项与 prompt token（`setup-ralph-loop.sh:14-110`）。
3. 把 `PROMPT_PARTS` 以空格拼接成最终 prompt（`setup-ralph-loop.sh:112-113`）。
4. 校验 prompt 非空（`setup-ralph-loop.sh:115-128`）。
5. 生成状态文件并写 frontmatter + body（`setup-ralph-loop.sh:140-150`）。
6. 打印循环激活信息与承诺规则（`setup-ralph-loop.sh:153-203`）。

### B. 核心数据结构：`.claude/ralph-loop.local.md`

示例结构（由脚本写入，已在临时目录复现）：

```md
---
active: true
iteration: 1
max_iterations: 3
completion_promise: "DONE"
started_at: "2026-03-19T22:31:27Z"
---

Build API
```

消费关系：

- `stop-hook.sh` 解析 `iteration/max_iterations/completion_promise` 决定是否继续循环（`plugins/ralph-wiggum/hooks/stop-hook.sh:21-25,50-55,114-127`）。
- 同脚本从 body 读取 prompt 作为下一轮 `reason`（`stop-hook.sh:133-137,165-173`）。
- `/cancel-ralph` 直接读取并删除该文件实现手动终止（`plugins/ralph-wiggum/commands/cancel-ralph.md:11-18`）。

### C. Stop Hook 交互协议（与 scripts 目录强相关）

虽然协议实现在 `hooks/stop-hook.sh`，但 `scripts/setup-ralph-loop.sh` 的输出与状态格式直接决定协议能否运行。

Stop Hook 输入（stdin JSON）：

- 关键字段：`transcript_path`（`stop-hook.sh:57-58`）。

Stop Hook 输出（stdout JSON，阻断退出时）：

```json
{
  "decision": "block",
  "reason": "<原始prompt>",
  "systemMessage": "<迭代提示>"
}
```

对应实现：`stop-hook.sh:167-174`。

在临时目录模拟验证中，未满足 completion promise 时输出 `decision=block`，并将状态迭代从 `1` 更新为 `2`；满足 `<promise>DONE</promise>` 时删除状态文件并放行退出。

### D. 关键命令与约束

- 启动命令：`/ralph-loop` -> `"${CLAUDE_PLUGIN_ROOT}/scripts/setup-ralph-loop.sh" $ARGUMENTS`（`commands/ralph-loop.md:12-14`）。
- 命令权限：`allowed-tools` 限制只能执行该脚本（`commands/ralph-loop.md:4`）。
- Hook 注册：Stop 事件绑定 `hooks/stop-hook.sh`（`hooks/hooks.json:3-10`）。

## 关键代码路径与文件引用

### 目录内（目标 DIR）

- `plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:14-110`：参数解析。
- `plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:115-128`：空 prompt 防御。
- `plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:140-150`：状态文件写入。
- `plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:153-203`：运行时提示与完成承诺约束。

### 调用方

- `plugins/ralph-wiggum/commands/ralph-loop.md:1-18`：脚本唯一直接调用入口。
- `plugins/ralph-wiggum/.claude-plugin/plugin.json:2-4`：插件定义与职责描述。

### 被调用方/下游消费者

- `plugins/ralph-wiggum/hooks/stop-hook.sh:13-177`：读取状态、推进迭代、返回 hook 决策 JSON。
- `plugins/ralph-wiggum/commands/cancel-ralph.md:11-18`：读取并删除状态文件。

### 配置与文档上下文

- `plugins/ralph-wiggum/hooks/hooks.json:3-10`：Stop Hook 绑定配置。
- `plugins/ralph-wiggum/README.md:13-33,50-70`：用户面向的机制与命令说明。
- `plugins/README.md:26`：仓库级插件定位（循环命令 + Stop Hook）。

### 测试与验证现状

- 仓库中未发现该目录对应的自动化测试（未检索到 test/spec 入口对 `setup-ralph-loop.sh` 的直接覆盖）。
- 语法层验证：`bash -n plugins/ralph-wiggum/scripts/setup-ralph-loop.sh` 通过。
- 端到端轻量验证：临时目录执行 setup + stop-hook 模拟，状态推进与 promise 终止行为符合代码预期。

## 依赖与外部交互

### 运行依赖

- `bash`：脚本执行环境。
- `date`：生成 `started_at` UTC 时间戳（`setup-ralph-loop.sh:146`）。
- 间接依赖（由下游 hook 消费链路引入）：`jq`、`grep`、`sed`、`awk`、`perl`（`stop-hook.sh:21-25,57-58,71,90-95,119,136,167-174`）。

### 文件系统交互

- 写入：`.claude/ralph-loop.local.md`（setup）。
- 读取/更新/删除：同一状态文件（stop-hook、cancel 命令）。
- 读取：hook input 指向的 transcript JSONL（stop-hook）。

### 协议与环境变量

- 命令执行路径通过 `${CLAUDE_PLUGIN_ROOT}` 引用插件内脚本（`commands/ralph-loop.md:4,13`）。
- Hook 脚本通过 stdin 接收 Claude Code Stop Hook 事件 JSON，通过 stdout 输出 decision JSON（`stop-hook.sh:10,167-174`）。

## 风险、边界与改进建议

### 1) YAML 转义边界不足（高优先级）

现状：`completion_promise` 仅做双引号包裹（`setup-ralph-loop.sh:133-136`），未转义内部 `"`、反斜杠和换行。

风险：特定输入可能写出无效 frontmatter，导致 `stop-hook.sh` 解析异常并提前清理状态。

建议：对 promise 进行 YAML-safe 转义（至少处理 `\`、`"`、换行），或改为 JSON 状态文件避免手写 YAML 转义。

### 2) 未知选项静默并入 prompt（中优先级）

现状：`case *` 分支把任何未知 token 当作 prompt（`setup-ralph-loop.sh:104-108`）。

风险：如用户误写 `--max-iteration`（少 s）不会报错，而是污染 prompt，导致循环边界配置缺失。

建议：当参数以 `-` 开头但不在白名单时直接报错退出。

### 3) 用户提示与实际能力存在不一致（中优先级）

- setup 帮助与激活提示写“cannot be stopped manually”（`setup-ralph-loop.sh:50,166-167`），但插件提供 `/cancel-ralph` 手动终止（`cancel-ralph.md:7-18`）。
- `commands/help.md` 里状态文件路径写成 `.claude/.ralph-loop.local.md`（`help.md:49,69`），与真实实现 `.claude/ralph-loop.local.md` 不一致（`setup-ralph-loop.sh:140`、`stop-hook.sh:13`、`cancel-ralph.md:11`）。

建议：统一文案，避免误导运维与排障。

### 4) 缺少自动化测试（中优先级）

现状：仅能依赖脚本语法检查与人工 smoke test。

建议：补充最小 Bats/shunit2 用例，覆盖：

- 参数错误路径（缺参、非法数字、空 prompt）。
- 状态文件写入格式与字段正确性。
- 与 `stop-hook.sh` 的联动（未完成阻断、完成放行、max_iterations 终止）。

### 5) 输出 prompt 带前导空行（低优先级）

在当前实现中，`reason` 常见值以换行开头（来自 frontmatter 后的空行；`stop-hook.sh:136,171-173`）。

风险较低，但可能影响日志可读性或 prompt 规范化处理。

建议：在提取 prompt 后做一次前导空白裁剪。
