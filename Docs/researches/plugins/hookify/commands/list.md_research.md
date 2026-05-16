# FILE `plugins/hookify/commands/list.md` 研究文档

## 场景与职责

`/hookify:list` 是 Hookify 的规则可观测入口，用于枚举当前项目中已配置规则及其状态。它属于只读管理命令，核心价值是“看得见当前治理策略”。

职责边界：
- 上游：用户需要确认哪些规则已生效、命中范围是什么。
- 中游：命令执行 `Glob + Read` 扫描并汇总规则。
- 下游：输出结果为 `/hookify:configure` 和手工编辑提供决策依据。

关键定位：
- 命令定义：`plugins/hookify/commands/list.md:1`
- 规则文件来源：`.claude/hookify.*.local.md`（`plugins/hookify/commands/list.md:16`）
- 运行时同源扫描：`plugins/hookify/core/config_loader.py:210`

## 功能点目的

1. 提供规则资产总览
- 通过表格展示 `Name/Enabled/Event/Pattern/File`，快速判断“是否加载了预期规则”（`plugins/hookify/commands/list.md:24-36`）。

2. 提供规则细节预览
- 每条规则给出事件、pattern、message 摘要和状态，帮助用户判断规则是否写对（`plugins/hookify/commands/list.md:38-47`）。

3. 为空项目提供引导
- 当无规则时引导用户使用 `/hookify` 或手工创建文件，降低首用门槛（`plugins/hookify/commands/list.md:62-82`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

1. 命令协议
- frontmatter 限定 `allowed-tools: ["Glob", "Read", "Skill"]`，确保命令为只读浏览（`plugins/hookify/commands/list.md:3`）。
- 首步要求加载 `hookify:writing-rules`，用于正确理解 frontmatter 字段（`plugins/hookify/commands/list.md:8`）。

2. 规则扫描流程
- 使用 Glob 模式 `.claude/hookify.*.local.md` 发现候选规则文件（`plugins/hookify/commands/list.md:14-17`）。
- Read 每个文件并提取前置字段 `name/enabled/event/pattern` 与消息前 100 字摘要（`plugins/hookify/commands/list.md:19-23`）。

3. 展示协议
- 第一层：聚合表格 + 总数统计（`plugins/hookify/commands/list.md:26-36`）。
- 第二层：每条规则的详细 preview（`plugins/hookify/commands/list.md:38-47`）。
- 第三层：管理动作提示（编辑/启停/删除/新建）（`plugins/hookify/commands/list.md:49-60`）。

4. 与运行时一致性要求
- list 的扫描路径与 runtime `load_rules()` 一致（同样是 `.claude/hookify.*.local.md`），减少“看得到/用不到”的偏差（`plugins/hookify/core/config_loader.py:209-211`）。
- 但 list 读取的是原文件字段，runtime 还会做 event 过滤与 enabled 过滤（`plugins/hookify/core/config_loader.py:219-227`），两者语义并不完全等价。

## 关键代码路径与文件引用

- 命令本体：`plugins/hookify/commands/list.md`
- 规则语法技能：`plugins/hookify/skills/writing-rules/SKILL.md`
- 规则加载实现：`plugins/hookify/core/config_loader.py`
- 规则求值实现：`plugins/hookify/core/rule_engine.py`
- Hook 执行器：`plugins/hookify/hooks/pretooluse.py`、`posttooluse.py`、`stop.py`、`userpromptsubmit.py`
- 用户文档：`plugins/hookify/README.md:283-287`
- 示例规则：`plugins/hookify/examples/*.local.md`

测试/脚本上下文：
- list 命令无专门自动化测试；当前仓库未见对“空目录/多规则/复杂条件规则”展示稳定性的回归测试。

## 依赖与外部交互

1. 文件系统依赖
- 依赖项目 `.claude/` 目录结构和命名约定。

2. 工具协议依赖
- 依赖 Glob 与 Read 的稳定输出；若其中之一失败，展示完整性下降。

3. 与其他命令交互
- 与 `/hookify` 形成“生成后可见性验证”闭环。
- 与 `/hookify:configure` 形成“查看后启停”管理闭环。

4. 外部交互范围
- 无网络依赖；外部交互仅为本地文件读取和 Claude Code 命令输出。

## 风险、边界与改进建议

1. 对复杂规则信息展示不足
- 风险：当前提取字段偏向 simple pattern，`conditions` 规则可能显示为空 pattern，用户误判规则失效。
- 建议：当 `pattern` 为空时渲染 `conditions` 摘要（如 `file_path regex_match ... AND new_text contains ...`）。

2. 缺少 `action` 可见性
- 风险：用户无法快速区分 `warn` 与 `block` 规则，治理强度不透明。
- 建议：表格新增 `Action` 列，并在预览中突出 block 规则。

3. 不做规则可解析性检查
- 风险：文件存在但 frontmatter 错误时，list 可能给出不完整信息，用户以为规则可用。
- 建议：新增“解析状态/错误”列，复用 `config_loader.load_rule_file()` 做校验。

4. 大规模规则下可读性与性能
- 边界：规则数量上升后，逐条 preview 可能过长。
- 建议：默认按 enabled 优先、支持分页或仅展开异常项。
