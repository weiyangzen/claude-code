# FILE `plugins/hookify/matchers/__init__.py` 研究文档

## 场景与职责

`plugins/hookify/matchers/__init__.py` 当前是一个 **0 字节的空文件**，位于 `hookify/matchers` 目录下，直接运行职责为“Python 包初始化占位”。

在现有代码中，它不承载任何函数、类、常量，也没有被 import 到运行链路中；但从目录结构和命名上，它代表了 Hookify 在架构上预留的“匹配器（matcher）模块边界”。

可定位到的现状证据：

- 文件本体为空：`plugins/hookify/matchers/__init__.py`
- 同类占位文件存在于 `core/hooks/utils`：  
  `plugins/hookify/core/__init__.py`、`plugins/hookify/hooks/__init__.py`、`plugins/hookify/utils/__init__.py`（均为 0 字节）
- 仓库内无 `hookify.matchers` 的实际导入调用（`rg` 检索无结果）

结论：该文件是“包边界声明”，不是“功能实现点”。

## 功能点目的

虽然文件为空，但在插件体系里有三个明确目的：

1. 维持插件分层结构完整性

- `plugins/hookify` 目录按 `commands/agents/skills/hooks/core/matchers/utils` 分层。
- 该文件确保 `matchers` 可被视为包（尤其在历史 Python 兼容语境下），便于未来模块化扩展。

2. 为匹配逻辑拆分预留命名空间

- 目前匹配逻辑集中在 `plugins/hookify/core/rule_engine.py`（工具匹配、条件运算、字段提取）。
- `matchers/__init__.py` 提供了未来迁移到 `hookify.matchers.*` 的落点，不改变上层调用入口（hooks -> rule_engine）。

3. 与规则 DSL 的语义对齐

- 规则模型已包含 matcher 语义字段：`Rule.tool_matcher`（`plugins/hookify/core/config_loader.py:41,82`）。
- 运行时已有工具匹配函数 `_matches_tool(...)`（`plugins/hookify/core/rule_engine.py:127-143`）。
- 说明“matcher”概念已存在于数据模型和执行引擎，只是尚未抽离到 `matchers` 包。

## 具体技术实现（关键流程/数据结构/协议/命令）

`__init__.py` 本身无代码实现；因此需要从其上下游还原实际技术链路，确认它当前不在执行路径。

### 1) 关键流程（调用方 -> 核心引擎）

1. Claude Hook 事件由 `hooks/hooks.json` 绑定命令脚本：
   - `PreToolUse` -> `python3 ${CLAUDE_PLUGIN_ROOT}/hooks/pretooluse.py`
   - `PostToolUse` -> `python3 ${CLAUDE_PLUGIN_ROOT}/hooks/posttooluse.py`
   - `Stop` -> `python3 ${CLAUDE_PLUGIN_ROOT}/hooks/stop.py`
   - `UserPromptSubmit` -> `python3 ${CLAUDE_PLUGIN_ROOT}/hooks/userpromptsubmit.py`  
   见 `plugins/hookify/hooks/hooks.json:1-49`

2. 各 hook 脚本统一读取 stdin JSON，并调用：
   - `load_rules(event=...)`
   - `RuleEngine().evaluate_rules(rules, input_data)`  
   见：
   - `plugins/hookify/hooks/pretooluse.py:35-57`
   - `plugins/hookify/hooks/posttooluse.py:30-50`
   - `plugins/hookify/hooks/stop.py:30-42`
   - `plugins/hookify/hooks/userpromptsubmit.py:30-42`

3. `load_rules` 动态扫描 `.claude/hookify.*.local.md`，过滤 event、enabled：
   - `plugins/hookify/core/config_loader.py:198-241`

4. `RuleEngine` 内部完成全部 matcher 行为：
   - 工具匹配：`_matches_tool`（`plugins/hookify/core/rule_engine.py:127-143`）
   - 条件匹配：`_check_condition`（`144-181`）
   - 字段提取：`_extract_field`（`182-254`）
   - 正则缓存：`compile_regex` + `_regex_match`（`13-25`, `256-273`）

5. 输出 Hook 协议 JSON：
   - Stop 阻断：`{"decision":"block", ...}`
   - Pre/PostToolUse 阻断：`hookSpecificOutput.permissionDecision="deny"`
   - warn：仅返回 `systemMessage`  
   见 `plugins/hookify/core/rule_engine.py:65-91`

结论：当前运行流程没有任何 `hookify.matchers` 导入路径，`matchers/__init__.py` 不参与执行。

### 2) 关键数据结构（matcher 语义实际落点）

1. `Condition`（`plugins/hookify/core/config_loader.py:15-29`）
- 字段：`field/operator/pattern`
- 代表单个匹配条件

2. `Rule`（`plugins/hookify/core/config_loader.py:32-84`）
- matcher 相关字段：
  - `conditions: List[Condition]`
  - `pattern`（legacy，转为 condition）
  - `tool_matcher`（工具级前置匹配）

3. 规则语义转换
- `pattern` 在 `Rule.from_dict` 中按 event 推断为 condition：
  - bash -> `field=command`
  - file -> `field=new_text`
  - 其他 -> `field=content`  
  见 `plugins/hookify/core/config_loader.py:56-73`

### 3) 协议与命令

1. Hook 输入协议
- 各脚本 `json.load(sys.stdin)` 读取 hook runtime 输入  
  见 `plugins/hookify/hooks/*.py`

2. Hook 输出协议
- 一律 `print(json.dumps(result))` 输出 JSON 到 stdout  
  见 `pretooluse.py:58-59`、`posttooluse.py:51-52`、`stop.py:43-44`、`userpromptsubmit.py:43-44`

3. 错误策略
- hook 脚本 `finally: sys.exit(0)`，避免因插件错误阻塞主流程  
  见 `pretooluse.py:68-70`、`posttooluse.py:60-62`、`stop.py:53-55`、`userpromptsubmit.py:52-54`

4. 规则来源约束
- 仅扫描当前项目 `.claude/hookify.*.local.md`（非插件目录）  
  见 `plugins/hookify/core/config_loader.py:209-211`、`plugins/hookify/commands/hookify.md:128-138`

## 关键代码路径与文件引用

### 目标对象

- `plugins/hookify/matchers/__init__.py`（空文件，0 bytes）

### 调用链关键文件

- `plugins/hookify/hooks/hooks.json:1-49`
- `plugins/hookify/hooks/pretooluse.py:12-70`
- `plugins/hookify/hooks/posttooluse.py:12-62`
- `plugins/hookify/hooks/stop.py:12-55`
- `plugins/hookify/hooks/userpromptsubmit.py:12-54`
- `plugins/hookify/core/config_loader.py:15-84`
- `plugins/hookify/core/config_loader.py:198-275`
- `plugins/hookify/core/rule_engine.py:35-125`
- `plugins/hookify/core/rule_engine.py:127-273`

### 配置与文档路径

- `plugins/hookify/README.md:71-128`（规则结构与 event）
- `plugins/hookify/README.md:235-260`（operator 与 field）
- `plugins/hookify/commands/hookify.md:82-157`（规则创建与落盘路径）
- `plugins/hookify/commands/list.md:14-22`（规则发现模式）
- `plugins/hookify/commands/configure.md:16-31`（规则启停流程）
- `plugins/hookify/commands/help.md:18-49`（hook 工作原理）
- `plugins/hookify/skills/writing-rules/SKILL.md:13-99`（规则 DSL）
- `plugins/hookify/examples/*.local.md`（示例规则）

### 测试/脚本上下文

- `plugins/hookify` 目录无 `tests/`；无 `pytest` 或单测入口
- 可复用通用脚本：
  - `plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh:69-75`
  - `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh`（README 说明）
  - `plugins/plugin-dev/skills/hook-development/scripts/hook-linter.sh`（README 说明）

## 依赖与外部交互

### 运行时依赖

- Python 3（hook 命令均为 `python3 ...`）
- 仅 Python 标准库：`json/re/glob/os/sys/dataclasses/functools`
- 环境变量：`CLAUDE_PLUGIN_ROOT` 用于修正 `sys.path`  
  见 `plugins/hookify/hooks/pretooluse.py:14-23`（其他 hook 同构）

### 文件系统交互

- 读取 `.claude/hookify.*.local.md`  
  见 `plugins/hookify/core/config_loader.py:209-211`
- Stop 事件按需读取 `transcript_path` 文件  
  见 `plugins/hookify/core/rule_engine.py:207-225`

### 协议交互

- 入站：hook runtime 通过 stdin 传 JSON
- 出站：stdout 返回 JSON 决策结果
- 错误日志：stderr 打 warning/error（解析失败、regex 非法、文件读失败）

`plugins/hookify/matchers/__init__.py` 当前不直接参与任何 I/O，只在包结构层面间接参与可导入性。

## 风险、边界与改进建议

1. 风险：命名语义与实现位置脱节
- 现状：`matchers` 目录存在，但 matcher 实现都在 `core/rule_engine.py`。
- 影响：维护者会误判修改入口，增加定位与重构成本。
- 建议：在 `plugins/hookify/matchers/__init__.py` 增加模块说明（当前空实现、未来拆分计划）。

2. 风险：文档与实现能力不完全一致
- 现状：`plugin-dev` 的 hook schema 校验器要求 `hooks.json` 事件项必须有 `matcher`（`validate-hook-schema.sh:69-75`），但 `hookify/hooks/hooks.json` 未声明该字段。
- 影响：使用通用脚本校验 hookify 时可能误报失败。
- 建议：统一 schema 约定（要么补 matcher 字段，要么放宽校验脚本）。

3. 风险：`tool_matcher` 已支持但缺少用户侧文档
- 现状：`Rule` 支持 `tool_matcher`，`RuleEngine` 也实现匹配；但 README/技能文档未系统说明该字段。
- 影响：能力存在但难以被正确使用，可能引发误配置。
- 建议：在 `README.md` 与 `writing-rules/SKILL.md` 新增 `tool_matcher` 段落与示例。

4. 边界：当前 matcher 能力边界
- 支持：
  - 工具匹配：`*` 或 `A|B|C` 精确命中（非 regex）
  - 条件 operator：`regex_match/contains/equals/not_contains/starts_with/ends_with`
- 不支持：
  - 在 `tool_matcher` 上使用正则
  - matcher 级插件化扩展点（无 `hookify.matchers.*` 实现）

5. 改进路线（可渐进）

1. 最小改动：文档化
- 补 `matchers/__init__.py` 注释与 `matchers/README.md`，定义 matcher 术语与边界。

2. 中等改动：代码拆分
- 将 `_matches_tool/_check_condition/_extract_field` 迁移到 `hookify/matchers/` 下独立模块；
- `RuleEngine` 仅做流程编排与结果聚合。

3. 质量改动：补测试
- 为 matcher 语义增加回归测试矩阵（operator、tool_matcher、event 字段映射、stop transcript 异常路径）。
- 在 CI 中补一个 hookify 专项验证脚本，避免只依赖通用 plugin-dev 脚本导致的 schema 偏差。
