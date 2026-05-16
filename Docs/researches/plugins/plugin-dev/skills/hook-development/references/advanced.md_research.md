# plugins/plugin-dev/skills/hook-development/references/advanced.md 研究

## 场景与职责

`advanced.md` 是 `hook-development` 技能体系中的“进阶策略层”文档，定位不是基础 API 教程，而是给已经能写普通 hook 的开发者提供复杂编排模板。

它在插件开发流程中的职责来源明确：
- 在技能主文档中被列为附加资源，作为进阶补充（`plugins/plugin-dev/skills/hook-development/SKILL.md:665-673`）。
- 在 `plugin-dev` 总体工作流中，hooks 被放在“Add Automation + Test & Validate”阶段，`advanced.md` 提供这些阶段会遇到的复杂场景模板（`plugins/plugin-dev/README.md:249-257`）。
- 在 `/plugin-dev:create-plugin` 的 Hooks 实施阶段，要求优先 prompt hook 并结合工具脚本测试，该文档补的是“多阶段/跨事件/外部系统”这些基础教程之外的实践（`plugins/plugin-dev/commands/create-plugin.md:201-209`）。

职责边界：
- 该文件本身不被运行时直接执行，不参与 CI。
- 它通过“可复制配置片段 + shell 模板”影响后续 `hooks/hooks.json` 与脚本实现质量。
- 其内容应与 `SKILL.md` 协议保持一致（输入字段、退出码、并行语义），否则会造成误导。

## 功能点目的

`advanced.md` 的功能点是围绕“复杂度上升后如何保持可维护”展开，主要目的如下：

1. 提供多层校验设计
- 通过 command hook + prompt hook 组合实现“快筛 + 深判”的结构化思路（`advanced.md:5-49`）。

2. 提供上下文条件化策略
- 用环境变量、用户角色、项目配置文件驱动差异化行为，避免全局一刀切（`advanced.md:50-144`）。

3. 提供跨事件状态编排思路
- 展示 SessionStart/PostToolUse/Stop 之间如何通过临时状态协同（`advanced.md:229-265`）。

4. 给出性能优化基线
- 通过缓存和并行独立化设计降低延迟（`advanced.md:168-227`）。

5. 覆盖组织级集成场景
- 给出 Slack/数据库/指标系统接入模板，扩展到审计与可观测性（`advanced.md:267-311`）。

6. 补齐安全与测试面
- 增加限流、审计、秘密检测模式及单测/集成测样例（`advanced.md:313-427`）。

7. 明确反模式
- 直接标注“依赖执行顺序、超长执行、未处理异常”三类常见失败路径（`advanced.md:440-475`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 结构模式：以事件为顶层、以 hooks 数组表达执行单元

文档中的 JSON 片段统一使用：
- 事件键（如 `PreToolUse`、`Stop`）
- 事件项（`matcher` + `hooks`）
- 子 hook（`type=command|prompt` + `command|prompt` + `timeout`）

这与技能文档定义的核心结构一致（`plugins/plugin-dev/skills/hook-development/SKILL.md:121-209`, `340-383`）。

### 2) 多阶段校验流程（逻辑层）

“多阶段校验”示例想表达：
1. command hook 先做低成本规则判断
2. prompt hook 再做语义分析

关键实现片段：
- `quick-check.sh` 用 `jq` 提取 `.tool_input.command`，对安全命令快速放行（`advanced.md:35-45`）。
- prompt hook 用 `$TOOL_INPUT` 做更高语义检查（`advanced.md:21-23`）。

但这里有一个协议层细节：技能文档明确同一事件下匹配的 hooks 并行执行（`plugins/plugin-dev/skills/hook-development/SKILL.md:495-517`）。因此该示例是“逻辑上的分层”，并非运行时短路链。也就是说 quick-check 不会阻止 prompt hook 被调度。

### 3) 条件执行与配置驱动

`advanced.md` 提供两种条件分流实现：

1. 环境分流
- 基于 `CI`、`USER` 等环境变量提前 `exit 0` 跳过（`advanced.md:54-83`）。

2. 项目配置分流
- 在 `CLAUDE_PROJECT_DIR` 下读取 `.claude-hooks-config.json`，以 `strict_mode` 控制策略（`advanced.md:119-144`）。

这类实现与技能中的环境变量约定一致（`plugins/plugin-dev/skills/hook-development/SKILL.md:322-339`）。

### 4) 跨事件状态协作

文档展示了 `SessionStart -> PostToolUse -> Stop` 的计数闭环：
- SessionStart 初始化计数文件（`advanced.md:233-239`）
- PostToolUse 根据 `tool_result` 增量计数（`advanced.md:241-254`）
- Stop 阶段读取计数决定 block/allow（`advanced.md:256-265`）

实现依赖是临时文件 + shell 退出码协议（`exit 2` 阻断）。这与技能文档的退出码语义一致（`plugins/plugin-dev/skills/hook-development/SKILL.md:294-299`）。

### 5) 性能与可观测实现

1. 缓存
- 以 `file_path -> md5 -> /tmp/hook-cache-*` 建键并带 300 秒 TTL（`advanced.md:172-194`）。
- 用 `stat -f%m || stat -c%Y` 兼容 BSD/GNU 的修改时间读取（`advanced.md:181`）。

2. 并行友好
- 示例强调多个子 hook 必须“互相独立”，避免共享中间状态（`advanced.md:196-227`）。

3. 外部系统接入
- `curl` 推送 Slack webhook（`advanced.md:271-285`）。
- `psql` 写审计表（`advanced.md:289-297`）。
- `nc` 发 StatsD 指标（`advanced.md:302-310`）。

### 6) 安全模式与测试模式

1. 安全模式
- 限流：按分钟窗口计数并超过阈值阻断（`advanced.md:317-347`）。
- 审计日志：附带用户、工具、输入落盘（`advanced.md:351-360`）。
- 秘钥检测：对内容应用 regex 规则阻断（`advanced.md:365-377`）。

2. 测试模式
- 单测样例：通过 stdin 构造输入并断言 exit code（`advanced.md:381-402`）。
- 集成样例：显式设置 `CLAUDE_PROJECT_DIR`/`CLAUDE_PLUGIN_ROOT` 后跑会话钩子（`advanced.md:408-427`）。

结合工具脚本，可形成更稳定的测试闭环：
- 自动造样本：`test-hook.sh --create-sample <event>`（`plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh:25-100`）。
- 输出与退出码解析：`test-hook.sh` 主流程（`plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh:183-252`）。

## 关键代码路径与文件引用

- 目标文档：`plugins/plugin-dev/skills/hook-development/references/advanced.md:1-479`
- 上游入口：`plugins/plugin-dev/skills/hook-development/SKILL.md:665-673`
- 协议基础：
  - 配置格式与事件：`plugins/plugin-dev/skills/hook-development/SKILL.md:60-209`
  - 输入/输出/退出码：`plugins/plugin-dev/skills/hook-development/SKILL.md:278-320`
  - 并行语义：`plugins/plugin-dev/skills/hook-development/SKILL.md:493-517`
- 实现样例（对应 advanced 片段）：
  - `plugins/plugin-dev/skills/hook-development/examples/validate-bash.sh:1-43`
  - `plugins/plugin-dev/skills/hook-development/examples/validate-write.sh:1-38`
  - `plugins/plugin-dev/skills/hook-development/examples/load-context.sh:1-55`
- 测试/校验工具：
  - `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh:1-252`
  - `plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh:1-159`
  - `plugins/plugin-dev/skills/hook-development/scripts/hook-linter.sh:1-153`
- 调用链：
  - `plugins/plugin-dev/README.md:56-75,249-257,271-284`
  - `plugins/plugin-dev/commands/create-plugin.md:201-209,257-260`
  - `plugins/plugin-dev/agents/plugin-validator.md:107-115`

## 依赖与外部交互

1. 运行时依赖（模板级）
- Shell: `bash`
- JSON 解析: `jq`
- 文件与时间工具: `stat`, `date`, `md5sum`
- 超时与测试: `timeout`（由测试脚本使用）

2. Claude Code 协议依赖
- stdin JSON 字段：`tool_name`, `tool_input`, `tool_result`, `reason` 等（`SKILL.md:300-320`）。
- 环境变量：`CLAUDE_PROJECT_DIR`, `CLAUDE_PLUGIN_ROOT`, `CLAUDE_ENV_FILE`（`SKILL.md:322-339`）。
- 决策出口：`exit 0/2` 与结构化 JSON。

3. 外部系统交互（可选能力）
- 网络：Slack webhook (`curl`)
- 数据库：PostgreSQL (`psql`)
- 指标：StatsD (`nc`)

4. 本次仓内实测（验证依赖一致性）
- 执行 `validate-hook-schema.sh` 校验真实插件 hooks（如 `plugins/security-guidance/hooks/hooks.json`）会因 wrapper 结构触发 `jq` 索引错误并退出 5，而非文档承诺的完整汇总（脚本行为见 `validate-hook-schema.sh:43,65-67`，实测输出为 `Cannot index string with number`）。

## 风险、边界与改进建议

1. 风险：多阶段示例容易被理解为顺序执行
- `advanced.md` 的“先 quick-check 再 prompt”描述容易被读成短路链，但同事件 hooks 是并行调度（`SKILL.md:495-517`）。
- 建议：在多阶段段落追加“此组合是并行执行，不会互相短路”的显式说明；若要真分层，改为跨事件或拆成单 hook 内部流程。

2. 风险：状态文件命名冲突与失配
- 示例大量使用 `/tmp/*-$$`；不同进程下 `$$` 不稳定，跨事件协同时可能读不到同一状态。
- 建议：优先使用 `session_id` 派生命名键；统一清理生命周期，避免残留状态污染。

3. 风险：示例片段与严格 JSON 不完全一致
- 并行示例 JSON 内含 `//` 注释（`advanced.md:208,213,218`），直接复制会导致 JSON 解析失败。
- 建议：把注释移出代码块或改成旁注文本。

4. 风险：外部写入示例存在注入/泄漏面
- `psql` 片段直接拼接 `$input`（`advanced.md:294`），可能引发 SQL 注入与日志爆炸。
- 审计日志片段写入完整输入（`advanced.md:358`），可能泄漏敏感信息。
- 建议：参数化 SQL、字段脱敏、日志截断与采样。

5. 风险：测试样例与工具链割裂
- 文档自写 `test-hook.sh` 与仓内正式 `scripts/test-hook.sh` 同名但语义不同，可能造成认知冲突。
- 建议：优先引用仓内工具脚本，并提供“如何把 advanced 片段接入 test-hook.sh”步骤。

6. 边界
- 本文档提供的是“策略模板”，不保证即拷即用；必须结合插件实际 hooks 格式（wrapper/direct）与事件支持面进行适配。
- 对外部系统依赖（Slack/DB/StatsD）仅给连接样例，不含鉴权、重试、幂等、错误预算等生产要求。

7. 综合改进建议
- 在 `advanced.md` 末尾增加“可直接通过 validate-hook-schema/test-hook/hook-linter 验证”的落地检查单。
- 在每个复杂模式下补一行“适用事件/不适用事件”。
- 补充一节“与 plugin wrapper 格式对齐示例”，降低新用户复制失败率。
