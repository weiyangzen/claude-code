# plugins/plugin-dev/skills/plugin-settings 研究

## 场景与职责

`plugin-settings` 是 `plugin-dev` 工具包中的一个技能目录，目标是为插件开发者提供统一的“项目级、本地化、可编辑”配置模式：`.claude/plugin-name.local.md`。

该目录在 `plugin-dev` 体系中的职责可拆分为三层：

1. 规范层：定义 `.local.md` 的结构约定（YAML frontmatter + Markdown body）、命名、生命周期和 gitignore 规则。见 `plugins/plugin-dev/skills/plugin-settings/SKILL.md:11-19`、`318-336`。
2. 方法层：给出在 hook/command/agent 中读取配置的可复用解析套路（`sed`/`grep`/`awk`），并提供边界处理建议。见 `plugins/plugin-dev/skills/plugin-settings/SKILL.md:60-171`。
3. 工具层：提供两个可执行脚本（`parse-frontmatter.sh` 与 `validate-settings.sh`）以及 3 个示例文件，降低落地成本。见 `plugins/plugin-dev/skills/plugin-settings/SKILL.md:521-531`。

它的上游调用关系不是“代码 import”，而是“技能路由 + 工作流指令”方式：

- 在 `plugin-dev` 总览中被列为第 4 个核心技能，触发词明确为“plugin settings/.local.md/YAML frontmatter”等。见 `plugins/plugin-dev/README.md:114-133`。
- 在 `/plugin-dev:create-plugin` 的 Phase 5 中被显式要求按需加载，并指导创建模板、读取逻辑和 gitignore。见 `plugins/plugin-dev/commands/create-plugin.md:153-226`。
- 在 `skill-development` 中被作为“高质量技能样例”引用。见 `plugins/plugin-dev/skills/skill-development/SKILL.md:311-315`、`608-613`。

## 功能点目的

本目录的功能点与其目的如下。

1. 触发与适用范围定义
- 通过 `SKILL.md` frontmatter 的 `description` 提供高精度触发短语，保证仅在“需要插件配置/状态存储”场景加载该技能。
- 位置：`plugins/plugin-dev/skills/plugin-settings/SKILL.md:1-5`。

2. 配置文件模式定义
- 统一 `.claude/plugin-name.local.md` 作为 per-project 配置和状态文件。
- 用 YAML frontmatter 存结构化字段，用 markdown body 存补充说明或可回灌 prompt。
- 位置：`plugins/plugin-dev/skills/plugin-settings/SKILL.md:11-40`。

3. 读取与消费模式
- 针对 hook：先检查文件存在，再读取 frontmatter，再按 `enabled` 快速退出。
- 针对 command：读取配置后调整执行逻辑。
- 针对 agent：在系统指令中声明“若存在则解析并调整行为”。
- 位置：`plugins/plugin-dev/skills/plugin-settings/SKILL.md:60-134`。

4. 创建与维护模式
- 指导通过命令生成设置文件模板，提示“修改后需重启 Claude Code”。
- 位置：`plugins/plugin-dev/skills/plugin-settings/SKILL.md:272-311`。

5. 质量与安全实践
- 包括默认值策略、值校验、路径遍历防护、输入转义、文件权限建议。
- 位置：`plugins/plugin-dev/skills/plugin-settings/SKILL.md:338-423`。

6. 参考与实战映射
- `references/parsing-techniques.md` 侧重解析/更新/验证细节。
- `references/real-world-examples.md` 侧重真实插件模式（multi-agent-swarm、ralph-wiggum）。
- 位置：`plugins/plugin-dev/skills/plugin-settings/SKILL.md:508-516`。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 数据结构与文件协议

核心协议是 Markdown 文件中的双区块结构：

```markdown
---
key: value
... 
---

# markdown body
...
```

- frontmatter：键值型配置（布尔、数值、字符串、简单列表）。
- body：可选自由文本，典型用途是任务描述/提示词。
- 样例：`plugins/plugin-dev/skills/plugin-settings/SKILL.md:24-40`。

### 2) 关键读取流程（Bash）

标准读取流程在多个文件中一致：

1. 存在性快速退出：`[[ ! -f "$STATE_FILE" ]] && exit 0`
2. frontmatter 提取：`sed -n '/^---$/,/^---$/{ /^---$/d; p; }'`
3. 字段提取：`grep '^field:' | sed 's/field: *//'`
4. 启用开关短路：`enabled != true` 时退出
5. 按字段驱动业务分支

位置：
- `plugins/plugin-dev/skills/plugin-settings/SKILL.md:66-95`
- `plugins/plugin-dev/skills/plugin-settings/examples/read-settings-hook.sh:10-65`

### 3) body 提取流程

采用 `awk` 计数 marker：

```bash
awk '/^---$/{i++; next} i>=2' "$FILE"
```

该模式意图是“拿到第二个 `---` 之后全部内容”，用于把 body 当作后续 prompt。
位置：`plugins/plugin-dev/skills/plugin-settings/SKILL.md:166-171`，`references/parsing-techniques.md:109-141`。

### 4) 示例 Hook 协议与退出码

`examples/read-settings-hook.sh` 展示了典型 Hook 输入输出协议：

- 从 stdin 读 JSON：`input=$(cat)`。
- 使用 `jq` 提取字段：`tool_input.file_path`、`tool_input.content`。
- 拒绝时向 stderr 输出 JSON（含 `hookSpecificOutput.permissionDecision=deny` 与 `systemMessage`），并 `exit 2`。
- 允许时 `exit 0`。

位置：`plugins/plugin-dev/skills/plugin-settings/examples/read-settings-hook.sh:29-65`。

### 5) 脚本实现细节

#### `scripts/parse-frontmatter.sh`

功能：提取 frontmatter 全量内容，或提取单字段。

- 参数：`<settings-file.md> [field-name]`
- 失败条件：文件不存在、frontmatter 为空、字段不存在
- 核心实现：
  - 提取 frontmatter：`sed -n '/^---$/,/^---$/{ /^---$/d; p; }'`
  - 字段提取：`grep "^${FIELD}:" | sed ...`

位置：`plugins/plugin-dev/skills/plugin-settings/scripts/parse-frontmatter.sh:7-58`。

#### `scripts/validate-settings.sh`

功能：结构性校验，偏“可读性/提示”导向。

- 校验项：存在、可读、`---` marker 数量、frontmatter 非空、常见字段扫描、布尔字段警告、body 是否存在。
- 返回语义：
  - 硬错误返回 1（如不存在、marker 不足）
  - 值不合规多数仅告警，最终仍返回 0

位置：`plugins/plugin-dev/skills/plugin-settings/scripts/validate-settings.sh:26-101`。

### 6) 实测结果（针对当前仓库脚本）

执行了以下实测命令：

- `bash plugins/plugin-dev/skills/plugin-settings/scripts/parse-frontmatter.sh plugins/plugin-dev/skills/plugin-settings/examples/example-settings.md`
- `bash plugins/plugin-dev/skills/plugin-settings/scripts/validate-settings.sh plugins/plugin-dev/skills/plugin-settings/examples/example-settings.md`
- 对临时文件进行“marker 不在文件开头”“布尔值非 true/false”测试

观察到：

1. 解析器会把文件内多个 `--- ... ---` 区块串联输出，不要求 frontmatter 位于文件开头。
2. 校验器只要求“至少 2 个 marker”，同样不要求位于顶部。
3. `enabled` 出现多次时，`grep` 结果会拼接多行，导致布尔检查提示内容异常（同一警告后出现多行值）。
4. 布尔值非法（如 `enabled: maybe`）只告警，不阻断通过（exit code 仍为 0）。

这与脚本实现一致，说明当前工具更偏“轻量辅助”，不是严格解析器。

### 7) 参考文档给出的扩展协议

`references/parsing-techniques.md` 补充了：

- 原子更新（临时文件 + `mv`）
- 缺省值策略
- 数值范围校验
- `yq` 作为复杂 YAML 可选方案

位置：`plugins/plugin-dev/skills/plugin-settings/references/parsing-techniques.md:190-236`、`238-299`、`453-485`。

`references/real-world-examples.md` 补充了：

- multi-agent-swarm：协调会话通知场景
- ralph-wiggum：Stop Hook 循环控制与状态推进场景

位置：`plugins/plugin-dev/skills/plugin-settings/references/real-world-examples.md:5-127`、`129-252`。

## 关键代码路径与文件引用

### 目标目录（被研究对象）

- `plugins/plugin-dev/skills/plugin-settings/SKILL.md`
- `plugins/plugin-dev/skills/plugin-settings/scripts/parse-frontmatter.sh`
- `plugins/plugin-dev/skills/plugin-settings/scripts/validate-settings.sh`
- `plugins/plugin-dev/skills/plugin-settings/examples/create-settings-command.md`
- `plugins/plugin-dev/skills/plugin-settings/examples/read-settings-hook.sh`
- `plugins/plugin-dev/skills/plugin-settings/examples/example-settings.md`
- `plugins/plugin-dev/skills/plugin-settings/references/parsing-techniques.md`
- `plugins/plugin-dev/skills/plugin-settings/references/real-world-examples.md`

### 调用方（上游）

- `plugins/plugin-dev/README.md:114-133`
  - 将 plugin-settings 定位为核心技能，声明触发词、资源组成。
- `plugins/plugin-dev/commands/create-plugin.md:153-226`
  - Phase 5 明确“Settings: Load plugin-settings skill”。
- `plugins/plugin-dev/skills/skill-development/SKILL.md:311-315`
  - 把该技能作为“trigger + references + scripts”实践样例。

### 近邻复用证据（横向依赖）

- `plugins/plugin-dev/skills/command-development/references/marketplace-considerations.md:345-406`
  - 命令开发参考中直接复用了 `.claude/plugin-name.local.md` 配置模式。
- `plugins/plugin-dev/skills/command-development/references/advanced-workflows.md:279-347`
  - 工作流状态管理也采用 `.local.md` 方案。

### 被调用方（下游资源）

- `SKILL.md` 明确引用 `examples/*` 与 `scripts/*` 作为实现模板与开发工具。
- 其中 `read-settings-hook.sh` 是可执行示例，`parse-frontmatter.sh` 和 `validate-settings.sh` 是可执行工具脚本。

### 关联真实插件（仓库内可核验）

- `plugins/ralph-wiggum/hooks/stop-hook.sh:13-177`
  - 使用 `.claude/ralph-loop.local.md` + frontmatter/body 解析驱动循环控制。
- `plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:130-150`
  - 创建状态文件并写入 frontmatter。

备注：`real-world-examples.md` 提到 `multi-agent-swarm`，但当前仓库中不存在 `plugins/multi-agent-swarm/` 目录。

## 依赖与外部交互

### 本地命令依赖

- 强依赖：`bash`、`sed`、`awk`、`grep`
- 条件依赖：`jq`（hook 示例解析 JSON）、`perl`（在真实 ralph hook 中提取 promise）、`tmux`（reference 中 swarm 通知示例）、`yq`（reference 提供的可选复杂 YAML 解析）

### 文件系统交互

- 读写路径中心：项目根目录下 `.claude/*.local.md`
- 生命周期建议：
  - 由用户或命令创建/编辑
  - 不应进入版本库，需写入 `.gitignore`
  - 修改后通常需要重启 Claude Code（文档约定）

### 与 Claude Code Hook 机制交互

- Hook 脚本通过 stdin 接受事件 JSON，通过 stdout/stderr 输出决策 JSON，并用退出码表达 allow/deny。
- 示例中用 `exit 2` 表示阻断决策。

### 测试与验证生态

- 目录内未提供自动化测试（未发现 `test/spec` 文件）。
- 主要依赖“示例脚本 + 手工验证脚本”模式。

## 风险、边界与改进建议

### 主要风险与边界

1. frontmatter 定位不严格
- 当前 `sed` 范式匹配“文件中首个 `---` 到下一个 `---`”，不是“文件开头 frontmatter”。
- 对包含多个代码块分隔符的 markdown 文档会误解析。
- 影响位置：`parse-frontmatter.sh:37`，`validate-settings.sh:41-56`。

2. 多同名字段处理不确定
- `grep '^enabled:'` 返回多行时，布尔校验会出现异常字符串比较，提示可读性差。
- 影响位置：`validate-settings.sh:76-83`。

3. 校验“通过”语义较弱
- 非法布尔值仅告警，不返回非零；可能让上游误以为配置完全正确。
- 影响位置：`validate-settings.sh:75-101`。

4. YAML 支持有限
- 当前实现是文本匹配，不支持复杂 YAML 结构（嵌套对象、缩进敏感语义、转义边界）。
- 文档虽提到 `yq`，但默认工具链仍是弱解析。

5. 文档与仓库实物存在漂移
- `real-world-examples` 引用的 `multi-agent-swarm` 不在当前仓库。
- `ralph-wiggum` 的真实实现比 reference 示例更严格（有更完整错误处理与字段校验）。

6. 缺少自动化回归
- 关键解析/校验脚本没有测试集；行为变更风险较高。

### 改进建议（按优先级）

1. 强化 frontmatter 边界约束
- 仅允许文件首行 `---` 作为起始 marker。
- 对多段 marker 文件显式报错而非隐式拼接。

2. 强化字段提取策略
- 对同名字段采用“首个/最后一个”明确策略并告警。
- 对 field 参数做正则转义，避免特殊字符导致误匹配。

3. 区分 warning 与 hard-fail 模式
- 为 `validate-settings.sh` 增加 `--strict` 选项。
- 在 strict 模式下对布尔/枚举/数值范围违规返回非零。

4. 提供可选强解析后端
- 在检测到 `yq` 可用时走强解析路径。
- 保留纯 shell fallback 以兼容最小环境。

5. 建立最小测试集
- 建议新增 `scripts/tests/`（或任意测试目录）覆盖：
  - 正常 frontmatter
  - marker 不在顶部
  - 多段 marker
  - 重复字段
  - 非法布尔/数值
  - body 包含 `---`

6. 同步 reference 与仓库状态
- 在 `real-world-examples.md` 标注“external example”或补充当前仓库内可追溯路径，降低读者误解。

