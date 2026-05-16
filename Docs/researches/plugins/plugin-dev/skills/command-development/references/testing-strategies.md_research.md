# testing-strategies.md 研究

## 场景与职责

`plugins/plugin-dev/skills/command-development/references/testing-strategies.md` 是 `command-development` 技能在“质量保障”维度的专题参考文档，定位是把命令开发从“可写出来”推进到“可验证、可发布、可回归”。

它在仓库内的职责边界与调用关系如下：

1. 上游触发场景：`/plugin-dev:create-plugin` 在组件实现阶段要求先加载 `command-development` 技能（`plugins/plugin-dev/commands/create-plugin.md:153-190`），并在验证/测试阶段明确要求验证命令是否出现在 `/help`、是否可执行（`plugins/plugin-dev/commands/create-plugin.md:285-299`）。
2. 直接文档调用方：`command-development/README.md` 的 Progressive Disclosure 将 `testing-strategies.md` 列为 references 之一（`plugins/plugin-dev/skills/command-development/README.md:94-111`，尤其 `:104`）。
3. 同级能力对齐：主技能 `SKILL.md` 已覆盖“Testing Pattern”和故障排查（`plugins/plugin-dev/skills/command-development/SKILL.md:455-525`），`testing-strategies.md` 负责把这些提示扩展为分层测试策略与自动化模板。
4. 被调用对象（文档内协议依赖）：命令 frontmatter 语义（`allowed-tools`、`model`）来自 `frontmatter-reference.md`（`plugins/plugin-dev/skills/command-development/references/frontmatter-reference.md:60-66,130-137`），命令与 hooks/MCP/agents 的集成语义来自 `plugin-features-reference.md`（`plugins/plugin-dev/skills/command-development/references/plugin-features-reference.md:330-407`）。

结论：该文件不是执行引擎代码，而是命令质量流程的“测试策略规范层”。

## 功能点目的

该文件按 7 级测试面组织目标，目的是覆盖从静态格式到运行时集成的主要失效模式。

1. Level 1 结构校验（`testing-strategies.md:11-71`）
- 目的：在运行前拦截最基础的文件错误（路径、扩展名、frontmatter 分隔符、空文件）。

2. Level 2 frontmatter 字段校验（`testing-strategies.md:73-124`）
- 目的：确保关键元数据可被命令系统正确消费，避免字段值越界（如非法 `model`）。

3. Level 3 手工调用验证（`testing-strategies.md:126-154`）
- 目的：验证“可发现 + 可执行 + 可观测”（`/help`、执行结果、debug 日志）。

4. Level 4 参数测试（`testing-strategies.md:156-208`）
- 目的：覆盖 `$1/$2/$ARGUMENTS` 的边界输入，降低参数替换错误风险。

5. Level 5 文件引用测试（`testing-strategies.md:210-244`）
- 目的：验证 `@file` 读入流程在不存在文件、大文件、多文件等场景下的稳定性。

6. Level 6 Bash 执行测试（`testing-strategies.md:246-289`）
- 目的：验证 `!\`cmd\`` 的成功路径、失败路径和权限边界（`allowed-tools`）。

7. Level 7 集成测试（`testing-strategies.md:291-339`）
- 目的：验证命令与 hooks、MCP、多命令状态流的协同行为。

此外，文档还补齐了自动化与发布前保障：

1. 自动化脚本模板（`validate-command.sh`、`validate-frontmatter.sh`、`test-commands.sh`，`testing-strategies.md:34-124,343-386`）。
2. pre-commit 与 CI 样例（`testing-strategies.md:388-452`）。
3. UX/UAT/调试清单（`testing-strategies.md:545-702`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 关键流程

建议执行顺序（由文档结构可推导）：

1. 结构预检：使用 `validate-command.sh` 做文件级快速失败（`testing-strategies.md:36-71`）。
2. 字段预检：使用 `validate-frontmatter.sh` 校验 frontmatter 关键字段（`testing-strategies.md:82-124`）。
3. 运行时手测：`claude --debug` + `/help` + 命令调用 + debug log（`testing-strategies.md:135-154`）。
4. 专项边界：参数、文件引用、Bash 注入、集成联调（`testing-strategies.md:156-339`）。
5. 持续化门禁：接入 pre-commit 与 CI（`testing-strategies.md:392-452`）。
6. 发布前验收：执行 Testing Checklist 与 UAT（`testing-strategies.md:592-635,558-590`）。

### 2) 数据结构与协议

1. 命令文件协议
- 目标路径默认 `.claude/commands/*.md`（`testing-strategies.md:22,28,351`）。
- Frontmatter 以 `---` 成对分隔（`testing-strategies.md:54-59`）。

2. frontmatter 字段协议（在本文件中的测试视角）
- `model` 值域：`sonnet|opus|haiku`（`testing-strategies.md:96-103`）。
- `allowed-tools`：当前模板仅检查字段是否存在（`testing-strategies.md:106-110`）。
- `description`：长度建议 <60，超过 80 输出 warning（`testing-strategies.md:112-120`）。

3. 交互与观测协议
- 通过 `/help` 验证命令发现（`testing-strategies.md:129-141`）。
- 通过 `~/.claude/debug-logs/latest` 观察异常（`testing-strategies.md:151-153`）。

4. 自动化协议
- 预提交：对 staged 命令文件逐个调用验证脚本（`testing-strategies.md:398-412`）。
- CI：在 workflow 中遍历 `.claude/commands/*.md`，执行结构/字段验证并检查 TODO（`testing-strategies.md:421-451`）。

### 3) 关键命令与脚本模式

1. 结构校验命令：`head/grep/test/ls`（`testing-strategies.md:21-31`）。
2. frontmatter 解析命令：`sed` 提取 frontmatter，再 `grep/cut/tr` 校验字段（`testing-strategies.md:89-120`）。
3. 文件引用压力样例：`dd` 生成 100MB 文件验证行为（`testing-strategies.md:238-243`）。
4. 性能观测样例：`date +%s%N`、`watch ps aux`（`testing-strategies.md:503-543`）。

## 关键代码路径与文件引用

### 目标对象

1. `plugins/plugin-dev/skills/command-development/references/testing-strategies.md:1-702`

### 调用方（谁在引导使用它）

1. `plugins/plugin-dev/skills/command-development/README.md:94-111`
- 将该文件列为 references 组成部分，属于渐进披露链路。

2. `plugins/plugin-dev/commands/create-plugin.md:153-190,285-299`
- 要求在命令实现与测试阶段执行验证；该策略文档为实际测试提供模板。

3. `plugins/plugin-dev/skills/command-development/SKILL.md:455-525`
- 主技能提供测试与排障的简版模式，详细流程应下钻到 references（含本文件）。

### 被调用方/上下文依赖（该文件引用哪些规则）

1. `plugins/plugin-dev/skills/command-development/references/frontmatter-reference.md:60-66,130-137`
- 对齐 `allowed-tools` 与 `model` 语义。

2. `plugins/plugin-dev/skills/command-development/references/plugin-features-reference.md:330-407`
- 对齐命令与 agents/skills/hooks 集成模式，映射到 Level 7 集成测试场景。

3. `plugins/plugin-dev/skills/command-development/references/advanced-workflows.md:281-330`
- Level 7 的“命令序列与状态管理”可复用 `.claude/*.local.md` 状态协议。

4. `plugins/plugin-dev/skills/command-development/references/interactive-commands.md:34-67,903-907`
- 交互命令依赖 AskUserQuestion 协议，测试时需覆盖 tool 参数合法性与可用性。

### 配置/测试/脚本/文档路径

1. 配置/输入路径：`.claude/commands/*.md`、`~/.claude/debug-logs/latest`（`testing-strategies.md:22,28,151-153`）。
2. 测试脚本模板名：`validate-command.sh`、`validate-frontmatter.sh`、`test-commands.sh`（`testing-strategies.md:38,84,349`）。
3. 流水线路径模板：`.git/hooks/pre-commit`、`.github/workflows/test-commands.yml`（`testing-strategies.md:394,422`）。
4. 现实仓库状态：当前仓库不存在上述脚本与 workflow 实体文件（`rg --files | rg 'validate-command\.sh|validate-frontmatter\.sh|test-commands\.sh|test-commands\.yml'` 无结果）。

## 依赖与外部交互

1. Claude Code 运行环境依赖
- 需要 `claude --debug` 与 slash command 交互会话（`testing-strategies.md:137,140-149`）。

2. Shell/系统工具依赖
- 依赖 `bash`, `head`, `grep`, `sed`, `cut`, `tr`, `test`, `dd`, `rm`, `tail`, `watch`, `ps`, `git`（分布在 `testing-strategies.md:21-452,531-543`）。

3. Git 与 CI 生态依赖
- 依赖 git staged diff 机制驱动 pre-commit（`testing-strategies.md:398-405`）。
- 依赖 GitHub Actions 执行持续检查（`testing-strategies.md:421-452`）。

4. 组件协同依赖
- 集成测试要求命令可与 hooks/MCP/多命令工作流协同（`testing-strategies.md:291-339`），其前提是这些组件在目标插件中已存在并可触发。

## 风险、边界与改进建议

### 风险

1. 文档与落地产物脱节（高）
- 本文件提供了完整脚本与 CI 模板，但当前仓库无对应实体脚本/工作流文件；团队若不手动落地，测试策略无法自动执行。

2. 校验规则覆盖不足（高）
- `validate-frontmatter.sh` 只检查 `allowed-tools` 存在性，不校验格式合法性与最小权限；对 `argument-hint`、`disable-model-invocation` 也未覆盖（`testing-strategies.md:106-110`）。

3. 路径与作用域偏项目命令（中）
- 大量示例固定 `.claude/commands` 路径（`testing-strategies.md:22,28,351,435`），对插件命令 `plugin-name/commands/` 的场景支持不足。

4. 脚本路径约定不一致（中）
- 聚合脚本用 `./validate-command.sh`（`testing-strategies.md:363`），pre-commit/CI 用 `./scripts/validate-command.sh`（`testing-strategies.md:408,437`），直接复制容易踩路径错误。

5. 可移植性细节风险（中）
- 边界样例中使用 `python -c` 生成长参数（`testing-strategies.md:475`），未说明 Python 依赖前提。

6. README 状态与事实不一致（低）
- `command-development/README.md` 仍把 Testing Strategies 标记为 in progress（`plugins/plugin-dev/skills/command-development/README.md:250-253`），与文件实际已存在的状态不一致。

### 边界

1. 该文件是“测试策略参考”，不是测试框架实现。
2. 样例多数是模板化命令，默认需要按仓库目录、工具链、权限模型二次适配。
3. Level 3-7 多为手工步骤或半自动流程，无法替代真实端到端场景验证。

### 改进建议

1. 落地可执行基线
- 在 `plugins/plugin-dev/skills/command-development/scripts/` 提供正式版 `validate-command.sh`、`validate-frontmatter.sh`、`test-commands.sh`，并让文档改为“解释 + 调用”而非仅贴模板。

2. 补齐 schema 级校验
- 为 frontmatter 建立统一 schema（至少覆盖 `description/allowed-tools/model/argument-hint/disable-model-invocation`），并与 `plugin-validator` 共享规则源。

3. 双场景路径支持
- 在测试模板中同时给出项目命令路径（`.claude/commands/`）与插件命令路径（`plugin-name/commands/`）的检查命令。

4. 统一脚本入口
- 约定单一脚本目录（例如 `scripts/`），修正文档中 `./` 与 `./scripts/` 混用。

5. 把手测步骤结构化
- 将 Level 3-7 的手工检查改写为可执行 checklist 脚本（至少输出 PASS/FAIL），减少人工解释偏差。

6. 同步 README 状态
- 更新 `command-development/README.md` 的状态段，移除“Testing strategies in progress”或改为“已提供，持续迭代”。
