# FILE `plugins/hookify/commands/configure.md` 研究文档

## 场景与职责

`/hookify:configure` 是 Hookify 的“运行态规则开关”入口。它不创建新规则，也不执行 hook，而是针对项目目录 `.claude/hookify.*.local.md` 中已存在规则做交互式启停。

该命令在链路中的职责边界：
- 上游：用户在会话中发起 `/hookify:configure`。
- 中游：命令文档约束代理按 `Glob -> Read -> AskUserQuestion -> Edit` 执行。
- 下游：被编辑后的 frontmatter `enabled` 字段由 hook 运行时动态加载并立即生效。

关键定位：
- 命令定义：`plugins/hookify/commands/configure.md:1`
- 规则加载：`plugins/hookify/core/config_loader.py:198`
- 事件执行器：`plugins/hookify/hooks/pretooluse.py:51`、`plugins/hookify/hooks/stop.py:37`

## 功能点目的

1. 提供低风险启停能力
- 通过切换 `enabled: true/false` 达到“临时关闭/恢复”效果，避免直接删除规则文件（`plugins/hookify/README.md:263`）。

2. 降低手工编辑成本
- 统一扫描、展示、选择、批量更新，减少用户逐个打开文件改 frontmatter 的操作负担（`plugins/hookify/commands/configure.md:14-31`）。

3. 与“立即生效”体验保持一致
- 命令结束后无需重启，下一次 hook 触发会重新扫描规则（`plugins/hookify/commands/configure.md:108`，`plugins/hookify/core/config_loader.py:210-227`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

1. 命令协议与工具权限
- frontmatter 定义 `allowed-tools: ["Glob", "Read", "Edit", "AskUserQuestion", "Skill"]`，明确该命令可以扫描、读取、编辑规则并与用户交互（`plugins/hookify/commands/configure.md:3`）。
- 首步要求加载 `hookify:writing-rules`，确保对 frontmatter 字段语义一致（`plugins/hookify/commands/configure.md:8`，`plugins/hookify/skills/writing-rules/SKILL.md:29-52`）。

2. 规则发现与状态读取
- 使用 Glob 模式 `.claude/hookify.*.local.md`（`plugins/hookify/commands/configure.md:18`）。
- 读取每个文件并提取 `name`、`enabled` 用于界面展示（`plugins/hookify/commands/configure.md:28-31`）。

3. 交互选择与切换逻辑
- AskUserQuestion 采用 `multiSelect: true`，选项标签格式为 `{rule-name} (currently {enabled|disabled})`（`plugins/hookify/commands/configure.md:35-65`）。
- 选中项执行“反转”：enabled -> disabled，disabled -> enabled（`plugins/hookify/commands/configure.md:67-72`）。

4. 文件更新方式
- 文档建议使用文本替换：`enabled: false` <-> `enabled: true`（`plugins/hookify/commands/configure.md:75-90`）。
- 更新后输出 Enabled/Disabled/Unchanged 汇总并提示立即生效（`plugins/hookify/commands/configure.md:92-109`）。

5. 运行时消费语义
- `config_loader.load_rules()` 仅返回 `enabled` 为 true 的规则（`plugins/hookify/core/config_loader.py:224-227`）。
- 因此该命令仅通过 `enabled` 位即可影响后续 `PreToolUse/PostToolUse/Stop/UserPromptSubmit` 行为，无需改 `hooks.json`（`plugins/hookify/hooks/hooks.json:3-48`）。

## 关键代码路径与文件引用

- 命令实现文档：`plugins/hookify/commands/configure.md`
- 规则语法基线：`plugins/hookify/skills/writing-rules/SKILL.md`
- 规则文件读取与过滤：`plugins/hookify/core/config_loader.py:198-241`
- Hook 调用入口：
  - `plugins/hookify/hooks/pretooluse.py:35-57`
  - `plugins/hookify/hooks/posttooluse.py:30-50`
  - `plugins/hookify/hooks/stop.py:30-42`
  - `plugins/hookify/hooks/userpromptsubmit.py:30-42`
- 插件用户文档中的管理章节：`plugins/hookify/README.md:261-287`

测试/脚本上下文：
- `plugins/hookify/commands` 无专用自动化测试目录。
- `core` 仅有 `__main__` 手工测试段（`plugins/hookify/core/config_loader.py:277`、`plugins/hookify/core/rule_engine.py:276`）。

## 依赖与外部交互

1. 文件系统依赖
- 强依赖当前工作目录 `.claude/` 下规则文件存在性；命令本身不创建规则，只读写已有规则（`plugins/hookify/commands/configure.md:21-24`）。

2. 工具协议依赖
- 依赖 AskUserQuestion 的多选能力承载批量切换（`plugins/hookify/commands/configure.md:35-60`）。

3. 与运行时的耦合
- 与 `load_rules` 的 `enabled` 过滤逻辑强耦合；该字段是命令唯一生效杠杆（`plugins/hookify/core/config_loader.py:224-227`）。

4. 外部交互范围
- 无网络依赖；主要外部交互是 Claude Code 工具调用协议和本地文件读写。

## 风险、边界与改进建议

1. 文本替换方式脆弱
- 风险：`enabled: true`/`false` 的空格、注释、大小写、引号或重复字段会导致替换失败或误替换。
- 建议：改为解析 frontmatter 后只重写 `enabled` 键，避免纯字符串替换。

2. 仅根据 label 解析当前状态不稳
- 风险：若选项 label 文案变化，切换判断可能出错。
- 建议：AskUserQuestion 选项内部携带稳定 ID（文件路径或规则名）并从结构化数据判断状态。

3. 不显示 `action` 与 `conditions`
- 边界：复杂规则启停时用户缺少足够上下文，可能误关关键 block 规则。
- 建议：列表描述增加 `action`、条件摘要与事件类型。

4. 并发写入冲突未覆盖
- 风险：用户手工编辑与命令编辑同时发生时可能覆盖变更。
- 建议：更新前后做二次 Read 校验，或提示“检测到文件变化，请重试”。

5. 测试缺口
- 风险：文档修改后实际切换流程可能漂移。
- 建议：补充 smoke 测试，至少覆盖“多选切换 + enabled 过滤生效”闭环。
