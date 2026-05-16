# DIR `plugins/learning-output-style/hooks` 研究文档

## 场景与职责
`plugins/learning-output-style/hooks` 是该插件的 Hook 配置目录，当前仅包含 `hooks.json`，职责是把插件意图声明为可执行的 Claude Code Hook 事件映射。

在运行链路中它承担“事件路由层”角色，而不是业务逻辑层：
- 业务意图来源于插件文档（学习式输出 + explanatory insight）：`plugins/learning-output-style/README.md:3-5,11-23,56-70`
- 路由配置在本目录 `hooks/hooks.json`：`plugins/learning-output-style/hooks/hooks.json:1-15`
- 真正执行逻辑在被调用脚本 `hooks-handlers/session-start.sh`：`plugins/learning-output-style/hooks-handlers/session-start.sh:1-15`

该目录被插件自动发现机制纳入加载范围：
- 标准结构中 `hooks/` 为可选组件目录：`plugins/README.md:49-60`
- 未在 `plugin.json` 自定义时，默认路径为 `./hooks/hooks.json`：`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:259-263`
- 自动发现流程第 5 步明确加载 `hooks/hooks.json`：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:343-348`

## 功能点目的
围绕 `hooks/hooks.json`，当前目录的功能目标可拆为 4 点：

1. 把插件能力挂到 `SessionStart` 事件
- 目录内唯一事件是 `SessionStart`：`plugins/learning-output-style/hooks/hooks.json:4`
- 含义是“每次会话开始时自动注入学习模式指令”。

2. 通过 command hook 将控制权转交给脚本
- 声明 `type: command`：`plugins/learning-output-style/hooks/hooks.json:8`
- 目标命令为 `${CLAUDE_PLUGIN_ROOT}/hooks-handlers/session-start.sh`：`plugins/learning-output-style/hooks/hooks.json:9`

3. 使用插件包装格式保持可读元信息
- 顶层 `description` + `hooks` 包装结构：`plugins/learning-output-style/hooks/hooks.json:2-3`
- 与 hook-development 文档定义的“plugin hooks.json 格式”一致：`plugins/plugin-dev/skills/hook-development/SKILL.md:62-80`

4. 提供可移植路径而非硬编码路径
- 借助 `${CLAUDE_PLUGIN_ROOT}` 避免绝对路径耦合：`plugins/learning-output-style/hooks/hooks.json:9`
- 与插件开发规范一致：`plugins/plugin-dev/skills/hook-development/SKILL.md:327-337`

## 具体技术实现（关键流程/数据结构/协议/命令）
### 1) 关键流程
1. Claude Code 启用插件后读取 `.claude-plugin/plugin.json`（本插件未覆写 hooks 路径）：`plugins/learning-output-style/.claude-plugin/plugin.json:1-9`
2. 按默认规则加载 `hooks/hooks.json`：`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:259-263`
3. 运行时遇到 `SessionStart` 事件，匹配本配置项：`plugins/learning-output-style/hooks/hooks.json:4-12`
4. 执行 command hook：`${CLAUDE_PLUGIN_ROOT}/hooks-handlers/session-start.sh`：`plugins/learning-output-style/hooks/hooks.json:9`
5. 脚本输出 `hookSpecificOutput.additionalContext`，会话上下文被注入学习模式指令：`plugins/learning-output-style/hooks-handlers/session-start.sh:8-10`

### 2) 关键数据结构
`hooks/hooks.json` 的有效结构（简化）为：

```json
{
  "description": "...",
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "${CLAUDE_PLUGIN_ROOT}/hooks-handlers/session-start.sh"
          }
        ]
      }
    ]
  }
}
```

字段语义：
- `description`：给人读的配置说明，不参与事件匹配。
- `hooks`（顶层）：插件格式的事件容器。
- `SessionStart`：事件键，表示会话启动触发。
- `hooks[]`（内层）：该事件要执行的 hook 列表。
- `type=command`：使用外部命令脚本执行。
- `command`：通过 `${CLAUDE_PLUGIN_ROOT}` 定位插件内脚本。

### 3) 协议与命令约定
Hook 协议要点（来自开发工具链定义）：
- 命令 Hook 从 stdin 接收 JSON 输入，包含 `hook_event_name/session_id/cwd` 等通用字段：`plugins/plugin-dev/skills/hook-development/SKILL.md:300-319`
- `SessionStart` 常用于加载上下文或环境初始化：`plugins/plugin-dev/skills/hook-development/SKILL.md:238-264`
- 命令 Hook 返回码语义：`0` 成功，`2` 阻断，其它为异常：`plugins/plugin-dev/skills/hook-development/SKILL.md:294-299`

本目录配置指向的脚本输出协议（被调用方）：
- 输出 JSON 包含 `hookSpecificOutput.hookEventName` 与 `hookSpecificOutput.additionalContext`：`plugins/learning-output-style/hooks-handlers/session-start.sh:8-10`

### 4) 实测命令与结果
为验证本目录配置可用性，本次执行了两类脚本测试：

1. Hook 执行冒烟测试
```bash
bash plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh --create-sample SessionStart > /tmp/learning-hooks-sessionstart-input.json
bash plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh \
  plugins/learning-output-style/hooks-handlers/session-start.sh \
  /tmp/learning-hooks-sessionstart-input.json
```
结果：退出码 `0`，输出可解析 JSON，包含 `hookSpecificOutput.additionalContext`。

2. Hook schema 校验测试
```bash
bash plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh \
  plugins/learning-output-style/hooks/hooks.json
```
结果：
- 先报告 `description/hooks` 为“Unknown event type”；
- 后续 `jq` 报错 `Cannot index string with number`；
- 实际退出码 `5`。

结论：该校验脚本按“事件在顶层”进行遍历，不兼容当前插件包装格式（`{"hooks": {...}}`）。

## 关键代码路径与文件引用
核心路径（调用链最短闭环）：
1. `plugins/learning-output-style/hooks/hooks.json:1-15`
2. `plugins/learning-output-style/hooks-handlers/session-start.sh:1-15`

调用方与配置来源：
- `plugins/learning-output-style/.claude-plugin/plugin.json:1-9`
- `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:259-263`
- `plugins/plugin-dev/skills/plugin-structure/SKILL.md:343-348`

协议/工具链与验证脚本：
- `plugins/plugin-dev/skills/hook-development/SKILL.md:62-80,238-264,294-337`
- `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh:25-31,83-93,170-191,205-252`
- `plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh:41-57,65-87`
- `plugins/plugin-dev/skills/hook-development/scripts/README.md:5-22,29-61,92-123`

文档与同类对照：
- `plugins/learning-output-style/README.md:24-27,72-82`
- `plugins/explanatory-output-style/hooks/hooks.json:1-15`
- `plugins/explanatory-output-style/hooks-handlers/session-start.sh:1-15`

## 依赖与外部交互
运行时依赖：
- Claude Code 插件系统与 Hook 事件系统（`SessionStart`）。
- Shell 执行环境（`bash`）。
- 环境变量展开（`${CLAUDE_PLUGIN_ROOT}`）。

开发与验证依赖：
- `jq`：JSON 校验/解析（`validate-hook-schema.sh`、`test-hook.sh`）。
- `timeout`：测试脚本执行超时控制（`test-hook.sh`）。

外部交互特征：
- 本目录配置本身不执行网络请求、不读写业务代码文件。
- 通过被调用脚本向 Claude 注入 `additionalContext`，影响后续会话行为。
- 与外部系统的主要交互是 Claude Hook 运行时协议（stdin JSON 输入、stdout/stderr + exit code 输出）。

## 风险、边界与改进建议
风险与边界：
1. 固定 token 成本
- `SessionStart` 每次会话都会注入较长 `additionalContext`，README 也明确提示 token 成本：`plugins/learning-output-style/README.md:7`。

2. 风格作用域是“会话级”
- 当前目录只配置 `SessionStart`，无细粒度 matcher 或场景分流；一旦启用即全会话生效。

3. 缺少目录内专用测试资产
- `plugins/learning-output-style/hooks` 本身没有独立测试文件，主要依赖 `plugin-dev` 通用脚本手工验证。

4. 校验脚本与插件包装格式错配
- `validate-hook-schema.sh` 对插件格式误报并异常退出（本次实测退出码 `5`）。

5. 与 explanatory 插件文案存在维护漂移风险
- 两插件均采用 `SessionStart + additionalContext` 模式，配置结构高度同构；后续演进若不同步可能导致行为分叉。

改进建议：
1. 在 `plugins/learning-output-style` 下新增轻量 smoke test 脚本，至少校验：
- `hooks/hooks.json` 可解析；
- `SessionStart` 路由存在；
- `session-start.sh` 输出包含 `hookSpecificOutput.additionalContext`。

2. 改造 `validate-hook-schema.sh` 兼容两种根结构：
- settings 直写格式（事件在顶层）；
- plugin 包装格式（事件在 `.hooks` 下）。

3. 为 `hooks/hooks.json` 增补简短注释性文档（README 附录即可），明确当前使用插件包装格式，避免误用工具。

4. 评估提供“短提示词”版本（精简 `additionalContext`），降低默认 token 占用。
