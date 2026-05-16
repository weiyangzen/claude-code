# FILE `plugins/ralph-wiggum/README.md` 研究文档

## 场景与职责
`plugins/ralph-wiggum/README.md` 是 Ralph Wiggum 插件的对外操作说明与方法论入口，职责不是实现循环本身，而是把“会话内自循环开发”这套能力讲清楚，并把用户引导到正确命令与安全边界。

该 README 在插件体系中的定位：
- 在插件总览中被标记为“`/ralph-loop` + `/cancel-ralph` + Stop Hook”的组合型插件（`plugins/README.md:26`）。
- 与插件标准结构约定一致（`.claude-plugin/plugin.json`、`commands/`、`hooks/`、`scripts/`）（`plugins/README.md:49-60`）。
- 插件元数据把该能力定义为“while-true 风格同 prompt 迭代直到完成”（`plugins/ralph-wiggum/.claude-plugin/plugin.json:2-4`）。

README 自身承担三类职责：
1. 概念建模：解释 Ralph 的“同 prompt 重复 + 文件状态累积”机制（`plugins/ralph-wiggum/README.md:13-33`）。
2. 操作引导：给出 `/ralph-loop`、`/cancel-ralph` 的使用入口与参数（`plugins/ralph-wiggum/README.md:48-70`）。
3. 风险防护：强调 `--max-iterations` 作为主安全阀，避免无限循环（`plugins/ralph-wiggum/README.md:119-135`）。

## 功能点目的
围绕 README 所声明能力，实际功能可以拆成 6 个目的层：

1. 启动循环
- 目的：一次命令启动会话内持续迭代任务。
- 对应入口：`/ralph-loop`（`plugins/ralph-wiggum/README.md:50-57`，`plugins/ralph-wiggum/commands/ralph-loop.md:1-14`）。

2. 维持“同 prompt”闭环
- 目的：每次尝试 Stop 时将原始 prompt 回灌，形成“同输入、异状态”迭代。
- 对应实现：Stop hook 返回 `decision:block` 且 `reason=prompt`（`plugins/ralph-wiggum/hooks/stop-hook.sh:165-174`）。

3. 自动终止条件
- 目的：在“达到最大轮次”或“命中完成承诺”后退出循环。
- 对应实现：
  - `max_iterations` 上限判定（`plugins/ralph-wiggum/hooks/stop-hook.sh:50-55`）。
  - `<promise>...</promise>` 文本与 `completion_promise` 精确匹配（`plugins/ralph-wiggum/hooks/stop-hook.sh:114-127`）。

4. 人工取消
- 目的：提供运维逃生口，避免必须等待自动条件。
- 对应入口：`/cancel-ralph` 删除状态文件（`plugins/ralph-wiggum/commands/cancel-ralph.md:11-18`）。

5. 提示词工程约束
- 目的：通过 README 的 best practices 提升循环可收敛性，减少“无验证目标”的空转。
- 对应章节：清晰完成标准、分阶段目标、自纠步骤、escape hatch（`plugins/ralph-wiggum/README.md:72-135`）。

6. 操作预期管理
- 目的：限定“适合/不适合”场景，避免被用于需要高强人类判断的任务。
- 对应章节：When to Use Ralph（`plugins/ralph-wiggum/README.md:152-165`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 关键流程（调用链）
完整调用链如下：
1. 用户执行 `/ralph-loop <PROMPT> [--max-iterations N] [--completion-promise TEXT]`（`plugins/ralph-wiggum/README.md:56-61`）。
2. 命令文件只允许调用 setup 脚本，并透传参数（`plugins/ralph-wiggum/commands/ralph-loop.md:4,12-14`）。
3. `setup-ralph-loop.sh` 解析参数、校验输入、写入状态文件 `.claude/ralph-loop.local.md`（`plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:14-150`）。
4. Claude 尝试结束时触发 Stop 事件，`hooks/hooks.json` 将其路由到 `stop-hook.sh`（`plugins/ralph-wiggum/hooks/hooks.json:3-10`）。
5. `stop-hook.sh` 读取状态 + transcript，判定是否完成/到达上限；未完成则递增 `iteration` 并返回 block 决策，把原 prompt 作为下一轮输入（`plugins/ralph-wiggum/hooks/stop-hook.sh:20-174`）。
6. 若用户执行 `/cancel-ralph`，状态文件被删除，后续 Stop hook 直接放行退出（`plugins/ralph-wiggum/commands/cancel-ralph.md:11-18` + `plugins/ralph-wiggum/hooks/stop-hook.sh:15-18`）。

### 2) 数据结构（状态文件）
状态文件：`.claude/ralph-loop.local.md`

结构为 “YAML frontmatter + markdown body” ：
- frontmatter 字段：
  - `active`（当前实现写入但 hook 未消费）
  - `iteration`（当前轮次）
  - `max_iterations`（0 表示无限）
  - `completion_promise`（`null` 或字符串）
  - `started_at`（UTC 时间）
  - 来源：`plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:140-147`
- body：原始 prompt 文本（`plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:149`）。

读取策略：
- frontmatter：`sed -n '/^---$/,/^---$/{...}'` + `grep '^key:'`（`plugins/ralph-wiggum/hooks/stop-hook.sh:21-25`）。
- body：`awk '/^---$/{i++; next} i>=2'`（`plugins/ralph-wiggum/hooks/stop-hook.sh:136`）。

### 3) 协议

A. Hook 输入协议（stdin JSON）
- `stop-hook.sh` 从 stdin 接收 Hook 输入，并读取 `transcript_path`（`plugins/ralph-wiggum/hooks/stop-hook.sh:9-10,57-58`）。
- `transcript_path` 属于 Hook 输入通用字段（`plugins/plugin-dev/skills/hook-development/SKILL.md:300-312`）。

B. Stop 决策输出协议
- Stop hook 需要输出：`decision`、`reason`、`systemMessage`（`plugins/plugin-dev/skills/hook-development/SKILL.md:202-208`）。
- Ralph 的实现：
```json
{
  "decision": "block",
  "reason": "<原始PROMPT>",
  "systemMessage": "🔄 Ralph iteration N ..."
}
```
- 代码位置：`plugins/ralph-wiggum/hooks/stop-hook.sh:167-174`。

C. 完成承诺协议
- 通过 assistant 最近一条文本中 `<promise>...</promise>` 提取声明值，并与 `completion_promise` 做字面相等比较（`=`，避免 glob）（`plugins/ralph-wiggum/hooks/stop-hook.sh:119-124`）。
- README 对该机制有用户级约束说明（exact string matching）（`plugins/ralph-wiggum/README.md:134`）。

### 4) 命令与参数

A. `/ralph-loop`
- 参数格式：`PROMPT [--max-iterations N] [--completion-promise TEXT]`（`plugins/ralph-wiggum/commands/ralph-loop.md:3`）。
- 输入校验：
  - `--max-iterations` 必须是非负整数（`plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:61-85`）。
  - `--completion-promise` 必须带值（`plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:87-103`）。
  - `PROMPT` 必填（`plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:115-128`）。

B. `/cancel-ralph`
- 检查状态文件是否存在，存在则读取迭代并删除（`plugins/ralph-wiggum/commands/cancel-ralph.md:11-18`）。

C. `/help`
- 文本解释命令，不执行工具（`plugins/ralph-wiggum/commands/help.md:1-3`）。

### 5) 错误处理与恢复策略
`stop-hook.sh` 采用“异常即停循环并清理状态文件”的恢复策略，避免坏状态无限阻塞：
- frontmatter 数字字段非法（`plugins/ralph-wiggum/hooks/stop-hook.sh:27-48`）。
- transcript 文件不存在/无 assistant 消息/JSON 解析失败/无文本内容（`plugins/ralph-wiggum/hooks/stop-hook.sh:60-112`）。
- prompt body 丢失（`plugins/ralph-wiggum/hooks/stop-hook.sh:138-150`）。

## 关键代码路径与文件引用

核心对象与上下游：
- 目标文档（研究对象）
  - `plugins/ralph-wiggum/README.md:1-179`
- 插件元数据（被插件加载器消费）
  - `plugins/ralph-wiggum/.claude-plugin/plugin.json:1-9`
- 命令入口（README 的用户操作入口落地）
  - `plugins/ralph-wiggum/commands/ralph-loop.md:1-18`
  - `plugins/ralph-wiggum/commands/cancel-ralph.md:1-18`
  - `plugins/ralph-wiggum/commands/help.md:1-126`
- 运行时脚本（核心被调用方）
  - `plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:1-203`
  - `plugins/ralph-wiggum/hooks/stop-hook.sh:1-177`
- Hook 注册配置
  - `plugins/ralph-wiggum/hooks/hooks.json:1-15`
- 插件系统上下文与规范
  - `plugins/README.md:26,49-60`
  - `plugins/plugin-dev/skills/hook-development/SKILL.md:62-80,202-208,300-312`

调用方/被调用方关系图（简化）：
- 调用方：用户 `/ralph-loop` -> `commands/ralph-loop.md`
- 被调用方：`scripts/setup-ralph-loop.sh` -> 写 `.claude/ralph-loop.local.md`
- 调用方：Claude Stop 事件 -> `hooks/hooks.json`
- 被调用方：`hooks/stop-hook.sh` -> 读 state + transcript -> 输出 Stop 决策 JSON
- 调用方：用户 `/cancel-ralph` -> `commands/cancel-ralph.md`
- 被调用方：删除 `.claude/ralph-loop.local.md`

## 依赖与外部交互

### 1) 运行时依赖（脚本级）
`stop-hook.sh` 依赖以下命令：
- `jq`：解析 hook 输入与 transcript、构建 JSON 输出（`plugins/ralph-wiggum/hooks/stop-hook.sh:58,90-95,167-174`）。
- `sed` / `grep` / `awk` / `tail`：frontmatter/body/最后 assistant 消息解析（`plugins/ralph-wiggum/hooks/stop-hook.sh:21-25,71,81,136,155`）。
- `perl`：跨行提取 `<promise>` 内容（`plugins/ralph-wiggum/hooks/stop-hook.sh:119`）。
- `mv`：迭代计数原子更新（`plugins/ralph-wiggum/hooks/stop-hook.sh:154-156`）。

`setup-ralph-loop.sh` 依赖：
- `date`：写 `started_at`（`plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:146`）。
- 标准 shell IO：创建 `.claude/ralph-loop.local.md`（`plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:131-150`）。

### 2) 与 Claude Code 运行时交互
- Hook 输入通过 stdin 注入 JSON（含 `transcript_path`）并由脚本消费（`plugins/ralph-wiggum/hooks/stop-hook.sh:9-10,57-58`）。
- Hook 输出通过 stdout JSON 驱动 Stop 行为（`decision:block`）并把 prompt 回灌（`plugins/ralph-wiggum/hooks/stop-hook.sh:167-174`）。

### 3) 文件系统交互
- 写入/更新/删除：`.claude/ralph-loop.local.md`（setup 和 stop/cancel 三者共享）。
- 读取：`$TRANSCRIPT_PATH` 指向的会话 transcript JSONL（`plugins/ralph-wiggum/hooks/stop-hook.sh:60-95`）。

### 4) 文档与外链交互
README 对外引用：
- 原始方法说明：`https://ghuntley.com/ralph/`（`plugins/ralph-wiggum/README.md:174`）。
- 第三方 orchestrator：`https://github.com/mikeyobrien/ralph-orchestrator`（`plugins/ralph-wiggum/README.md:175`）。

### 5) 测试与校验现状
该插件目录内未提供自动化测试（无 `test/spec/__tests__`）。
本次验证结果：
- `bash -n plugins/ralph-wiggum/hooks/stop-hook.sh` 通过。
- `bash -n plugins/ralph-wiggum/scripts/setup-ralph-loop.sh` 通过。
- `bash plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh plugins/ralph-wiggum/hooks/hooks.json` 失败：该校验脚本按“顶层事件”遍历，无法处理 plugin wrapper 格式（把 `description/hooks` 误当事件并在字符串上索引）（`plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh:43,65-70`），而 hook-development 文档本身又声明插件格式应使用 wrapper（`plugins/plugin-dev/skills/hook-development/SKILL.md:64-80`）。

## 风险、边界与改进建议

### 风险 1：文档与实现路径不一致（`/help`）
- 现状：`help.md` 写的是 `.claude/.ralph-loop.local.md`（多一个点）（`plugins/ralph-wiggum/commands/help.md:49,69`）。
- 实现：setup/stop/cancel 一致使用 `.claude/ralph-loop.local.md`（`plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:140`，`plugins/ralph-wiggum/hooks/stop-hook.sh:13`，`plugins/ralph-wiggum/commands/cancel-ralph.md:3`）。
- 影响：用户按帮助文档排障可能定位错误文件。
- 建议：统一修正文档路径，避免运行手册漂移。

### 风险 2：README 与脚本对“手动停止”表述不一致
- 现状：setup 帮助/提示中强调“cannot be stopped manually”（`plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:50,166-167`）。
- 同时：插件公开了 `/cancel-ralph`（`plugins/ralph-wiggum/README.md:63-70`）。
- 影响：用户对退出策略理解冲突，可能误判不可控。
- 建议：将“无法手动停止”改为“默认不会自动停止，除非 max-iterations/promise 或执行 /cancel-ralph”。

### 风险 3：Stop hook transcript 解析对格式变化敏感
- 现状：依赖 `grep '"role":"assistant"'` + 单行 JSONL 假设（`plugins/ralph-wiggum/hooks/stop-hook.sh:71,81`）。
- 边界：若 transcript 序列化格式变化（字段顺序、空白、换行对象），可能触发误判并清理 state 停循环。
- 建议：改为 `jq -s` 或逐行 `jq` 解析 role 字段，降低对文本模式匹配的耦合。

### 风险 4：状态字段解析依赖严格 key 格式
- 现状：frontmatter 解析使用 `grep '^key:'`，字段缺失即进入“corrupted”分支并删除状态（`plugins/ralph-wiggum/hooks/stop-hook.sh:22-48`）。
- 边界：人工编辑、格式化工具改写、引号转义异常都可能导致 loop 被动终止。
- 建议：引入更稳健 frontmatter 解析（例如 yq，或统一脚本库函数），并对“字段缺失”给出更具体诊断。

### 风险 5：`completion_promise` YAML 引号转义不完整
- 现状：写入时直接拼接 `"$COMPLETION_PROMISE"`（`plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:134-145`）。
- 边界：如果 promise 自身包含双引号或反斜杠，可能生成不规范 YAML 值。
- 建议：写入前对引号/反斜杠进行转义，或改为单引号策略并处理单引号 escaping。

### 风险 6：工具链内部校验脚本与规范不一致
- 现状：hook-development 文档宣称插件 `hooks/hooks.json` 使用 wrapper（`plugins/plugin-dev/skills/hook-development/SKILL.md:64-80`），但 `validate-hook-schema.sh` 仅支持顶层事件，导致对 Ralph 配置误报失败（`plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh:43,65-70`）。
- 影响：开发者会得到“配置错”假阳性，降低工具信任度。
- 建议：先探测 `{"hooks": ...}` 包装并切换 jq 查询根路径，兼容 plugin 与 settings 两种格式。

### 边界总结
- 该插件适合“可验证、可迭代”的任务，不适合高主观决策任务（`plugins/ralph-wiggum/README.md:152-165`）。
- 若不设置 `--max-iterations` 且不触发 completion promise，循环理论上可无限运行（`plugins/ralph-wiggum/README.md:121-135` + `plugins/ralph-wiggum/hooks/stop-hook.sh:159-163`）。

