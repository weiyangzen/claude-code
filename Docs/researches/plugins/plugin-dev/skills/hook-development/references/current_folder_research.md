# DIR 研究：plugins/plugin-dev/skills/hook-development/references

## 场景与职责

`plugins/plugin-dev/skills/hook-development/references` 是 `hook-development` 技能的“深水区知识层”，承担 progressive disclosure 中“按需加载细节”的职责。

在该技能体系中的分工关系：

- `SKILL.md` 提供触发条件、核心协议、快速上手流程。
- `references/` 提供可复用模式、迁移策略、复杂场景扩展。
- `examples/` 提供可运行脚本样例。
- `scripts/` 提供校验/测试/lint 工具。

因此，本目录并不直接被运行时执行，但直接影响开发者如何设计 Hook 策略、如何在 prompt/command 之间做架构取舍，以及如何把 Hook 从“可用”演进到“可维护”。

上游与下游关系：

- 上游调用方（知识入口）：
  - `plugins/plugin-dev/skills/hook-development/SKILL.md:665-674` 将本目录三份文档列为额外资源。
  - `plugins/plugin-dev/skills/skill-development/SKILL.md:190-194` 把 `references/` 作为“SKILL.md 减负”的标准实践。
- 下游消费者（落地执行）：
  - `plugins/plugin-dev/commands/create-plugin.md:201-209` 要求创建 hooks 并结合验证工具测试。
  - `plugins/plugin-dev/agents/plugin-validator.md:107-115` 要求对 hooks 配置进行结构校验。
  - `plugins/plugin-dev/README.md:56-75,249-257,273-284` 把 hook-development 作为 plugin 开发流程中的自动化与验证核心能力。

结论：`references/` 的职责不是“再写一遍 SKILL”，而是提供在真实工程中反复复用的策略模板与迁移路径。

## 功能点目的

本目录包含 3 个文件，定位清晰且互补。

### 1) `patterns.md`：标准模式库（从常见到配置化）

- 提供至少 10 类模式：安全写入校验、测试门禁、SessionStart 上下文加载、通知日志、MCP 删除保护、构建校验、危险命令确认、PostToolUse 质量检查、临时启用 Hook、配置驱动 Hook（`patterns.md:5-346`）。
- 目标是“拎包即用”：每个模式都给出 JSON 配置或脚本骨架，降低首次实现成本。
- 与 `SKILL.md` 的关系：`SKILL.md` 讲原则，`patterns.md` 给模板。

### 2) `migration.md`：从命令式到提示式的迁移方法论

- 核心目标：把硬编码 shell 规则迁移成 prompt hook 的自然语言判定标准（`migration.md:14-154`）。
- 给出“何时不迁移”的边界：确定性数学检查、外部工具强耦合、极低延迟场景（`migration.md:155-204`）。
- 提供混合架构建议（先 command 快筛，再 prompt 深判）（`migration.md:205-232`）。
- 提供迁移 checklist，降低大规模改造风险（`migration.md:233-244`）。

### 3) `advanced.md`：复杂编排与组织级集成

- 面向高级场景：多阶段校验、条件执行、跨事件状态协作、缓存优化、外部系统对接（Slack/DB/Metrics）、限流与审计（`advanced.md:5-360`）。
- 强调并行执行模型下“独立性优先”，避免顺序依赖（`advanced.md:196-227,440-448`）。
- 补充测试策略（单元/集成）与复杂 Hook 常见陷阱（`advanced.md:379-477`）。

目录级目标可概括为：

1. 让新手有模式可抄。
2. 让存量脚本有迁移路径。
3. 让进阶场景有工程化边界与性能/可靠性约束。

## 具体技术实现（关键流程/数据结构/协议/命令）

尽管本目录是文档，但它定义了可执行实现的关键技术约定。

### A. 配置结构与事件模型

三份文档统一围绕 Hook 事件数组结构展开：

- 事件键：`PreToolUse/PostToolUse/Stop/SubagentStop/SessionStart/...`
- 事件项：`matcher + hooks[]`
- 子 Hook：
  - `type: command` + `command`(+`timeout`)
  - `type: prompt` + `prompt`(+`timeout`)

示例覆盖：

- `patterns.md:9-23,31-45,54-67,90-104,112-126,134-148,157-170,179-192,212-257`
- `migration.md:19-33,58-73,89-102,133-146,209-229,282-316`
- `advanced.md:9-29,150-164,200-225`

### B. 迁移流程（migration 核心）

`migration.md` 的实际工程流程：

1. 识别旧 command hook 中的硬编码逻辑。
2. 把 regex/字符串匹配规则改写成“判定标准语句”（intent-based criteria）。
3. 为 prompt hook 设置超时和适用事件。
4. 用 edge cases 回归，确保新策略覆盖旧策略并补齐盲点。
5. 对仍需确定性的逻辑保留 command hook，必要时采用 hybrid。

关键片段：`migration.md:233-252,320-365`。

### C. 高级编排流程（advanced 核心）

`advanced.md` 给出三类关键编排：

1. 多阶段校验：`command quick-check + prompt deep-check`（`advanced.md:5-49`）。
2. 跨事件工作流：`SessionStart 初始化 -> PostToolUse 计数 -> Stop 审核`（`advanced.md:229-265`）。
3. 性能优化：缓存 5 分钟结果、并行独立执行（`advanced.md:168-227`）。

这三类流程共同强调：

- 不在同事件并行 Hook 之间做顺序假设。
- 把“快 + 准”拆成不同层次，避免单一 Hook 过载。

### D. 与工具脚本的协议耦合

references 中的实现片段，实际上默认遵守 `hook-development/scripts` 工具链和 SKILL 协议：

- 输入来自 stdin JSON（`SKILL.md:300-320`）。
- 退出码 `0/2` 语义（`SKILL.md:294-299`）。
- `test-hook.sh` 可生成样例并解释结果（`scripts/test-hook.sh:25-100,205-252`）。
- `validate-hook-schema.sh` 会校验事件与字段（`scripts/validate-hook-schema.sh:41-159`）。

实测（本次）：

```bash
bash plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh \
  plugins/security-guidance/hooks/hooks.json
```

- 结果：对 `description/hooks` 顶层键发出 unknown event，并在后续 `jq` 访问时异常退出（exit 5）。
- 说明：references 中给出的事件结构与“插件 wrapper 格式”在当前校验脚本实现下仍有兼容缝隙，需要额外处理。

### E. 文档中出现的关键命令族

references 侧重以下命令与协议：

- `jq`：stdin JSON 解析与字段提取。
- `stat/date/md5sum`：缓存与时效判断。
- `curl/psql/nc`：外部系统通知、审计、指标。
- shell 控制语义：`exit 0` 放行、`exit 2` 阻断。

这些命令在 `advanced.md` 和 `patterns.md` 中作为模板存在，属于“推荐实现片段”，不是仓库中默认开启的生产流水线。

## 关键代码路径与文件引用

### 目标目录

- `plugins/plugin-dev/skills/hook-development/references/patterns.md`
- `plugins/plugin-dev/skills/hook-development/references/migration.md`
- `plugins/plugin-dev/skills/hook-development/references/advanced.md`

### 直接上下文

- `plugins/plugin-dev/skills/hook-development/SKILL.md:665-689`
- `plugins/plugin-dev/skills/hook-development/examples/validate-write.sh:1-38`
- `plugins/plugin-dev/skills/hook-development/examples/validate-bash.sh:1-43`
- `plugins/plugin-dev/skills/hook-development/examples/load-context.sh:1-55`
- `plugins/plugin-dev/skills/hook-development/scripts/README.md:5-123`
- `plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh:41-159`
- `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh:25-252`
- `plugins/plugin-dev/skills/hook-development/scripts/hook-linter.sh:23-153`

### 调用链与治理链

- `plugins/plugin-dev/README.md:56-75,249-257,273-284`
- `plugins/plugin-dev/commands/create-plugin.md:201-209,257-260`
- `plugins/plugin-dev/agents/plugin-validator.md:107-115`
- `plugins/plugin-dev/skills/skill-development/SKILL.md:130-135,190-194,298-304,331-340`

### 真实插件配置参照（被动消费者）

- `plugins/security-guidance/hooks/hooks.json:1-16`
- `plugins/learning-output-style/hooks/hooks.json:1-15`
- `plugins/explanatory-output-style/hooks/hooks.json:1-15`

## 依赖与外部交互

### 本地依赖

- shell：`bash`
- JSON 处理：`jq`
- 超时控制：`timeout`（由 `test-hook.sh` 使用）
- 常规 Unix 工具：`grep/stat/date/md5sum`

### Claude Code 协议依赖

references 的脚本模板默认依赖这些环境变量与字段：

- `CLAUDE_PROJECT_DIR`
- `CLAUDE_PLUGIN_ROOT`
- `CLAUDE_ENV_FILE`
- stdin 中的 `hook_event_name/tool_name/tool_input/...`

### 外部系统交互（可选）

`advanced.md` 提供了组织级扩展模式：

- Slack webhook 通知（`curl`）
- 数据库审计写入（`psql`）
- StatsD 指标上报（`nc`）

这些均为“示例能力”，并未在本仓库默认流程中强制启用。

### 测试与验证现状

- references 本身无独立自动化测试。
- 实践上依赖 `scripts/` 三件套做离线校验。
- 本次实测确认：
  - `test-hook.sh --create-sample PreToolUse` 能正确生成样例输入。
  - `test-hook.sh` 测 `validate-bash.sh`（危险命令）可得到 exit 2 + JSON 阻断输出。
  - `test-hook.sh` 直接测 `load-context.sh` 在默认 `/tmp/test-project` 不存在时失败，需显式设置存在的 `CLAUDE_PROJECT_DIR`。

## 风险、边界与改进建议

### 1) 文档示例与校验器实现存在结构错位

现状：

- 技能文档明确插件常用 wrapper：`{"description":...,"hooks":{...}}`（`SKILL.md:62-80`）。
- 当前 `validate-hook-schema.sh` 默认按“事件在顶层”遍历（`validate-hook-schema.sh:43,65`）。

影响：

- 对真实插件 hooks 文件易误报或异常退出，削弱 references 的可信度与可执行性闭环。

建议：

1. 在校验脚本中自动兼容 wrapper/direct 两种根结构。
2. 在 `scripts/README.md` 明确脚本当前支持边界，避免误用。

### 2) 部分高级模式对执行模型假设较强

现状：

- `advanced.md` 给出“临时文件共享状态”方案（`advanced.md:85-114,229-265`）。
- 若用户误用于同事件并行 Hook，存在竞态与不确定性。

建议：

1. 在相应段落增加“仅跨事件串行场景适用”的显式警告。
2. 推荐使用 `session_id` 命名隔离状态文件，降低并发冲突。

### 3) 示例命令跨平台细节未统一

现状：

- references 与脚本中同时出现 GNU/BSD 差异处理（如 `stat -f`/`stat -c`），但并非所有片段都做了兼容。

建议：

1. 在 `references/advanced.md` 的所有文件时间/大小相关片段统一补充跨平台写法。
2. 在文档开头增加“依赖命令可用性检查”建议。

### 4) `test-hook.sh` 默认环境对 SessionStart 示例不够友好

现状：

- 脚本默认 `CLAUDE_PROJECT_DIR=/tmp/test-project`（`test-hook.sh:171`），目录未自动创建。
- 这会让 `examples/load-context.sh` 首次测试直接失败。

建议：

1. 在 `test-hook.sh` 中默认 `mkdir -p "$CLAUDE_PROJECT_DIR"`，或
2. 在 `scripts/README.md` 的 SessionStart 测试示例中显式说明需提供现有目录。

### 5) references 与 examples 的边界可再收敛

现状：

- references 中存在较多脚本片段，部分与 examples 职责重叠。

建议：

1. references 保留“模式+准则+关键片段”。
2. 完整可执行脚本优先落入 `examples/`，并在 references 链接，减少双处维护。

---

综合评估：`references/` 目录结构合理、覆盖完整，已经形成“模式库 + 迁移指南 + 高级编排”三位一体体系；当前主要短板是与本地校验工具在配置根结构上的一致性，以及少数高级模式的适用边界提示不够显式。
