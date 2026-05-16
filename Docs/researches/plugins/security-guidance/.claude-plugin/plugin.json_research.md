# FILE `plugins/security-guidance/.claude-plugin/plugin.json` 研究文档

## 场景与职责

`plugins/security-guidance/.claude-plugin/plugin.json` 是 `security-guidance` 插件的 manifest 入口，核心职责是让插件可被 Claude Code 识别、注册并进入默认组件加载链路，而不是直接承载安全检测逻辑（`plugins/security-guidance/.claude-plugin/plugin.json:1-9`，`plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:11-16`）。

在上下文中的位置：

1. 调用方（上游）
- Claude Code 插件发现阶段先读取 `.claude-plugin/plugin.json`（`plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:11-16`，`plugins/plugin-dev/skills/plugin-structure/SKILL.md:343-348`）。
- 仓库 marketplace 条目通过 `source: "./plugins/security-guidance"` 指向该插件根目录（`.claude-plugin/marketplace.json:139-147`）。

2. 被调用方（下游）
- 该 manifest 未定义 `hooks` 自定义路径，因此依赖默认 `./hooks/hooks.json`（`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:259-263,356-361`）。
- 下游 `hooks/hooks.json` 在 `PreToolUse` 事件对 `Edit|Write|MultiEdit` 调起 Python hook（`plugins/security-guidance/hooks/hooks.json:4-13`）。
- 实际安全策略执行在 `hooks/security_reminder_hook.py`（`plugins/security-guidance/hooks/security_reminder_hook.py:31-280`）。

3. 配置、测试、脚本、文档关系
- 配置：`plugin.json`（元数据）+ `hooks/hooks.json`（事件绑定）。
- 测试：插件目录内无单测/集成测试文件（`find plugins/security-guidance -maxdepth 2 -type f` 仅 3 个生产文件）。
- 脚本：可复用 `plugin-dev` 的 `test-hook.sh`、`validate-hook-schema.sh` 做验证（`plugins/plugin-dev/skills/hook-development/scripts/README.md:5-27,29-53`）。
- 文档：插件总览与职责说明在 `plugins/README.md`（`plugins/README.md:27,49-60,67-69`）。

## 功能点目的

### 1) 插件身份与唯一性声明
- `name: "security-guidance"` 作为插件唯一标识（`plugins/security-guidance/.claude-plugin/plugin.json:2`）。
- 命名符合 kebab-case 规范（`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:15-36`）。

### 2) 版本可治理
- `version: "1.0.0"` 提供语义版本基线，支持升级/兼容管理（`plugins/security-guidance/.claude-plugin/plugin.json:3`，`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:42-63`）。

### 3) 能力定位可发现
- `description` 明确该插件定位是“编辑时安全提醒，覆盖命令注入、XSS、危险代码模式”（`plugins/security-guidance/.claude-plugin/plugin.json:4`）。
- 与仓库清单与 marketplace 一致，降低分发语义漂移（`plugins/README.md:27`，`.claude-plugin/marketplace.json:139-147`）。

### 4) 元数据最小集策略
- 当前仅含 `name/version/description/author`，不显式配置 `hooks/mcpServers/commands/agents/skills` 路径（`plugins/security-guidance/.claude-plugin/plugin.json:1-9`）。
- 目的是依赖默认自动发现，减少 manifest 复杂度（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:365-368`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 关键流程

1. 插件被启用后，运行时读取 `.claude-plugin/plugin.json` 并注册插件元数据（`plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:11-16`）。
2. 因 manifest 未覆写 `hooks` 字段，运行时按默认路径加载 `hooks/hooks.json`（`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:259-263,356-361`）。
3. `hooks/hooks.json` 注册 `PreToolUse`，matcher 为 `Edit|Write|MultiEdit`，命令为 `python3 ${CLAUDE_PLUGIN_ROOT}/hooks/security_reminder_hook.py`（`plugins/security-guidance/hooks/hooks.json:4-13`）。
4. Hook 读取 stdin JSON（`session_id/tool_name/tool_input`），提取编辑内容并匹配 `SECURITY_PATTERNS`（`plugins/security-guidance/hooks/security_reminder_hook.py:31-126,230-257`）。
5. 命中后首次告警写入会话状态文件并以 exit code 2 阻断本次工具调用（`plugins/security-guidance/hooks/security_reminder_hook.py:129-179,258-273`）。
6. 未命中或已提示过则 exit 0 放行（`plugins/security-guidance/hooks/security_reminder_hook.py:265-276`）。

### B. 关键数据结构

1. Manifest 结构（本研究对象）
```json
{
  "name": "security-guidance",
  "version": "1.0.0",
  "description": "Security reminder hook ...",
  "author": {
    "name": "David Dworken",
    "email": "dworken@anthropic.com"
  }
}
```
对应文件：`plugins/security-guidance/.claude-plugin/plugin.json:1-9`。

2. Hook 配置结构（被调用方）
- 插件 wrapper 格式：`{"description":..., "hooks": {...}}`（`plugins/security-guidance/hooks/hooks.json:1-16`，`plugins/plugin-dev/skills/hook-development/SKILL.md:62-80`）。

3. 规则结构（执行层）
- `SECURITY_PATTERNS` 是规则数组，元素包含：
  - `ruleName`
  - `path_check` 或 `substrings`
  - `reminder`
（`plugins/security-guidance/hooks/security_reminder_hook.py:31-126`）。

4. 会话状态结构
- 状态文件：`~/.claude/security_warnings_state_<session_id>.json`（`plugins/security-guidance/hooks/security_reminder_hook.py:129-132`）。
- 记录 key 形式：`<file_path>-<ruleName>`（`plugins/security-guidance/hooks/security_reminder_hook.py:259-270`）。

### C. 协议与命令

1. Hook 输入协议
- 命令 hook 通过 stdin 接收 JSON；PreToolUse 关键字段包括 `tool_name`、`tool_input`（`plugins/plugin-dev/skills/hook-development/SKILL.md:300-317`）。

2. 环境变量协议
- 路径使用 `${CLAUDE_PLUGIN_ROOT}` 保证可移植（`plugins/security-guidance/hooks/hooks.json:9`，`plugins/plugin-dev/skills/hook-development/SKILL.md:326-337`）。
- 运行开关：`ENABLE_SECURITY_REMINDER=0` 时整体禁用（`plugins/security-guidance/hooks/security_reminder_hook.py:220-224`）。

3. 返回码协议
- 该 hook 用 `sys.exit(2)` 阻断，用 `sys.exit(0)` 放行（`plugins/security-guidance/hooks/security_reminder_hook.py:273,276`）。
- 这是 PreToolUse 常见阻断语义（`examples/hooks/bash_command_validator_example.py:78-79`，`plugins/plugin-dev/skills/hook-development/SKILL.md:176-179`）。

### D. 实测命令与结果（本次研究）

1. 安全样例（未命中）
```bash
python3 plugins/security-guidance/hooks/security_reminder_hook.py < /tmp/security_hook_input_safe.json
```
结果：`exit=0`，stdout/stderr 均为空。

2. `eval(` 命中样例
```bash
python3 plugins/security-guidance/hooks/security_reminder_hook.py < /tmp/security_hook_input_eval.json
```
结果：`exit=2`，stderr 输出 eval 安全警告。

3. GitHub Actions 工作流路径命中
```bash
python3 plugins/security-guidance/hooks/security_reminder_hook.py < /tmp/security_hook_input_actions.json
```
结果：`exit=2`，stderr 输出 workflow 注入风险提醒。

4. 同会话重复命中去重
```bash
python3 plugins/security-guidance/hooks/security_reminder_hook.py < /tmp/security_hook_input_eval.json
```
再次执行结果：`exit=0`，无重复提醒；状态文件存在：`~/.claude/security_warnings_state_research-eval.json`，内容示例 `[
  "src/unsafe.ts-eval_injection"
]`。

5. 全局开关禁用验证
```bash
ENABLE_SECURITY_REMINDER=0 python3 plugins/security-guidance/hooks/security_reminder_hook.py < /tmp/security_hook_input_actions.json
```
结果：`exit=0`，无提醒。

6. Schema 校验脚本兼容性
```bash
bash plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh plugins/security-guidance/hooks/hooks.json
```
结果：退出码 `5`，并出现 `Cannot index string with number`；原因是脚本按“事件在 JSON 顶层”遍历，而插件 `hooks.json` 使用 wrapper 格式（脚本逻辑见 `plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh:41-67`，格式规范见 `plugins/plugin-dev/skills/hook-development/SKILL.md:62-80`）。

## 关键代码路径与文件引用

### 目标对象
- `plugins/security-guidance/.claude-plugin/plugin.json:1-9`

### 调用方（上游）
- `.claude-plugin/marketplace.json:139-147`（目录索引与分发入口）
- `plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:11-16`（插件发现生命周期）
- `plugins/plugin-dev/skills/plugin-structure/SKILL.md:343-348`（自动发现步骤）

### 被调用方（下游）
- `plugins/security-guidance/hooks/hooks.json:4-13`（PreToolUse 绑定）
- `plugins/security-guidance/hooks/security_reminder_hook.py:217-276`（运行主流程）
- `plugins/security-guidance/hooks/security_reminder_hook.py:31-126`（规则库）

### 配置/测试/脚本/文档上下文
- 配置：
  - `plugins/security-guidance/.claude-plugin/plugin.json`
  - `plugins/security-guidance/hooks/hooks.json`
- 测试与验证脚本：
  - `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh:25-45,170-218`
  - `plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh:41-67`
- 文档：
  - `plugins/README.md:27,49-60,67-69`
  - `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:7-10,259-263,356-361`
  - `plugins/plugin-dev/skills/hook-development/SKILL.md:62-80,300-317,326-337`

## 依赖与外部交互

1. 运行时依赖
- `plugin.json` 自身仅是 JSON 元数据；实际执行依赖 Python 3（`plugins/security-guidance/hooks/hooks.json:9`）。

2. 内部交互依赖
- 依赖 Claude Code 对 `.claude-plugin/plugin.json` 固定路径和默认目录扫描规则（`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:7-10,356-361`）。
- 依赖 hook 系统将 `PreToolUse` 输入 JSON 注入命令 hook（`plugins/plugin-dev/skills/hook-development/SKILL.md:300-317`）。

3. 外部交互
- 文件系统写入：
  - `~/.claude/security_warnings_state_*.json`（会话去重状态）
  - `/tmp/security-warnings-log.txt`（调试日志，异常吞掉）
  （`plugins/security-guidance/hooks/security_reminder_hook.py:14,129-179`）
- 网络：本插件链路本身不做网络请求。

## 风险、边界与改进建议

### 风险

1. 规则误报/漏报风险
- 当前以路径和子串匹配为主（如 `eval(`、`pickle`），缺少语义分析，易产生误报与漏报（`plugins/security-guidance/hooks/security_reminder_hook.py:71-124,183-199`）。

2. 单次匹配早停风险
- `check_patterns` 命中首条后立即返回，只提示一条规则，可能掩盖同次编辑中的其他高风险信号（`plugins/security-guidance/hooks/security_reminder_hook.py:188-199`）。

3. 规则文案与仓库耦合风险
- 提示中硬编码“本仓库安全替代路径 `src/utils/execFileNoThrow.ts`”，当前仓库并不存在该路径，可能给出误导性建议（`plugins/security-guidance/hooks/security_reminder_hook.py:74-81`，`rg --files` 未命中该文件）。

4. 校验工具错配风险
- 官方 utility `validate-hook-schema.sh` 对插件 wrapper 格式不兼容，可能导致“配置有效但工具报错”的治理噪音（`plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh:41-67`）。

5. 文档与治理资产不足
- 插件目录缺少 README 和自动化测试，维护者需要跨目录理解行为，回归风险高（`plugins/security-guidance` 仅 3 个文件，且 `plugins/README.md:67-69` 建议插件应有 README）。

### 边界

1. `plugin.json` 仅负责身份/元数据，不承担策略执行。
2. 安全决策边界在 hook 脚本，不在 manifest。
3. 是否阻断由 hook exit code 决定，manifest 不直接定义 deny/allow 逻辑。

### 改进建议

1. 为 `security_reminder_hook.py` 增加自动化测试
- 覆盖路径命中、内容命中、重复去重、`ENABLE_SECURITY_REMINDER` 开关、异常输入容错。

2. 升级规则匹配模型
- 从纯子串匹配升级到“子串 + 轻量 AST/上下文”策略，降低误报。

3. 支持多规则聚合输出
- 在一次编辑中收集多个命中并统一输出，避免首条规则掩盖后续风险。

4. 解耦仓库特定建议
- 将 `execFileNoThrow` 等仓库特定路径改为可配置文案，或回退到通用建议模板。

5. 修复 `validate-hook-schema.sh`
- 先判断是否存在 `hooks` wrapper，再下钻校验事件结构，兼容插件格式与 settings 直写格式。

6. 补充插件 README
- 明确“manifest -> hooks.json -> python hook”调用图、环境变量、阻断语义与排障方式。
