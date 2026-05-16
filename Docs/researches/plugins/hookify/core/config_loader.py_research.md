# FILE `plugins/hookify/core/config_loader.py` 研究文档

## 场景与职责

`config_loader.py` 是 Hookify 的规则配置入口，负责把项目本地规则文件（`.claude/hookify.*.local.md`）转换成可执行的内存对象，并按事件筛选出当前 hook 需要的规则。

它处于执行链的“配置解析层”：

1. 上游：`hooks/pretooluse.py`、`posttooluse.py`、`stop.py`、`userpromptsubmit.py`。
2. 本层：frontmatter 解析、规则对象构建、启用状态与事件过滤。
3. 下游：`rule_engine.RuleEngine.evaluate_rules(...)` 做条件匹配和决策输出。

## 功能点目的

1. 规则数据模型统一。
- `Condition` 统一三元条件（`field/operator/pattern`）。
- `Rule` 统一规则元信息（`name/enabled/event/action/tool_matcher/message`）和 `conditions` 列表。

2. 兼容两种规则写法。
- 新写法：`conditions` 列表。
- 旧写法：`pattern` 单字段（自动转换为一个 `Condition`）。

3. markdown 规则文件拆解。
- `extract_frontmatter` 从 markdown 文本中分离 YAML frontmatter 与正文消息。
- 消息正文直接进入 `Rule.message`，供 hook 命中时输出。

4. 批量加载与按事件过滤。
- `load_rules(event)` 扫描 `.claude/hookify.*.local.md`。
- 仅返回 `enabled: true` 且事件匹配（`rule.event == event` 或 `all`）的规则。

5. 容错与降级。
- 单文件读取/解析失败打印 stderr warning 并跳过，不中断整批规则加载。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 数据结构

- `Condition` dataclass
  - `field`: `command/new_text/old_text/file_path/reason/transcript/user_prompt/...`
  - `operator`: `regex_match/contains/equals/not_contains/starts_with/ends_with`
  - `pattern`: 匹配内容
- `Rule` dataclass
  - 核心字段：`name/enabled/event/action/tool_matcher/message`
  - 兼容字段：`pattern`（legacy）+ `conditions`（modern）

`Rule.from_dict` 的关键兼容策略：

1. 若 frontmatter 有 `conditions` 且为 list，优先转为 `Condition[]`。
2. 若只有 `pattern` 且前一步未产生条件，按 `event` 推断默认字段并生成单条件：
   - `bash -> command`
   - `file -> new_text`
   - 其他 -> `content`

### 2) frontmatter 解析算法

`extract_frontmatter(content)` 使用手写解析器处理 YAML 子集：

1. 校验文件是否以 `---` 起始；否则返回 `({}, 原文)`。
2. 使用 `split('---', 2)` 分割 frontmatter 区块与正文。
3. 逐行扫描 frontmatter：
   - 识别顶层 `key: value`
   - 识别 list item（`- ...`）
   - 支持“多行 dict list item”的缩进续行
   - 支持 `true/false` 字符串转布尔
4. 返回 `(frontmatter_dict, message_body)`。

该实现不依赖第三方 YAML 库，保证轻量，但只覆盖仓库规则场景常用语法。

### 3) 批量加载流程

`load_rules(event=None)` 的关键流程：

1. 通过 `glob.glob('.claude/hookify.*.local.md')` 发现候选规则文件。
2. 逐文件调用 `load_rule_file(file_path)`：
   - 读取文件文本
   - `extract_frontmatter`
   - `Rule.from_dict(...)` 构建对象
3. 按 `event` 过滤：
   - 若 `event` 为空不过滤
   - 否则仅保留 `rule.event == event` 或 `rule.event == 'all'`
4. 仅保留 `rule.enabled == True`。
5. 解析异常按文件降级处理，继续后续文件。

### 4) 单文件加载与错误处理

`load_rule_file(file_path)` 返回 `Rule | None`：

1. `open(file_path, 'r')` 读内容。
2. 若无 frontmatter，stderr 告警并返回 `None`。
3. 成功则返回 `Rule`。
4. 针对 I/O、编码、解析等异常分别输出不同错误文本。

### 5) 与 hook 协议的连接方式

本文件不直接处理 Hook 输入输出 JSON，但它决定了 engine 可用规则集合：

1. hook 脚本从 stdin 读事件 payload。
2. hook 脚本基于事件决定 `event` 参数后调用 `load_rules(event)`。
3. `rule_engine` 对返回规则集做匹配并生成 stdout JSON。

## 关键代码路径与文件引用

- 核心实现：
  - `plugins/hookify/core/config_loader.py:15-29` (`Condition`)
  - `plugins/hookify/core/config_loader.py:32-84` (`Rule` + `from_dict`)
  - `plugins/hookify/core/config_loader.py:87-195` (`extract_frontmatter`)
  - `plugins/hookify/core/config_loader.py:198-241` (`load_rules`)
  - `plugins/hookify/core/config_loader.py:244-275` (`load_rule_file`)

- 直接调用方：
  - `plugins/hookify/hooks/pretooluse.py:26,52`
  - `plugins/hookify/hooks/posttooluse.py:22,45`
  - `plugins/hookify/hooks/stop.py:22,37`
  - `plugins/hookify/hooks/userpromptsubmit.py:22,37`

- 规则来源与格式文档：
  - `plugins/hookify/README.md:71-260`
  - `plugins/hookify/skills/writing-rules/SKILL.md:11-373`
  - `plugins/hookify/examples/*.local.md`

- 关联引擎：
  - `plugins/hookify/core/rule_engine.py:10,35`

## 依赖与外部交互

1. Python 标准库依赖：
- `os`/`glob`：规则文件发现
- `dataclasses`/`typing`：结构化规则对象
- `sys`：stderr 输出
- `re`：本文件仅用于类型与示例上下文，不做主匹配

2. 文件系统交互：
- 读取相对当前工作目录的 `.claude/hookify.*.local.md`
- 单文件读取失败不终止全局加载

3. 与其他模块交互：
- 向 `rule_engine` 提供 `Rule` / `Condition` 对象
- 被 `hooks/*.py` 在每次 hook 触发时动态调用

4. 外部协议边界：
- 不直接产出 Hook 协议 JSON
- 通过“规则是否可解析、可加载”间接决定 Hook 阶段告警/阻断行为

## 风险、边界与改进建议

1. 手写 YAML 子集解析器边界有限。
- 风险：复杂 YAML（深层嵌套、转义、复杂引号）可能被错误解析或静默降级。
- 建议：引入标准 YAML 解析 + schema 校验，或至少补充严格字段校验和行号级错误提示。

2. legacy `pattern` 在 `stop/prompt/all` 事件默认映射到 `content`。
- 风险：`rule_engine._extract_field` 对很多场景并无 `content` 字段，导致规则“已加载但不命中”。
- 建议：针对 `stop/prompt` 分别默认到 `transcript`/`user_prompt`，或在加载阶段告警要求显式 `conditions`。

3. `glob.glob` 结果未排序。
- 风险：多规则同名或信息冲突时，消息拼接顺序依赖文件系统返回顺序，跨平台可能不稳定。
- 建议：对文件路径排序后再加载，保证可复现性。

4. `enabled/event/action` 等字段缺少强类型校验。
- 风险：`enabled: "false"`（字符串）在 Python 中是真值，可能误开启规则；未知 `action` 未在加载层拦截。
- 建议：在 `Rule.from_dict` 做 schema normalization（布尔/枚举校验），无效值 fail-fast 或显式降级。

5. 编码策略未显式声明。
- 风险：`open(..., 'r')` 依赖系统默认编码，跨环境可能出现非 UTF-8 解析差异。
- 建议：显式 `encoding='utf-8'`，并在错误提示中附文件路径与建议修复方式。

6. 自动化测试缺口。
- 现状：仅 `__main__` 手工段落，无针对解析器的回归测试。
- 建议：补充参数化用例覆盖 simple/conditions/malformed/frontmatter 缺失等场景。
