# FILE `plugins/ralph-wiggum/hooks/stop-hook.sh` 研究文档

## 场景与职责

`plugins/ralph-wiggum/hooks/stop-hook.sh` 是 Ralph Wiggum 插件的核心执行脚本，在 `Stop` 事件触发时决定当前会话是“允许结束”还是“阻断结束并继续下一轮”。

它在系统中的职责边界：

1. 调用方
- 由 `plugins/ralph-wiggum/hooks/hooks.json:4-11` 在 `Stop` 事件路由执行。
- 触发前置条件是用户通过 `/ralph-loop` 启动循环并生成状态文件（`plugins/ralph-wiggum/commands/ralph-loop.md:12-14`，`plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:140-150`）。

2. 被调用方
- 读取 `.claude/ralph-loop.local.md` 前置状态。
- 读取 `transcript_path` 指向的会话 JSONL 转录。
- 输出 Stop 协议 JSON 供 Claude Runtime 决策（block/allow）。

3. 业务目标
- 在未完成时回灌同一 prompt，形成“同任务反复迭代”。
- 在达到最大迭代数或命中 completion promise 时退出循环并清理状态。

## 功能点目的

1. 激活判定
- 仅当 `.claude/ralph-loop.local.md` 存在时接管 Stop；不存在则直接 `exit 0` 放行（`stop-hook.sh:13-18`）。

2. 状态完整性守护
- 校验 `iteration`、`max_iterations` 是数字，避免后续算术错误（`stop-hook.sh:27-48`）。
- 状态损坏时删除状态文件并结束循环，避免无限异常重试。

3. 收敛条件判定
- `max_iterations > 0 && iteration >= max_iterations` 时停止并清理状态（`stop-hook.sh:50-55`）。
- 当 assistant 输出中 `<promise>...</promise>` 与 `completion_promise` 字面一致时停止并清理状态（`stop-hook.sh:114-127`）。

4. 继续循环注入
- 从状态文件 frontmatter 之后抽出原始 prompt（`stop-hook.sh:133-137`）。
- 迭代计数 `+1` 写回状态文件（临时文件 + `mv` 原子替换，`stop-hook.sh:154-156`）。
- 输出 `decision:block`，把原 prompt 放在 `reason` 字段返还运行时（`stop-hook.sh:167-174`）。

5. 异常兜底
- transcript 缺失/无 assistant/JSON 解析失败/无文本等异常时，统一清理状态并放行，避免挂死（`stop-hook.sh:60-112`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 关键流程

```text
Stop 事件 -> 读取 stdin JSON -> 检查状态文件是否存在
  不存在: exit 0
  存在:
    解析 frontmatter(iteration/max_iterations/completion_promise)
    校验数值字段
    检查是否达到 max_iterations
    读取 transcript_path，提取最后 assistant 文本
    若 completion_promise 命中 <promise>...</promise>:
      删除状态并 exit 0
    否则:
      iteration+1 写回状态文件
      输出 {"decision":"block","reason":"<原prompt>","systemMessage":"<迭代提示>"}
      exit 0
```

### 2) 数据结构

1. 状态文件（由 setup 脚本写入）
- 文件：`.claude/ralph-loop.local.md`
- frontmatter 字段：`active`, `iteration`, `max_iterations`, `completion_promise`, `started_at`（`setup-ralph-loop.sh:141-147`）
- body：原始 prompt（`setup-ralph-loop.sh:149`）

2. Hook 输入（stdin JSON）
- 脚本实际读取 `transcript_path` 字段（`stop-hook.sh:58`）。
- Stop 事件通常还带 `reason` 字段（`test-hook.sh:66-68`），但本脚本未使用。

3. Hook 输出（仅续跑时）

```json
{
  "decision": "block",
  "reason": "<原始 prompt 文本>",
  "systemMessage": "🔄 Ralph iteration N | ..."
}
```

### 3) 关键命令与实现细节

1. frontmatter 解析
- `sed -n '/^---$/,/^---$/{ /^---$/d; p; }'` 提取 frontmatter（`stop-hook.sh:21`）。
- `grep '^iteration:'` 等抓取字段（`stop-hook.sh:22-25`）。

2. transcript 提取
- 先用 `grep -q '"role":"assistant"'` 做存在性检查（`stop-hook.sh:71-78`）。
- 再 `grep ... | tail -1` 取最后一条 assistant 记录（`stop-hook.sh:81`）。
- 使用 `jq` 从 `.message.content[]` 里筛 `type=="text"` 并拼接文本（`stop-hook.sh:90-95`）。

3. promise 匹配策略
- `perl -0777` 多行提取首个 `<promise>...</promise>`，并归一化空白（`stop-hook.sh:119`）。
- 用 `=` 做字面相等比较，避免 `==` 的 glob 模式匹配副作用（`stop-hook.sh:121-123`）。

4. prompt 抽取与状态更新
- `awk '/^---$/{i++; next} i>=2'` 提取 frontmatter 之后全部文本（`stop-hook.sh:136`）。
- `sed` 改写 `iteration` 到临时文件，再 `mv` 覆盖（`stop-hook.sh:154-156`）。

### 4) 本次实测（非模板）

执行命令：

```bash
bash -n plugins/ralph-wiggum/hooks/stop-hook.sh
bash -n plugins/ralph-wiggum/scripts/setup-ralph-loop.sh
```

结果：语法检查通过。

临时目录模拟 Stop 输入三种分支：

1. 无状态文件
- 结果：`exit 0`，无输出（放行）。

2. 有状态且未命中 promise
- 结果：输出 `decision:block` JSON。
- `iteration` 从 `1` 自动更新到 `2`。
- 返回 `reason` 为原始 prompt 文本（带前导换行，见风险项）。

3. transcript 含 `<promise>DONE</promise>` 且与配置匹配
- 结果：输出 `✅ Ralph loop: Detected ...`，`exit 0`。
- 状态文件被删除，循环结束。

## 关键代码路径与文件引用

核心文件：
- `plugins/ralph-wiggum/hooks/stop-hook.sh:1-177`

直接调用与配置：
- `plugins/ralph-wiggum/hooks/hooks.json:4-11`
- `plugins/ralph-wiggum/commands/ralph-loop.md:2-5,12-14`
- `plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:130-150`

同状态文件相关命令：
- `plugins/ralph-wiggum/commands/cancel-ralph.md:3,11-18`

文档与行为描述：
- `plugins/ralph-wiggum/README.md:13-27,59-71,121-135`
- `plugins/README.md:26`

测试/规范上下文：
- `plugins/plugin-dev/skills/hook-development/SKILL.md:181-209,300-319`
- `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh:59-69`
- `plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh:41-75`

## 依赖与外部交互

1. 运行时依赖
- `bash`（脚本主执行环境，`set -euo pipefail`）。
- `jq`（读取 `transcript_path`、解析 assistant message、输出决策 JSON）。
- `perl`（多行 promise 标签提取）。
- `sed/awk/grep/tail/mv/rm`（文本和文件处理）。

2. 文件系统交互
- 读：`.claude/ralph-loop.local.md`、`$TRANSCRIPT_PATH`。
- 写：临时状态文件 `${RALPH_STATE_FILE}.tmp.$$`。
- 删：状态文件（达成条件或异常分支）。

3. 与 Claude Runtime 的协议交互
- 入站：stdin JSON（Stop 事件上下文）。
- 出站：stdout 决策 JSON（仅在阻断续跑分支）。

4. 网络交互
- 无任何网络请求。

## 风险、边界与改进建议

1. 风险：`set -e` 与 `jq` 错误检查写法冲突
- 代码在 `LAST_OUTPUT=$(... | jq ... 2>&1)` 后再检查 `$?`（`stop-hook.sh:90-99`）。
- 但在 `set -e` 下，若该命令替换失败，脚本会提前退出，后面的错误提示分支可能永远执行不到。
- 建议：改为 `if ! LAST_OUTPUT=$(...); then ... fi`，保证错误信息可控输出。

2. 风险：assistant 记录筛选过于脆弱
- 依赖 `grep '"role":"assistant"'` 纯字符串匹配 JSONL（`stop-hook.sh:71,81`）。
- 若 transcript JSON 格式有空格变化（如 `"role": "assistant"`）可能漏检。
- 建议：使用 `jq -c 'select(.message.role==\"assistant\")'` 过滤，避免文本格式耦合。

3. 风险：promise 提取存在“误提取”可能
- `perl` 正则在无标签时会返回原文本（随后因不等于 promise 而不误停，功能上可用）。
- 但该行为隐式且可读性低，调试时不直观。
- 建议：改为明确匹配逻辑（如 `if grep -q '<promise>'` 后再提取），并为“无标签”分支设置空字符串。

4. 风险：回灌 prompt 带前导空行
- 实测 block JSON 的 `reason` 以换行开头，来源是状态文件 frontmatter 后的空行 + `awk` 抽取策略。
- 影响：通常不致命，但会让下一轮 prompt 前有多余空白。
- 建议：在输出前对 `PROMPT_TEXT` 做首尾空白清理。

5. 边界：无状态即完全放行
- 如果状态文件意外丢失，Ralph 会立刻失效并允许退出。
- 这与“容错优先”一致，但意味着循环可靠性依赖本地文件完好性。

6. 边界：无 completion promise 时默认无限循环
- 当前仅靠 `max_iterations` 或 promise 收敛，若两者配置不当会长期运行。
- 建议：在 setup 阶段继续强化默认安全提示（脚本已给 warning），并考虑设置软上限告警（例如超过 N 轮仅提示不强制停止）。
