# DIR `plugins/ralph-wiggum/hooks` 研究文档

## 场景与职责

`plugins/ralph-wiggum/hooks` 是 Ralph Wiggum 插件的运行时控制核心。它不负责“如何创建循环”，而负责在 Claude 会话准备结束时执行最终仲裁：允许停止，还是阻断停止并继续下一轮。

目录内仅两项文件：

- `plugins/ralph-wiggum/hooks/hooks.json`：Hook 注册入口（Stop 事件）。
- `plugins/ralph-wiggum/hooks/stop-hook.sh`：Stop 事件命令型 Hook 实现。

在完整插件链路中的职责边界：

1. 上游调用方
- Claude Code Hook 运行时在 Stop 事件触发时按配置调用该脚本（`plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:23-27`）。
- `plugins/ralph-wiggum/scripts/setup-ralph-loop.sh` 提前创建状态文件，激活本目录逻辑（`plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:130-150`）。

2. 下游被调用方
- 读取会话 transcript（通过 hook stdin 的 `transcript_path`）并解析最后一条 assistant 文本（`plugins/ralph-wiggum/hooks/stop-hook.sh:57-95`）。
- 读写与删除 `.claude/ralph-loop.local.md` 状态文件（`plugins/ralph-wiggum/hooks/stop-hook.sh:13,35,46,53,65,76,85,103,110,125,148,155-156`）。

3. 场景定位
- 对应插件说明中的“Stop - Intercepts exit attempts to continue iteration”（`plugins/README.md:26`）。
- 对应 Ralph 插件 README 的“在当前 session 内形成自引用循环”（`plugins/ralph-wiggum/README.md:13-27`）。

## 功能点目的

1. Stop Hook 注册与装配
- 目的：把 `stop-hook.sh` 绑定到 Stop 事件。
- 实现：`hooks.json` 使用 plugin wrapper 结构：`{"description":..., "hooks": {"Stop": [...]}}`，命令指向 `${CLAUDE_PLUGIN_ROOT}/hooks/stop-hook.sh`（`plugins/ralph-wiggum/hooks/hooks.json:1-15`，`plugins/plugin-dev/skills/hook-development/SKILL.md:62-80`）。

2. 循环激活判定
- 目的：避免无状态时误拦截所有退出。
- 实现：若 `.claude/ralph-loop.local.md` 不存在立即 `exit 0` 放行（`plugins/ralph-wiggum/hooks/stop-hook.sh:13-18`）。

3. 停止条件判定
- 目的：确保循环可控收敛。
- 实现：
  - 达到 `max_iterations` 时删除状态并结束（`plugins/ralph-wiggum/hooks/stop-hook.sh:50-55`）。
  - 命中 `<promise>...</promise>` 且与 `completion_promise` 字面完全相等时删除状态并结束（`plugins/ralph-wiggum/hooks/stop-hook.sh:114-127`）。

4. 续跑注入
- 目的：在未完成时阻断停止并把同一 prompt 回灌。
- 实现：`iteration+1` 回写状态文件后输出 Stop 决策 JSON：`decision=block`、`reason=<原始prompt>`、`systemMessage=<迭代提示>`（`plugins/ralph-wiggum/hooks/stop-hook.sh:131-174`）。

5. 容错与自愈
- 目的：状态或 transcript 异常时避免僵死循环。
- 实现：字段非法、transcript 缺失/无 assistant/JSON 解析失败/无文本内容时，打印告警并删除状态文件后退出（`plugins/ralph-wiggum/hooks/stop-hook.sh:27-48,60-112,138-150`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 关键流程

```text
Stop 事件触发
  -> hooks.json 命中 Stop -> 执行 stop-hook.sh
  -> 读取 stdin JSON (HOOK_INPUT)
  -> 若无 .claude/ralph-loop.local.md: exit 0 (放行)
  -> 解析 frontmatter: iteration/max_iterations/completion_promise
  -> 校验数字字段; 检查 max_iterations
  -> 从 HOOK_INPUT 取 transcript_path 并读取最后 assistant 文本
  -> 若 completion_promise 命中 <promise>...</promise>: 删除状态并 exit 0
  -> 否则 iteration+1 写回状态
  -> 输出 {decision:block, reason:原prompt, systemMessage:迭代提示}
```

### 2) 核心数据结构

1. Hook 配置结构（plugin wrapper）

```json
{
  "description": "...",
  "hooks": {
    "Stop": [
      {
        "hooks": [
          { "type": "command", "command": "${CLAUDE_PLUGIN_ROOT}/hooks/stop-hook.sh" }
        ]
      }
    ]
  }
}
```

来源：`plugins/ralph-wiggum/hooks/hooks.json:1-15`。

2. 运行时状态文件（由 setup 脚本创建，本目录消费）
- 路径：`.claude/ralph-loop.local.md`
- frontmatter 字段：`iteration`、`max_iterations`、`completion_promise`、`started_at`（写入见 `plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:140-147`）
- body：原始 prompt（`plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:149`）

3. Stop 输入/输出协议
- 输入：stdin JSON，至少读取 `transcript_path`（`plugins/ralph-wiggum/hooks/stop-hook.sh:9-10,57-58`）。
- 输出（阻断时）：

```json
{
  "decision": "block",
  "reason": "<prompt>",
  "systemMessage": "<iteration message>"
}
```

协议形态与 hook-development 文档一致（`plugins/plugin-dev/skills/hook-development/SKILL.md:181-208`）。

### 3) 关键命令与文本处理实现

1. frontmatter 解析
- `sed -n '/^---$/,/^---$/{ /^---$/d; p; }'` 取 frontmatter（`plugins/ralph-wiggum/hooks/stop-hook.sh:21`）。
- `grep '^iteration:'|sed ...` 等提取字段（`plugins/ralph-wiggum/hooks/stop-hook.sh:22-25`）。

2. transcript 提取
- 先 `grep -q '"role":"assistant"'` 做存在性检查（`plugins/ralph-wiggum/hooks/stop-hook.sh:71-78`）。
- 再 `grep ... | tail -1` 取最后一条 assistant 记录（`plugins/ralph-wiggum/hooks/stop-hook.sh:81`）。
- 用 `jq` 提取 `.message.content[]` 中 `type=="text"` 并 join（`plugins/ralph-wiggum/hooks/stop-hook.sh:90-95`）。

3. promise 匹配
- `perl -0777` 从多行文本提取 `<promise>...</promise>` 内文并规范空白（`plugins/ralph-wiggum/hooks/stop-hook.sh:116-123`）。
- 与 frontmatter 中 `completion_promise` 做字面相等比较（`=`，非 glob）（`plugins/ralph-wiggum/hooks/stop-hook.sh:121-124`）。

4. 状态更新
- `awk '/^---$/{i++; next} i>=2'` 提取 body prompt（`plugins/ralph-wiggum/hooks/stop-hook.sh:133-137`）。
- `sed` 改写 `iteration` 到临时文件，再 `mv` 原子替换（`plugins/ralph-wiggum/hooks/stop-hook.sh:152-156`）。

### 4) 本次实际验证（非占位）

在仓库内执行了以下校验：

1. 语法检查
- `bash -n plugins/ralph-wiggum/hooks/stop-hook.sh`：通过。
- `bash -n plugins/ralph-wiggum/scripts/setup-ralph-loop.sh`：通过。

2. Hook schema 脚本验证
- `bash plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh plugins/ralph-wiggum/hooks/hooks.json`
- 结果：对 `description/hooks` 发出 unknown event，随后 `jq` 报错 `Cannot index string with number`，退出码为 `5`。
- 原因：该校验脚本按“顶层即事件列表”遍历（`plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh:41-67`），与插件 wrapper 格式（`plugins/plugin-dev/skills/hook-development/SKILL.md:62-80`）不一致。

3. Stop Hook 分支模拟（临时目录）
- 无状态文件：脚本静默 `exit 0`（放行）。
- 有状态且未完成：输出 `decision:block` JSON，并把 `iteration: 1` 更新为 `iteration: 2`。
- transcript 含 `<promise>DONE</promise>` 且匹配：输出完成提示并删除状态文件。

## 关键代码路径与文件引用

### 调用方（Callers）

1. Claude Code Hook 运行时
- 依据 Stop 事件调用：`plugins/ralph-wiggum/hooks/hooks.json:4-11`
- 组件注册生命周期参考：`plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:7-17,23-27`

2. Ralph 循环启动链路（间接调用方）
- 命令入口：`plugins/ralph-wiggum/commands/ralph-loop.md:1-14`
- 状态文件创建：`plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:130-150`

### 被调用方（Callees）

1. 文件与状态
- `.claude/ralph-loop.local.md`（读取/更新/删除）
- transcript 文件（路径来自 stdin JSON 的 `transcript_path`）

2. Shell 工具链
- `jq`, `perl`, `sed`, `awk`, `grep`, `tail`, `mv`, `rm`

3. 相关命令
- `/cancel-ralph` 删除同一状态文件（`plugins/ralph-wiggum/commands/cancel-ralph.md:3,11-18`）

### 配置、脚本、文档与测试上下文

1. 配置
- 插件 hooks 配置：`plugins/ralph-wiggum/hooks/hooks.json`
- 插件元数据（默认 hooks 路径语义背景）：`plugins/ralph-wiggum/.claude-plugin/plugin.json:1-9` + `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:259-263`

2. 脚本
- 主逻辑：`plugins/ralph-wiggum/hooks/stop-hook.sh`
- 上游初始化：`plugins/ralph-wiggum/scripts/setup-ralph-loop.sh`
- 校验/测试工具链：
  - `plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh`
  - `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh`

3. 文档
- 插件说明：`plugins/ralph-wiggum/README.md:13-39,50-71`
- 插件总览：`plugins/README.md:26`
- 真实案例参考：`plugins/plugin-dev/skills/plugin-settings/references/real-world-examples.md:129-227`

4. 测试现状
- `plugins/ralph-wiggum/hooks` 目录内无自动化测试文件；当前以脚本静态检查和手工模拟验证为主。

## 依赖与外部交互

1. 运行时依赖
- Shell: `bash`（`set -euo pipefail`）
- JSON 处理: `jq`
- 文本抽取: `sed/awk/grep/tail/perl`
- 文件操作: `mv/rm`

2. Claude 运行时交互
- 输入：Stop 事件 stdin JSON（`transcript_path` 为关键字段）。
- 输出：Stop 决策 JSON（继续循环时 `decision:block`）。

3. 文件系统交互
- 读：`.claude/ralph-loop.local.md`、`$TRANSCRIPT_PATH`
- 写：状态文件 iteration 更新（临时文件 + 原子替换）
- 删：满足停止条件或异常中止时清理状态文件

4. 环境变量/路径约束
- `hooks.json` 使用 `${CLAUDE_PLUGIN_ROOT}` 提供跨安装路径可移植性（`plugins/ralph-wiggum/hooks/hooks.json:9`）。

5. 外部网络
- 无直接 HTTP/API 调用；仅本地脚本与文件交互。

## 风险、边界与改进建议

1. 校验工具与配置格式错位（高优先级）
- 现象：`validate-hook-schema.sh` 不能直接校验插件 wrapper 格式，且会异常退出（本次实测 exit 5）。
- 影响：团队可能误判 hooks 配置质量，且 plugin-validator 建议链路会被误导（`plugins/plugin-dev/agents/plugin-validator.md:107-113`）。
- 建议：校验脚本先检测顶层是否存在 `hooks`，存在则改对 `.hooks` 节点校验。

2. `matcher` 约束与仓库实践不一致（中优先级）
- 现象：校验规则要求每个事件项必须有 `matcher`（`validate-hook-schema.sh:69-75`，`plugin-validator.md:112`），但 ralph 及多插件实际省略 matcher（如 explanatory、learning、hookify）。
- 影响：规则与实践偏离，导致误报和规范歧义。
- 建议：统一规范：要么文档明确 matcher 可选并给默认 `*`，要么插件补齐 matcher 并统一迁移。

3. transcript 解析对日志序列化格式耦合较强（中优先级）
- 现状：依赖 `grep '"role":"assistant"'` 文本匹配（`stop-hook.sh:71,81`）。
- 边界：若 JSON 行格式变化（空格、字段顺序、压缩策略），可能误判“无 assistant 消息”并提前停止。
- 建议：改为 `jq -c 'select(.role=="assistant")'` 管道过滤，减少字符串层脆弱性。

4. frontmatter 解析对手工编辑容错有限（中优先级）
- 现状：通过 `grep '^key:'` 提取，字段缺失/格式漂移会触发“corrupted”并删除状态（`stop-hook.sh:27-48`）。
- 影响：用户手工调整 state 时易导致循环提前终止。
- 建议：增加“仅告警不删除”安全模式或更稳健 YAML 解析器（至少在缺字段时给修复建议再退出）。

5. completion promise 值来源的转义完整性依赖 setup 侧（中优先级）
- 现状：hooks 侧按字面匹配，若 setup 写入的 YAML 字符串转义不完整（引号/反斜杠/换行），可能造成误匹配。
- 影响：出现“看起来输出了 promise 但不停止”或反向误停。
- 建议：在 setup 侧用可靠序列化方式（如 `jq -R`）写入 `completion_promise`。

6. 自动化测试缺失（中优先级）
- 现状：无 hooks 目录下的可重复测试脚本/fixture。
- 建议：新增最小化回归测试，覆盖：
  - 无状态放行。
  - 命中 `max_iterations` 停止。
  - promise 命中停止。
  - transcript 缺失/损坏分支。
  - 非数字 `iteration/max_iterations` 分支。

7. 输出通道策略可进一步明确（低优先级）
- 现状：部分“正常结束”提示输出到 stdout（如 max iterations/promise 命中）。
- 边界：在不同 Hook 运行时 UI 呈现中可能造成噪音。
- 建议：统一约定“控制消息走 `systemMessage` / stderr，stdout 仅保留协议 JSON”，提高行为一致性。
