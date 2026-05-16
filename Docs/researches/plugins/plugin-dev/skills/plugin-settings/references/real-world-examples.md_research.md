# real-world-examples.md 研究

源文件：`plugins/plugin-dev/skills/plugin-settings/references/real-world-examples.md`
目标文档：`Docs/researches/plugins/plugin-dev/skills/plugin-settings/references/real-world-examples.md_research.md`

## 场景与职责

`real-world-examples.md` 在 `plugin-settings` 技能中的定位是“案例层 reference”：把 `.claude/*.local.md` 模式从抽象解析技巧映射到完整业务场景。

它承担的职责：

1. 用案例展示 settings 生命周期：创建 -> 读取 -> 更新 -> 清理。
2. 将通用模式（quick-exit、enabled、原子更新）放入具体插件语境。
3. 作为 `plugin-settings/SKILL.md` 的下钻材料，被技能触发后按需读取（`plugins/plugin-dev/skills/plugin-settings/SKILL.md:514-516`）。
4. 作为 plugin-dev 对外宣称的“real-world examples”证据（`plugins/plugin-dev/README.md:123-131`）。

## 功能点目的

### 1) 展示 settings 文件不是“静态配置”，而是“运行中状态机”

文档把 `.local.md` 分别用于：

1. 多 agent 协调态（`multi-agent-swarm`）。
2. 循环控制态（`ralph-wiggum`）。

这使读者理解 settings 的用途不仅是开关配置，也可承载状态推进。

### 2) 提供端到端闭环示例

文档分别给出：

1. 创建：`cat <<EOF` 写入 `.local.md`（`real-world-examples.md:100-116`, `233-252`）。
2. 消费：hook 读取 frontmatter 与 body（`real-world-examples.md:50-86`, `156-219`）。
3. 更新：`sed` + `mv` 更新字段（`real-world-examples.md:122-127`, `203-207`）。

### 3) 抽取跨场景通用设计原则

后半段总结 quick-exit、enabled、原子写入、默认值、防反模式（`real-world-examples.md:266-385`），用于指导其他插件复用。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 数据结构

两类示例都使用统一结构：

1. frontmatter：机器可解析字段（任务编号、迭代计数、开关、promise 等）。
2. body：人类可读任务文本/提示词。

文档位置：`real-world-examples.md:11-40`, `135-146`。

### 流程 A：multi-agent-swarm（文档案例）

文档中的控制流：

1. Stop hook 检查 `.claude/multi-agent-swarm.local.md` 是否存在，不存在即退出。
2. 解析 `coordinator_session/agent_name/task_number/pr_number/enabled`。
3. 若启用则通过 `tmux send-keys` 通知协调会话。

对应片段：`real-world-examples.md:50-86`。

创建与更新流程：`real-world-examples.md:100-127`。

### 流程 B：ralph-wiggum（文档案例）

文档中的流程摘要：

1. 读取 `iteration/max_iterations/completion_promise`。
2. 检查迭代上限。
3. 从 transcript 抽取最后一次 assistant 输出。
4. 检查 `<promise>...</promise>` 是否满足。
5. 若未完成则 `iteration + 1`，并把 body 作为下一轮 prompt 输出到 Hook 决策 JSON。

对应片段：`real-world-examples.md:156-219`。

### 与仓库真实实现的映射

#### ralph-wiggum 是可核验的真实实现

真实调用链：

1. 命令触发 setup：`plugins/ralph-wiggum/commands/ralph-loop.md:1-14`
2. setup 写入状态：`plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:140-150`
3. Stop Hook 消费状态并循环：`plugins/ralph-wiggum/hooks/stop-hook.sh:13-177`
4. Hook 注册：`plugins/ralph-wiggum/hooks/hooks.json:3-10`

实现特征：

1. 比 reference 示例更严格：含数值合法性校验、transcript 存在校验、JSON 解析错误处理。
2. 真实 state 还带 `active: true` 字段（`setup-ralph-loop.sh:142`），reference 示例未体现。

#### multi-agent-swarm 在本仓库不可核验

本次检索 `plugins/` 下不存在 `plugins/multi-agent-swarm/` 目录；相关命中主要出现在文档文本中。因此该部分在当前仓库语境更像“外部案例/历史案例”，而非本仓库可直接追踪实现。

### 协议与命令层面

1. frontmatter 解析协议：
   - `sed -n '/^---$/,/^---$/{ /^---$/d; p; }'`
2. body 解析协议：
   - `awk '/^---$/{i++; next} i>=2'`
3. Stop hook 决策协议：
   - `jq -n --arg ... '{"decision":"block", ...}'`
4. 状态更新协议：
   - `sed ... > temp && mv temp file`

上述协议在 `real-world-examples.md` 与 `ralph-wiggum` 实现中都能找到对应。

## 关键代码路径与文件引用

### 直接研究对象

1. `plugins/plugin-dev/skills/plugin-settings/references/real-world-examples.md`

### 调用方（文档消费者）

1. `plugins/plugin-dev/skills/plugin-settings/SKILL.md:514-516`
2. `plugins/plugin-dev/README.md:123-131`
3. `plugins/plugin-dev/commands/create-plugin.md:157-163,220-226`
4. `plugins/plugin-dev/skills/skill-development/SKILL.md:311-314`

### 被调用方/映射实现

1. `plugins/ralph-wiggum/commands/ralph-loop.md`
2. `plugins/ralph-wiggum/scripts/setup-ralph-loop.sh`
3. `plugins/ralph-wiggum/hooks/stop-hook.sh`
4. `plugins/ralph-wiggum/hooks/hooks.json`
5. `plugins/ralph-wiggum/commands/cancel-ralph.md`
6. `plugins/plugin-dev/skills/plugin-settings/scripts/parse-frontmatter.sh`
7. `plugins/plugin-dev/skills/plugin-settings/scripts/validate-settings.sh`

### 配置、测试、脚本、文档链路

1. 配置：`.claude/multi-agent-swarm.local.md`（文档案例）、`.claude/ralph-loop.local.md`（真实实现）
2. 脚本：`setup-ralph-loop.sh` 负责创建状态，`stop-hook.sh` 负责消费与推进
3. 测试：当前无针对 `real-world-examples.md` 的自动化校验脚本，主要靠人工同步维护
4. 文档链路：`plugin-dev/README.md` -> `plugin-settings/SKILL.md` -> `references/real-world-examples.md`

## 依赖与外部交互

### 运行依赖

1. 通用 shell 工具：`bash`, `sed`, `grep`, `awk`
2. JSON 处理：`jq`
3. ralph Promise 提取：`perl`（真实实现）
4. swarm 示例交互：`tmux`（文档案例）

### Claude Code 运行时交互

1. Stop Hook 读取 Hook 输入中的 transcript 路径（真实实现：`stop-hook.sh:57-60`）。
2. Stop Hook 读取 transcript JSONL 并提取 assistant 文本（`stop-hook.sh:69-95`）。
3. 满足条件时删除 state 文件并放行退出；否则输出 block 决策继续循环（`stop-hook.sh:123-177`）。

### 文件系统交互

1. setup 脚本创建 `.claude/ralph-loop.local.md`（`setup-ralph-loop.sh:131-150`）。
2. hook 原子更新 `iteration`（`stop-hook.sh:154-156`）。
3. `/cancel-ralph` 删除状态文件作为人工退出通道（`cancel-ralph.md:15-18`）。

## 风险、边界与改进建议

### 风险 1（高）：`real-world` 命名与仓库可追溯性不一致

问题：文档主案例之一 `multi-agent-swarm` 在当前仓库没有对应插件目录。

影响：读者容易误判为“仓库内可直接复用代码”，但实际无法本地追踪。

建议：

1. 在该章节显式标记为 `external example`。
2. 补充来源仓库或 commit 链接。
3. 若无稳定来源，替换为当前仓库内可核验插件案例。

### 风险 2（高）：reference 与真实 ralph 实现存在细节漂移

例子：

1. reference 说“两个插件都使用 enabled 字段”，但真实 ralph state 并未使用 `enabled` 作为主开关。
2. reference 的 ralph 片段较简化，未体现真实实现中的 transcript/JSON 异常处理分支。

建议：

1. 在文档中区分“教学版片段”与“生产版实现”。
2. 增加 `Reference vs Actual` 对照表，列出缺省逻辑与增强逻辑。

### 风险 3（中）：示例命令存在输入转义边界

问题：`cat <<EOF` 直接插值变量（如 `additional_instructions`、`completion_promise`）时，若含引号、换行、特殊字符，可能导致 YAML 结构意外破坏。

建议：

1. 统一使用 YAML 安全转义策略（例如先做字符串清洗）。
2. 对用户输入字段做字符约束或编码。

### 风险 4（中）：文档层缺少一致性检测机制

问题：当前没有自动化检查“reference 文档中的路径/文件名是否仍存在”。

建议：

1. 添加 `docs link check` 或路径存在性脚本。
2. 在 CI 中做最小校验：引用路径必须可解析、关键命令片段与实现签名同步。

### 风险 5（中）：教程文本与实际命令描述有局部不一致

例如 `plugins/ralph-wiggum/commands/help.md` 中有 `.claude/.ralph-loop.local.md` 的路径描述，而真实文件名是 `.claude/ralph-loop.local.md`（无前导点）。这会放大读者理解偏差。

建议：

1. 统一所有文档/命令中的状态文件路径写法。
2. 增加一次“命名一致性清扫”并纳入发布前 checklist。

### 风险 6（低）：最佳实践总结与具体实现未绑定版本

问题：文档总结长期有效，但实现持续演化时易出现“原则正确、代码细节过期”。

建议：

1. 给案例标注“最后核验日期 + 对应版本”。
2. 在 `plugin-dev` 版本升级时同步刷新 reference 案例片段。
