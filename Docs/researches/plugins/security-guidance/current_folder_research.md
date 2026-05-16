# plugins/security-guidance 研究

## 场景与职责

`plugins/security-guidance` 是一个“写文件前安全提醒”插件，运行在 Claude Code 的 Hook 机制中，职责是：在 `Edit/Write/MultiEdit` 工具真正执行前，按预置规则拦截高风险编辑并给出安全提示。

- 上游调用方：Claude Code Hook 运行时（`PreToolUse` 事件）
- 本目录核心职责：
  - 声明 Hook 注册与匹配范围
  - 实现安全规则匹配与阻断逻辑
  - 维护会话内提醒去重状态
- 下游被调用对象：
  - Python 解释器执行 hook 脚本
  - 本地状态文件 `~/.claude/security_warnings_state_<session>.json`
  - stderr 输出提醒文本并以退出码阻断

定位上它是“轻量防误操作保护层”，不是 SAST/DAST，不做完整代码审计，也不做修复建议自动落地。

## 功能点目的

1. 拦截可疑写入行为
- 覆盖 9 类规则：GitHub Actions workflow 注入、`exec`、`new Function`、`eval`、`dangerouslySetInnerHTML`、`document.write`、`innerHTML`、`pickle`、`os.system`。

2. 在调用前中断并反馈安全提醒
- 命中规则时通过 stderr 输出提醒，并 `exit 2` 让 `PreToolUse` 阶段阻断本次工具调用。

3. 避免同会话重复打扰
- 同一 `session_id` 下，同一文件+规则组合只提醒一次（`file_path-ruleName` 作为 key）。

4. 提供全局开关
- 通过 `ENABLE_SECURITY_REMINDER=0` 临时关闭提醒逻辑。

5. 做有限状态清理
- 按 10% 概率触发清理逻辑，删除 30 天前的历史状态文件。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) Hook 注册与触发范围

`hooks/hooks.json` 采用插件包装格式，在 `PreToolUse` 上注册 command hook，并用 matcher 限定只处理写文件相关工具：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write|MultiEdit",
        "hooks": [
          {
            "type": "command",
            "command": "python3 ${CLAUDE_PLUGIN_ROOT}/hooks/security_reminder_hook.py"
          }
        ]
      }
    ]
  }
}
```

这意味着 Bash/Read/其它工具不会经过此插件逻辑。

### 2) 规则数据结构

核心规则集是 `SECURITY_PATTERNS`（list[dict]），每条规则包含：

- `ruleName`: 规则唯一标识
- `path_check` 或 `substrings`: 路径匹配或内容子串匹配
- `reminder`: 命中后输出的提醒文案

当前实现是“顺序扫描 + 首条命中即返回”，规则顺序直接影响结果。

### 3) 主流程

`main()` 的执行链路：

1. 读取环境变量 `ENABLE_SECURITY_REMINDER`，为 `0` 则直接放行（`exit 0`）。
2. 以 10% 概率调用 `cleanup_old_state_files()` 清理旧状态文件。
3. 从 stdin 读取 JSON（hook input）；JSON 解析失败时 fail-open（`exit 0`）。
4. 提取 `session_id/tool_name/tool_input`。
5. 非 `Edit/Write/MultiEdit` 直接放行。
6. 从 `tool_input.file_path` 取目标文件；为空则放行。
7. 依据工具类型提取待检查内容：
   - `Write` -> `tool_input.content`
   - `Edit` -> `tool_input.new_string`
   - `MultiEdit` -> 拼接 `edits[*].new_string`
8. `check_patterns(file_path, content)` 做路径+内容匹配。
9. 命中后构造 `warning_key = "<file_path>-<ruleName>"`，读取会话状态去重。
10. 若首次命中：写入状态文件、打印提醒到 stderr、`exit 2` 阻断；否则放行。

### 4) 状态与清理策略

- 状态文件命名：`~/.claude/security_warnings_state_<session_id>.json`
- 存储内容：已提示 warning key 列表
- 清理策略：随机触发 + mtime 超过 30 天删除

该策略是“近似清理”，不是严格按会话生命周期回收。

### 5) Hook 协议与退出码约定

本插件依赖 Claude Code command hook 协议：

- 输入：stdin JSON，包含通用字段（`session_id`、`hook_event_name` 等）与事件字段（`tool_name`、`tool_input`）
- 输出：stderr 提示 + 退出码
- 退出码语义：`0` 放行，`2` 阻断（PreToolUse）

相关协议参考：
- `plugins/plugin-dev/skills/hook-development/SKILL.md`
- `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh`
- `examples/hooks/bash_command_validator_example.py`

### 6) 实测验证（本次研究执行）

使用命令行直接向脚本喂入 hook JSON：

- `eval` 首次命中：`exit=2`，输出 eval 风险提醒
- 同会话同文件同规则再次命中：`exit=0`（去重生效）
- `.github/workflows/*.yml` 路径命中：`exit=2`，输出 GitHub Actions 注入提醒
- `ENABLE_SECURITY_REMINDER=0`：`exit=0`
- 非法 JSON 输入：`exit=0`（fail-open）

## 关键代码路径与文件引用

### 目标目录

- `plugins/security-guidance/.claude-plugin/plugin.json`
  - 插件元数据（名称、版本、作者、描述）。
- `plugins/security-guidance/hooks/hooks.json`
  - `PreToolUse` 注册、matcher 范围、command 执行入口。
- `plugins/security-guidance/hooks/security_reminder_hook.py`
  - `SECURITY_PATTERNS` 规则表：`31-126`
  - 状态文件函数：`129-180`
  - 匹配逻辑：`183-199`
  - 内容提取：`202-214`
  - 主流程与阻断：`217-276`

### 上下文依赖（调用方/协议/文档/测试工具）

- `plugins/README.md:27`
  - 声明该插件用途与“9 类安全模式”定位。
- `plugins/plugin-dev/skills/hook-development/SKILL.md:123-153,294-331`
  - `PreToolUse` 输出与 exit code 语义、stdin 输入字段、`${CLAUDE_PLUGIN_ROOT}` 约定。
- `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh:30-44,170-173,205-211`
  - 生成 `PreToolUse` 样例输入、注入环境变量、退出码解释。
- `plugins/plugin-dev/skills/hook-development/scripts/README.md:29-61`
  - hook 测试与校验脚本的推荐工作流。
- `examples/hooks/bash_command_validator_example.py:56-79`
  - `PreToolUse` 脚本读取 `tool_input`、`exit 2` 阻断的最小参考实现。

## 依赖与外部交互

### 运行时依赖

- `python3`（由 `hooks.json` command 直接调用）
- Python 标准库：`json`、`os`、`random`、`sys`、`datetime`

### 环境变量

- `CLAUDE_PLUGIN_ROOT`：用于 command 中定位脚本
- `ENABLE_SECURITY_REMINDER`：功能开关（`0` 关闭）

### 文件系统交互

- 读取 stdin JSON（hook input）
- 写入/读取 `~/.claude/security_warnings_state_<session>.json`
- 可能写调试日志 `/tmp/security-warnings-log.txt`（仅异常分支）

### 外部交互

- 脚本本身无网络请求
- 提醒文案包含外链（GitHub blog 安全文章），仅作为给用户的参考文本

### 测试与脚本现状

- `plugins/security-guidance` 目录内无专属测试、无 README、无辅助脚本
- 可复用仓库内通用 hook 工具链做验证：
  - `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh`
  - `plugins/plugin-dev/skills/hook-development/scripts/hook-linter.sh`

## 风险、边界与改进建议

1. 规则匹配粒度较粗，误报/漏报并存
- 现状：大量规则用简单子串匹配（如 `pickle`、`eval(`、`exec(`）。
- 风险：注释/文档字符串也会触发；变体写法可能漏检。
- 建议：按语言引入最小语法感知（正则边界或 AST-lite），至少减少明显误报。

2. 首条命中即返回，无法一次展示多个风险
- 现状：`check_patterns()` 命中后立即返回。
- 风险：用户一次编辑包含多类风险时只看到首条，修复反馈循环变慢。
- 建议：改为收集多命中并统一输出（可限制最多 N 条）。

3. 去重 key 仅 `file_path-ruleName`
- 现状：同会话内该 key 命中过一次后不再提醒。
- 风险：同文件后续新增新的同类风险也被静默放行。
- 建议：在 key 中加入内容摘要（如哈希片段）或最近触发时间窗口。

4. fail-open 策略过于宽松
- 现状：JSON 解析失败、状态读写失败基本都放行。
- 风险：输入异常/状态异常时保护失效且不易被感知。
- 建议：至少输出结构化告警（stderr/日志可观测），并支持可选 fail-closed 模式。

5. 提醒文案含仓库外路径假设
- 现状：文案建议 `src/utils/execFileNoThrow.ts` 与 `../utils/execFileNoThrow.js`，本仓库不存在该路径。
- 风险：误导使用者，降低建议可信度。
- 建议：改为通用安全建议，或基于当前仓库可用工具动态文案。

6. 缺少超时配置与测试资产
- 现状：`hooks.json` 未设置 `timeout`；目录无单测/回归测试。
- 风险：异常情况下可能拖慢 hook 链路，升级改动缺乏回归保护。
- 建议：
  - 在 `hooks.json` 增加合理超时（例如 10s）
  - 增加最小测试集（命中/未命中/去重/开关/JSON 异常）
  - 补充 `README.md` 说明规则、开关、边界与测试方式

7. 调试日志落地到 `/tmp`
- 现状：异常日志写入固定文件 `/tmp/security-warnings-log.txt`。
- 风险：多用户环境下可读性与隐私边界不明确，且可能无限增长。
- 建议：改为显式 debug 开关控制，并增加日志轮转/大小上限。

