# plugins/ralph-wiggum 研究

## 场景与职责

`plugins/ralph-wiggum` 是一个“会话内自循环”插件，目标是让 Claude 在同一任务上迭代工作，直到满足退出条件。它通过 Stop Hook 拦截会话结束，并把原始 prompt 再次喂回模型，实现 Ralph Wiggum 技术中的 while-loop 体验。

核心职责分层如下：

- 插件元数据声明：`.claude-plugin/plugin.json` 提供名称、版本、描述与作者信息（`plugins/ralph-wiggum/.claude-plugin/plugin.json:1-9`）。
- 命令入口：
  - `/ralph-loop` 初始化循环状态（`plugins/ralph-wiggum/commands/ralph-loop.md:1-18`）。
  - `/cancel-ralph` 人工终止循环（`plugins/ralph-wiggum/commands/cancel-ralph.md:1-18`）。
  - `/help` 说明原理与使用方式（`plugins/ralph-wiggum/commands/help.md:1-126`）。
- Hook 执行：`hooks/hooks.json` 注册 Stop 事件命令型 hook（`plugins/ralph-wiggum/hooks/hooks.json:1-15`）。
- 状态机实现：
  - 初始化脚本写入 `.claude/ralph-loop.local.md`（`plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:130-150`）。
  - Stop Hook 读取状态与 transcript，决定“允许停止”或“阻断并继续”（`plugins/ralph-wiggum/hooks/stop-hook.sh:13-174`）。

在仓库级上下文中，插件目录索引明确把它定义为“自引用迭代循环插件”，包含 `/ralph-loop`、`/cancel-ralph` 和 Stop Hook（`plugins/README.md:26`）。`plugin-dev` 文档将该插件作为“状态文件 + Hook 控制流”的真实案例（`plugins/plugin-dev/README.md:123`、`plugins/plugin-dev/skills/plugin-settings/references/real-world-examples.md:129-264`）。

## 功能点目的

1. 启动循环（/ralph-loop）
- 目的：把一次任务提示词转化为“可恢复、可计数、可终止”的循环状态。
- 机制：解析命令参数后生成 `.claude/ralph-loop.local.md`，写入 frontmatter（`iteration/max_iterations/completion_promise/started_at`）+ body（原始 prompt）（`plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:140-150`）。

2. 自动续跑（Stop Hook）
- 目的：在模型尝试停止时，统一判断是否应继续。
- 机制：
  - 若未激活循环（状态文件不存在）直接放行（`plugins/ralph-wiggum/hooks/stop-hook.sh:15-18`）。
  - 达到最大迭代或命中完成承诺 `<promise>...</promise>` 时结束并清理状态文件（`plugins/ralph-wiggum/hooks/stop-hook.sh:50-55`、`114-127`）。
  - 否则迭代计数 +1，并输出 `{"decision":"block"...}` 阻断退出、注入下一轮 prompt（`plugins/ralph-wiggum/hooks/stop-hook.sh:131-174`）。

3. 人工取消（/cancel-ralph）
- 目的：提供人工兜底终止路径。
- 机制：检查状态文件是否存在；存在则读取当前 iteration 并删除文件（`plugins/ralph-wiggum/commands/cancel-ralph.md:11-18`）。

4. 使用指导（README/help）
- 目的：指导用户编写可收敛 prompt，并强调 `--max-iterations` 与 `--completion-promise` 的停机策略（`plugins/ralph-wiggum/README.md:72-135`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 关键流程

```text
用户输入 /ralph-loop <PROMPT> [--max-iterations N] [--completion-promise TEXT]
  -> commands/ralph-loop.md 调 setup-ralph-loop.sh
  -> 写入 .claude/ralph-loop.local.md (iteration=1 + prompt)
  -> 用户/模型执行任务
  -> 模型触发 Stop 事件
  -> hooks/stop-hook.sh 读取状态 + transcript
     - 若完成: 删除状态，允许结束
     - 若未完成: iteration++，输出 decision=block + reason=原始prompt
  -> 进入下一轮
```

### 2) 状态数据结构

状态文件：`.claude/ralph-loop.local.md`

- Frontmatter 字段（写入位置：`setup-ralph-loop.sh:141-147`）
  - `active: true`
  - `iteration: 1`
  - `max_iterations: <n|0>`（0 表示无限）
  - `completion_promise: <"text"|null>`
  - `started_at: <UTC ISO 时间>`
- Body：用户给定的原始 prompt（`setup-ralph-loop.sh:149`）。

Hook 侧解析方式：
- `sed` 提取 frontmatter（`stop-hook.sh:21`）
- `grep + sed` 抽取字段（`stop-hook.sh:22-25`）
- `awk` 读取正文 prompt（`stop-hook.sh:136`）
- `sed + mv` 原子替换 iteration（`stop-hook.sh:154-156`）

### 3) Hook 协议与 transcript 读取

输入协议（Stop 事件）：脚本从 stdin 读取 JSON，并取 `.transcript_path`（`stop-hook.sh:9-10`、`57-58`）。

输出协议（Stop 决策）：脚本输出 JSON：
- `decision: "approve|block"`
- `reason: <文本>`
- `systemMessage: <文本>`

本插件在继续循环时输出 `decision=block`，`reason` 直接放原始 prompt（`stop-hook.sh:165-174`）。这与 hook 开发文档的 Stop 输出结构一致（`plugins/plugin-dev/skills/hook-development/SKILL.md:202-209`）。

transcript 处理：
- 通过 `grep '"role":"assistant"' | tail -1` 取最后一条 assistant 记录（`stop-hook.sh:71-82`）。
- 使用 `jq` 读取 `.message.content` 中 `type=text` 的内容并拼接（`stop-hook.sh:90-95`）。
- 使用 `perl -0777` 从输出中抽取 `<promise>...</promise>`，支持多行匹配（`stop-hook.sh:116-123`）。

### 4) 命令级实现细节

- `/ralph-loop` 命令只允许调用 setup 脚本（`commands/ralph-loop.md:4`），脚本支持：
  - `--max-iterations <n>`（非负整数校验，`setup-ralph-loop.sh:61-86`）
  - `--completion-promise <text>`（`setup-ralph-loop.sh:87-103`）
  - `-h/--help`（`setup-ralph-loop.sh:16-60`）
- `/cancel-ralph` 通过 `test -f` + `rm` 删除状态文件（`commands/cancel-ralph.md:3,11-18`）。

## 关键代码路径与文件引用

### 调用方（Callers）

- 用户 Slash 命令入口：
  - `plugins/ralph-wiggum/commands/ralph-loop.md:1-18`
  - `plugins/ralph-wiggum/commands/cancel-ralph.md:1-18`
  - `plugins/ralph-wiggum/commands/help.md:1-126`
- Claude Code Hook 事件入口：
  - `plugins/ralph-wiggum/hooks/hooks.json:1-15`（注册 Stop -> `stop-hook.sh`）

### 被调用方（Callees）

- 初始化脚本：`plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:1-203`
- 循环控制脚本：`plugins/ralph-wiggum/hooks/stop-hook.sh:1-177`
- 本地状态文件：`.claude/ralph-loop.local.md`（由 setup 创建，被 stop-hook/cancel-ralph 消费）

### 配置/文档上下文

- 插件元数据：`plugins/ralph-wiggum/.claude-plugin/plugin.json:1-9`
- 插件对外说明：`plugins/ralph-wiggum/README.md:1-179`
- 仓库插件总览：`plugins/README.md:13-27`
- plugin-dev 对 Ralph 的模式化说明：
  - `plugins/plugin-dev/README.md:123`
  - `plugins/plugin-dev/skills/plugin-settings/SKILL.md:452-472`
  - `plugins/plugin-dev/skills/plugin-settings/references/real-world-examples.md:129-264`

### 测试与验证现状

- `plugins/ralph-wiggum` 目录下无 `test/spec` 文件。
- 本次静态检查：
  - `bash -n plugins/ralph-wiggum/hooks/stop-hook.sh` 通过。
  - `bash -n plugins/ralph-wiggum/scripts/setup-ralph-loop.sh` 通过。
- 使用 plugin-dev 验证脚本检查 `hooks/hooks.json` 时失败：
  - 命令：`bash plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh plugins/ralph-wiggum/hooks/hooks.json`
  - 结果：脚本按“根级事件结构”解析，遇到插件 wrapper（`description/hooks`）后在 jq 索引阶段报错。

## 依赖与外部交互

1. 运行时命令依赖（stop-hook）
- 必需工具：`bash`, `jq`, `perl`, `sed`, `awk`, `grep`, `tail`, `mv`, `rm`。
- 关键依赖点：
  - `jq`：读取 hook 输入和 transcript JSON（`stop-hook.sh:58,90-95,167-174`）。
  - `perl`：多行 promise 提取（`stop-hook.sh:119`）。

2. 运行时命令依赖（setup）
- 必需工具：`bash`, `date`, `mkdir`, `cat`。
- 关键依赖点：UTC 时间戳写入 `started_at`（`setup-ralph-loop.sh:146`）。

3. Claude Code 运行时交互
- 输入：Stop hook stdin JSON（至少包含 `transcript_path`，可参考测试脚本样例 `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh:59-69`）。
- 输出：Stop 决策 JSON（`stop-hook.sh:170-174`）。
- 文件交互：读写 `.claude/ralph-loop.local.md`，读取 transcript 文件路径（`stop-hook.sh:13,57-67`）。

4. 外部网络/服务
- 插件本身不直接访问网络，不调用外部 API。

## 风险、边界与改进建议

1. 文档路径不一致
- `commands/help.md` 使用 `.claude/.ralph-loop.local.md`（`help.md:49,69`），而真实实现与其他命令均使用 `.claude/ralph-loop.local.md`。
- 影响：用户按 help 操作可能定位错误文件。
- 建议：统一文档路径，优先以脚本真实路径为准。

2. “无法手动停止”文案与实际能力冲突
- setup 输出强调“cannot be stopped manually”（`setup-ralph-loop.sh:166-167`），但插件同时提供 `/cancel-ralph` 人工停止。
- 影响：认知冲突，误导操作策略。
- 建议：改为“默认会持续运行，除非达到停止条件或执行 /cancel-ralph”。

3. transcript 解析对格式假设较强
- 当前依赖 `grep '"role":"assistant"'` 精确字符串匹配（`stop-hook.sh:71,81`）。
- 边界：若 transcript 序列化格式改变（例如空格、字段顺序变化），可能提取失败并提前停止循环。
- 建议：改为逐行 `jq` 过滤 `role=="assistant"`，避免文本匹配脆弱性。

4. 依赖缺失时缺少显式预检
- `jq/perl` 缺失会导致脚本在运行时失败（当前未做 `command -v` 预检查）。
- 建议：在 setup 或 stop-hook 启动阶段进行依赖检测并输出明确错误。

5. 验证工具链与插件 hooks.json 格式存在错位
- plugin-dev 的 `validate-hook-schema.sh` 对插件 wrapper 结构处理不兼容，无法直接用于当前 `hooks/hooks.json`。
- 影响：开发者可能错误判断 hooks 配置质量。
- 建议：增强验证脚本以同时支持“插件 wrapper 格式”和“settings 直连格式”，或增加专用入口参数。

6. 自动化测试缺失
- 当前仅有脚本与文档，无插件级集成测试（例如：模拟 Stop 输入 + transcript 样本，断言 `approve/block` 行为）。
- 建议：新增最小 smoke tests，覆盖：
  - 达到 `max_iterations` 自动停止。
  - promise 命中停止。
  - transcript 缺失/损坏分支。
  - state 文件异常分支（iteration/max_iterations 非数字）。
