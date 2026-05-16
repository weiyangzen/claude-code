# FILE 研究：plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh

## 场景与职责

`test-hook.sh` 是 hook 开发工具链中的“离线执行器”。它把 Claude Code 运行时的 hook 输入协议和环境变量最小化复刻到本地 shell，帮助开发者在不启动完整会话的情况下验证脚本行为。

职责边界：

- 负责：样例输入生成、环境注入、超时执行、退出码解释、输出展示
- 不负责：`hooks.json` 结构合法性（由 `validate-hook-schema.sh` 负责）
- 不负责：静态质量检查（由 `hook-linter.sh` 负责）

调用入口主要来自：

- `plugins/plugin-dev/skills/hook-development/scripts/README.md:29-53,104-113`
- `plugins/plugin-dev/skills/hook-development/SKILL.md:707-709`
- `plugins/plugin-dev/commands/create-plugin.md:208,259`

## 功能点目的

核心功能点：

1. `--create-sample <event>`：快速产出可回放的 stdin JSON 样本（`25-100`）。
2. 参数化测试：支持 `-v`、`-t`，分别用于可观测性和超时边界（`102-127`）。
3. 运行前校验：检查 hook 脚本和输入文件存在，输入 JSON 可解析（`138-158`）。
4. 模拟运行环境：自动设置 `CLAUDE_PROJECT_DIR`、`CLAUDE_PLUGIN_ROOT`、`CLAUDE_ENV_FILE`（`170-174`）。
5. 结果判读：按 exit code 给出 approve/block/timeout/异常解释（`205-218`）。

目的：把“难以复现的运行时问题”前置到“可重复执行的本地命令”。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 输入样例构造

`create_sample()` 按事件输出 JSON：

- `PreToolUse`：含 `tool_name/tool_input`（`30-45`）
- `PostToolUse`：含 `tool_result`（`46-58`）
- `Stop|SubagentStop`：含 `reason`（`59-70`）
- `UserPromptSubmit`：含 `user_prompt`（`71-82`）
- `SessionStart|SessionEnd`：基础会话字段（`83-94`）

这套样例对齐了 `SKILL.md` 的 Hook Input Format（`SKILL.md:300-320`）。

### 2) 执行模型

执行命令：

```bash
timeout "$TIMEOUT" bash -c "cat '$TEST_INPUT' | $HOOK_SCRIPT"
```

对应实现：`test-hook.sh:189-191`。

关键点：

- `set +e` 捕获被测脚本退出码（`189-192`）
- 计时使用 `date +%s` 前后差（`187,194-195`）
- 输出同时捕获 stdout/stderr（`2>&1`）

### 3) 协议与判定

- 输入协议：stdin JSON
- 环境协议：`CLAUDE_PROJECT_DIR / CLAUDE_PLUGIN_ROOT / CLAUDE_ENV_FILE`
- 退出码语义：
  - `0`：通过
  - `2`：阻断/拒绝
  - `124`：超时（`timeout` 返回）
  - 其他：异常
- 脚本总体判定：仅 `0/2` 视为“测试成功”，其余退出 1（`246-252`）

### 4) 实测补充

- `--create-sample SessionEnd` 输出的 `hook_event_name` 实际为 `SessionStart`。
- `--create-sample SubagentStop` 输出的 `hook_event_name` 实际为 `Stop`。

实测命令：

```bash
bash plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh --create-sample SessionEnd | jq -r '.hook_event_name'
bash plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh --create-sample SubagentStop | jq -r '.hook_event_name'
```

输出分别为 `SessionStart`、`Stop`。

## 关键代码路径与文件引用

核心实现：

- `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh:1-252`

协议与文档上下文：

- `plugins/plugin-dev/skills/hook-development/SKILL.md:300-329`（输入字段与环境变量）
- `plugins/plugin-dev/skills/hook-development/scripts/README.md:29-61,104-113`
- `plugins/plugin-dev/README.md:273-284`
- `plugins/plugin-dev/commands/create-plugin.md:208,259`

可直接联调对象：

- `plugins/plugin-dev/skills/hook-development/examples/validate-write.sh`
- `plugins/plugin-dev/skills/hook-development/examples/validate-bash.sh`
- `plugins/plugin-dev/skills/hook-development/examples/load-context.sh`

## 依赖与外部交互

运行依赖：

- `bash`
- `jq`
- `timeout`
- `date`

外部交互：

- 读取测试输入 JSON 文件
- 通过管道把 JSON 喂给 hook 脚本
- 读取/删除 `CLAUDE_ENV_FILE`（`236-241`）

无网络请求。

## 风险、边界与改进建议

1. 样例事件名映射有误。
- `SessionEnd`、`SubagentStop` 被映射成其他事件名，可能误导测试结论。

2. 执行命令通过 `bash -c` 拼接字符串，路径边界较脆弱。
- `HOOK_SCRIPT` 在非可执行场景会被改写为字符串（`146`），若文件名含空格/特殊字符，命令拼接可读性和稳健性下降。

3. 输出 JSON 未作为硬约束。
- 脚本只在“可解析时美化输出”（`227-230`），不会因非 JSON 输出而失败；对依赖结构化输出的 hook 测试不够严格。

4. 参数健壮性可加强。
- `--create-sample` 缺参时在 `set -u` 下会触发未绑定参数错误。

建议：

- 修正 `create_sample()` 中 `SessionEnd/SubagentStop` 的 `hook_event_name`。
- 用数组执行替代字符串拼接，例如 `cmd=(bash "$HOOK_SCRIPT")` + stdin 重定向，减少 shell 解析歧义。
- 增加 `--strict-json-output` 选项，对需要 JSON 输出的 hook 做硬校验。
- 在 `--create-sample` 分支加入参数个数检查并给出友好错误信息。
