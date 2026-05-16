# plugins/plugin-dev/skills/plugin-settings/SKILL.md 研究

## 场景与职责

`plugins/plugin-dev/skills/plugin-settings/SKILL.md` 是 `plugin-dev` 工具包中“插件配置与状态持久化模式”的核心规范文件，职责不是直接执行，而是定义一套可被命令/Hook/Agent复用的约定。

它在体系中的位置可分为四层：

1. 触发路由层：通过 frontmatter `description` 声明触发短语（`plugin settings`、`.local.md`、`YAML frontmatter` 等），让技能在“需要可配置插件行为”场景被加载（`plugins/plugin-dev/skills/plugin-settings/SKILL.md:1-5`）。
2. 规范层：定义 `.claude/plugin-name.local.md` 文件协议（YAML frontmatter + Markdown body）及生命周期（本地文件、建议 gitignore）（`plugins/plugin-dev/skills/plugin-settings/SKILL.md:11-19`）。
3. 实施层：给出 hooks/commands/agents 读取与应用配置的标准流程（存在性快速退出、frontmatter 提取、字段解析、按 `enabled` 开关短路）（`plugins/plugin-dev/skills/plugin-settings/SKILL.md:60-134`）。
4. 支撑层：把示例、参考文档、工具脚本分拆到 `examples/`、`references/`、`scripts/`，体现 progressive disclosure（`plugins/plugin-dev/skills/plugin-settings/SKILL.md:508-544`）。

与上下文依赖关系：

- 主要调用方：
  - `plugins/plugin-dev/README.md` 将其列为第 4 个核心技能，并公开触发词与资源清单（`plugins/plugin-dev/README.md:114-133`）。
  - `/plugin-dev:create-plugin` 在 Phase 5 明确要求“Settings: Load plugin-settings skill”，并要求模板、读取实现、`.gitignore` 配置（`plugins/plugin-dev/commands/create-plugin.md:157-226`）。
- 被调用方（技能内下游）：
  - 示例：`examples/read-settings-hook.sh`、`examples/create-settings-command.md`、`examples/example-settings.md`。
  - 参考：`references/parsing-techniques.md`、`references/real-world-examples.md`。
  - 脚本：`scripts/parse-frontmatter.sh`、`scripts/validate-settings.sh`。

## 功能点目的

1. 明确“配置文件形态”
- 目的：统一每项目配置与状态存储路径，避免分散在任意 JSON/TXT。
- 实现：约定 `.claude/plugin-name.local.md`，frontmatter 放结构化字段，body 放补充上下文/提示词（`plugins/plugin-dev/skills/plugin-settings/SKILL.md:11-58`）。

2. 支持三类消费方
- Hook：用于运行期快速开关/策略分支。
- Command：用于按用户配置调整命令行为。
- Agent：用于在系统指令中适配项目偏好。
- 位置：`plugins/plugin-dev/skills/plugin-settings/SKILL.md:60-134`。

3. 提供可复用解析套路
- 目的：用最小依赖（`sed`/`grep`/`awk`）快速落地。
- 位置：`plugins/plugin-dev/skills/plugin-settings/SKILL.md:136-171`。

4. 规范常见业务模式
- 临时启停 Hook（`enabled`）、Agent 状态管理、配置驱动分支（`validation_level`）
- 位置：`plugins/plugin-dev/skills/plugin-settings/SKILL.md:173-270`。

5. 给出配置创建与文档模板
- 目的：降低插件作者从“有需求”到“可用配置”的路径成本。
- 位置：`plugins/plugin-dev/skills/plugin-settings/SKILL.md:272-311`。

6. 加入工程实践约束
- 命名规则、gitignore、默认值、输入校验、重启提示。
- 位置：`plugins/plugin-dev/skills/plugin-settings/SKILL.md:313-383`。

7. 加入安全约束
- 用户输入转义、路径穿越防护、权限建议（600）。
- 位置：`plugins/plugin-dev/skills/plugin-settings/SKILL.md:385-423`。

8. 提供真实案例映射
- 引入 `multi-agent-swarm`、`ralph-wiggum` 模式，说明状态文件如何驱动 Stop Hook 行为。
- 位置：`plugins/plugin-dev/skills/plugin-settings/SKILL.md:424-472`。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 数据结构/协议

核心数据结构是 markdown 文件内的“双区块协议”:

```markdown
---
enabled: true
mode: standard
max_retries: 3
---

# Body
可选任务说明、提示词、备注
```

- frontmatter：键值配置（布尔/数值/字符串/简易列表）。
- body：自由文本，常用于 stop hook 的回灌 prompt。
- 主定义：`plugins/plugin-dev/skills/plugin-settings/SKILL.md:22-58`。

### 2) Hook 读取关键流程（Bash）

标准流程：

1. `[[ ! -f "$SETTINGS_FILE" ]] && exit 0`（未配置时快速退出）。
2. `sed -n '/^---$/,/^---$/{ /^---$/d; p; }'` 提取 frontmatter。
3. `grep '^field:' | sed ...` 提取字段。
4. `enabled != true` 再次快速退出。
5. 按配置执行实际校验逻辑。

示例实现：`plugins/plugin-dev/skills/plugin-settings/examples/read-settings-hook.sh:10-65`。

### 3) Hook 输入输出协议差异（PreToolUse vs Stop）

1. PreToolUse 风格（`read-settings-hook.sh`）
- 输入：stdin JSON（读取 `tool_input.file_path`/`tool_input.content`）。
- 阻断输出：`{"hookSpecificOutput":{"permissionDecision":"deny"},"systemMessage":"..."}`。
- 退出码：`exit 2` 阻断，`exit 0` 放行。
- 参考：`plugins/plugin-dev/skills/plugin-settings/examples/read-settings-hook.sh:29-65`。

2. Stop 风格（真实插件 `ralph-wiggum`）
- 输出：`{"decision":"block","reason":"...","systemMessage":"..."}`。
- 该协议在 `plugins/ralph-wiggum/hooks/stop-hook.sh:167-174` 可见。

这说明同样是 `.local.md`，不同 Hook 事件的协议字段不同，不能混用。

### 4) Command 侧创建流程

`create-settings-command.md` 给出的流程：

1. AskUserQuestion 收集配置偏好（启用、模式）。
2. 解析答案映射到 YAML 字段。
3. Write 生成 `.claude/my-plugin.local.md`。
4. 回告“已创建 + 修改方法 + 重启提示 + gitignore”。

参考：`plugins/plugin-dev/skills/plugin-settings/examples/create-settings-command.md:12-98`。

### 5) 工具脚本实现

1. `parse-frontmatter.sh`
- CLI：`bash .../parse-frontmatter.sh <settings-file.md> [field]`。
- 功能：输出 frontmatter 全量或单字段。
- 关键点：`set -euo pipefail` + `grep/sed` 提取。
- 位置：`plugins/plugin-dev/skills/plugin-settings/scripts/parse-frontmatter.sh:7-58`。

2. `validate-settings.sh`
- CLI：`bash .../validate-settings.sh <path/to/settings.local.md>`。
- 功能：结构性校验（文件、marker、frontmatter、字段概览、body）。
- 行为：多数值异常为 warning，不阻断通过。
- 位置：`plugins/plugin-dev/skills/plugin-settings/scripts/validate-settings.sh:7-101`。

### 6) 真实插件中的“同模式异实现”

1. `ralph-wiggum`（状态循环）
- 创建状态文件：`plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:140-150`。
- 读取并推进状态：`plugins/ralph-wiggum/hooks/stop-hook.sh:20-156`。
- 输出 Stop 阻断 JSON：`plugins/ralph-wiggum/hooks/stop-hook.sh:167-174`。

2. `hookify`（动态规则）
- 动态读取 `.claude/hookify.*.local.md`：`plugins/hookify/core/config_loader.py:209-226`。
- 每次 hook 执行都会加载规则：`plugins/hookify/hooks/pretooluse.py:51-57`。
- README 明确“无需重启立即生效”：`plugins/hookify/README.md:28-29`。

### 7) 实测结论（本次研究执行）

执行命令（仓库根目录）：

```bash
bash plugins/plugin-dev/skills/plugin-settings/scripts/parse-frontmatter.sh \
  plugins/plugin-dev/skills/plugin-settings/examples/example-settings.md enabled

bash plugins/plugin-dev/skills/plugin-settings/scripts/parse-frontmatter.sh \
  plugins/plugin-dev/skills/plugin-settings/examples/example-settings.md nonexist

bash plugins/plugin-dev/skills/plugin-settings/scripts/validate-settings.sh <temp_invalid_bool_file>
bash plugins/plugin-dev/skills/plugin-settings/scripts/validate-settings.sh <temp_non_frontmatter_file>
```

观察：

1. `parse-frontmatter.sh ... enabled` 输出了多行 `true`，因为会抓取文档中多个 `--- ... ---` 代码块（不仅是文件开头 frontmatter）。
2. `parse-frontmatter.sh ... nonexist` 在 `set -euo pipefail` 下直接返回 1，未输出自定义“field not found”提示。
3. `validate-settings.sh` 对布尔值异常（如 `enabled: maybe`）只告警，最终仍返回 0。
4. `validate-settings.sh` 对“非 frontmatter 文件但包含两个 `---`”会进入不稳定分支并可能提前退出。

## 关键代码路径与文件引用

### 目标对象

1. `plugins/plugin-dev/skills/plugin-settings/SKILL.md`（主规范）。

### 调用方（上游）

1. `plugins/plugin-dev/README.md:114-133`
- 声明触发词、定位、资源组成。

2. `plugins/plugin-dev/commands/create-plugin.md:157-226`
- Phase 5 要求加载 plugin-settings，并落地 settings 模板/读取逻辑/gitignore。

3. `plugins/plugin-dev/skills/skill-development/SKILL.md:311-315`
- 把 plugin-settings 作为“触发词 + references + scripts”实践样例。

### 被调用方（下游资源）

1. `plugins/plugin-dev/skills/plugin-settings/examples/read-settings-hook.sh`
2. `plugins/plugin-dev/skills/plugin-settings/examples/create-settings-command.md`
3. `plugins/plugin-dev/skills/plugin-settings/examples/example-settings.md`
4. `plugins/plugin-dev/skills/plugin-settings/references/parsing-techniques.md`
5. `plugins/plugin-dev/skills/plugin-settings/references/real-world-examples.md`
6. `plugins/plugin-dev/skills/plugin-settings/scripts/parse-frontmatter.sh`
7. `plugins/plugin-dev/skills/plugin-settings/scripts/validate-settings.sh`

### 关联配置与协议文件

1. `plugins/ralph-wiggum/hooks/hooks.json:1-15`（Stop Hook 注册）。
2. `plugins/ralph-wiggum/hooks/stop-hook.sh:167-174`（Stop 阻断协议）。
3. `plugins/hookify/hooks/pretooluse.py:35-60`（PreToolUse 动态规则执行）。
4. `plugins/hookify/core/config_loader.py:87-195`（frontmatter 解析数据结构）。

### 测试与验证相关路径

1. `plugins/plugin-dev/skills/plugin-settings/scripts/validate-settings.sh`（手工校验工具）。
2. `plugins/plugin-dev/skills/plugin-settings/scripts/parse-frontmatter.sh`（提取工具）。
3. 当前目录未发现自动化测试文件（无 `test/`、`spec/`、CI 断言脚本直接覆盖该技能）。

### 文档路径（技能上下文）

1. `plugins/plugin-dev/README.md`
2. `plugins/plugin-dev/commands/create-plugin.md`
3. `plugins/plugin-dev/skills/plugin-settings/references/parsing-techniques.md`
4. `plugins/plugin-dev/skills/plugin-settings/references/real-world-examples.md`

## 依赖与外部交互

1. Shell 工具依赖
- 必需：`bash`、`sed`、`awk`、`grep`。
- 常见：`jq`（Hook JSON 解析/构造）、`perl`（promise 提取）、`tmux`（swarm 示例通知）。
- 可选：`yq`（复杂 YAML 解析，文档中建议按需使用）。

2. Claude Code 运行时交互
- Skill 路由：由 `description` 触发词决定是否加载该技能。
- Command 交互：通过 `AskUserQuestion` 与 `Write` 生成 `.local.md`。
- Hook 交互：stdin 读事件 JSON，stdout/stderr 输出决策 JSON，退出码决定行为。

3. 文件系统交互
- 主交互目录：项目根 `.claude/`。
- 文件模式：`*.local.md`（用户本地配置/状态）。
- 版本控制约束：建议写入 `.gitignore`（`plugins/plugin-dev/skills/plugin-settings/SKILL.md:329-334`）。

4. 文档与实现协同
- `plugin-settings` 文档以 Bash 文本解析为主。
- `hookify` 实现提供 Python 动态解析器和规则引擎，说明同一 `.local.md` 模式可有不同技术栈实现。

5. 外部服务交互
- 该技能本身不依赖网络服务；外部交互主要是本地 shell 与 Claude Code hook/command 协议。

## 风险、边界与改进建议

1. 风险（高）：frontmatter 提取范围过宽
- 现状：`sed -n '/^---$/,/^---$/'` 会匹配文件内每一对 marker；对文档型文件会拼接多个代码块。
- 证据：`parse-frontmatter.sh` 对 `example-settings.md` 提取 `enabled` 时返回多行。
- 建议：改为“仅解析文件开头 frontmatter”的状态机（awk/yq），并在脚本中显式拒绝非开头 marker。

2. 风险（高）：`parse-frontmatter.sh` 缺字段错误路径不可控
- 现状：`set -euo pipefail` 下，`grep` 未命中会提前退出，后续自定义错误提示可能不执行。
- 建议：对字段提取链路使用 `grep ... || true`，再统一做空值判定并输出明确错误。

3. 风险（中）：`validate-settings.sh` 的“有效”语义偏宽松
- 现状：布尔异常只 warning，仍返回 0。
- 影响：上游可能把“结构有效”误认为“配置可安全运行”。
- 建议：增加 `--strict` 模式；在 strict 下，关键字段非法返回非零。

4. 风险（中）：`validate-settings.sh` 对异常文件的分支稳定性不足
- 现状：当 `grep` 无字段匹配且开启 `pipefail` 时，字段打印阶段可能提前退出。
- 建议：字段扫描使用容错管道（例如 `grep ... || true`）并把“无字段”作为显式分支处理。

5. 风险（中）：文档与仓库实物存在漂移
- 现状：`real-world-examples.md` 引用 `multi-agent-swarm`，但当前仓库无该插件目录；`ralph-wiggum` 真实实现也比 reference 示例更严格。
- 建议：在 reference 标注“示意来源/版本”，并优先链接当前仓库可验证路径。

6. 边界（中）：`“修改后需重启”`不是全局真理
- 现状：`plugin-settings` 文档强调重启（`plugins/plugin-dev/skills/plugin-settings/SKILL.md:367-383`），但 `hookify` 明确“下一次工具调用立即生效”（`plugins/hookify/README.md:28-29`）。
- 解释：是否重启取决于插件实现是“启动时加载配置”还是“每次事件动态读取配置”。
- 建议：在 `SKILL.md` 加入条件化说明，避免过度泛化。

7. 风险（中）：缺少自动化回归测试
- 现状：仅有手工脚本，无 Bats/pytest 等自动化覆盖。
- 建议：为 `parse-frontmatter.sh`/`validate-settings.sh` 增加最小测试集，覆盖：
  - 缺字段
  - 多 frontmatter 块
  - body 含 `---`
  - 非法布尔/数值范围

8. 可维护性建议（低）
- 统一一份“字段 schema 示例”（必填/可选/类型/默认值）。
- 对复杂 YAML（嵌套对象、多行字符串）提供 `yq` 分支示例，避免文本解析误判。
- 在 `create-plugin` 工作流中增加 settings 校验步骤（调用 `validate-settings.sh` 或等价逻辑）。
