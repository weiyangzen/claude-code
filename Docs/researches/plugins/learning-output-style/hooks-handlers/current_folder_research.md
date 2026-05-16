# DIR `plugins/learning-output-style/hooks-handlers` 研究文档

## 场景与职责
`plugins/learning-output-style/hooks-handlers` 是 `learning-output-style` 插件的执行层目录，当前仅包含一个可执行脚本：`session-start.sh`。  
它不负责事件注册，而是负责在事件已匹配后产出标准 Hook 输出 JSON，把“学习式输出 + explanatory insight”指令注入 Claude 会话上下文。

该目录在调用链中的位置：
1. 插件清单被发现：`plugins/learning-output-style/.claude-plugin/plugin.json:1`
2. Hook 路由被加载：`plugins/learning-output-style/hooks/hooks.json:3`
3. `SessionStart` 触发时调用本目录脚本：`plugins/learning-output-style/hooks/hooks.json:9`
4. 脚本输出 `hookSpecificOutput.additionalContext`：`plugins/learning-output-style/hooks-handlers/session-start.sh:8`

可理解为：`hooks/` 负责“挂接事件”，`hooks-handlers/` 负责“执行并返回协议数据”。

## 功能点目的
围绕 `session-start.sh`，本目录的功能目标可以拆成 5 点：

1. 在会话开始时统一注入学习模式行为约束  
- 指导模型主动让用户贡献 5-10 行关键代码，而不是全自动实现：`plugins/learning-output-style/hooks-handlers/session-start.sh:10`

2. 明确“何时请求用户写代码 / 何时不请求”  
- 请求场景：业务逻辑、错误处理、算法/数据结构、架构选择等。  
- 不请求场景：样板代码、显而易见实现、配置、简单 CRUD。  
均由 `additionalContext` 长文本定义：`plugins/learning-output-style/hooks-handlers/session-start.sh:10`

3. 复用 explanatory 输出习惯，加入教育性洞察模板  
- 要求在编码前后给出 `★ Insight` 结构化洞察块：`plugins/learning-output-style/hooks-handlers/session-start.sh:10`
- 与对照插件 `explanatory-output-style` 共享同类模式：`plugins/explanatory-output-style/hooks-handlers/session-start.sh:10`

4. 作为“已下线 output style 的插件化迁移载体”  
- 插件 README 明确该插件用于迁移 Learning/Explanatory 行为：`plugins/learning-output-style/README.md:3`

5. 通过纯脚本输出保持实现简单、可移植  
- 仅 `bash + heredoc + stdout JSON`，不依赖 Python/Node：`plugins/learning-output-style/hooks-handlers/session-start.sh:1`

## 具体技术实现（关键流程/数据结构/协议/命令）
### 1) 关键流程（调用方 -> 目标目录 -> 被调用方）
1. Claude Code 在插件启用时读取 `.claude-plugin/plugin.json` 并自动发现 hooks：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:343`
2. 默认 hooks 路径为 `./hooks/hooks.json`（本插件未覆写）：`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:262`
3. 运行时发生 `SessionStart` 事件，匹配到 command hook：`plugins/learning-output-style/hooks/hooks.json:4`
4. 命令 `${CLAUDE_PLUGIN_ROOT}/hooks-handlers/session-start.sh` 被执行：`plugins/learning-output-style/hooks/hooks.json:9`
5. 脚本输出 JSON：
   - `hookSpecificOutput.hookEventName = "SessionStart"`
   - `hookSpecificOutput.additionalContext = <学习+讲解指令>`
   见：`plugins/learning-output-style/hooks-handlers/session-start.sh:8`
6. Hook runtime 将 `additionalContext` 追加进模型上下文，影响后续整个会话回答风格。

### 2) 关键数据结构
本目录核心数据结构是脚本输出 JSON（简化）：

```json
{
  "hookSpecificOutput": {
    "hookEventName": "SessionStart",
    "additionalContext": "..."
  }
}
```

字段职责：
- `hookSpecificOutput`：hook 专用返回载荷容器。
- `hookEventName`：声明该返回属于 `SessionStart`。
- `additionalContext`：核心行为载体，直接改变后续对话策略。

### 3) 协议要点（输入/输出/退出码）
1. 命令 hook 标准输入是 stdin JSON（包含 `hook_event_name/session_id/cwd` 等通用字段）：`plugins/plugin-dev/skills/hook-development/SKILL.md:302`
2. 本目录脚本没有消费 stdin，属于“纯静态上下文注入器”实现。
3. 命令 hook 退出码约定：`0` 成功，`2` 阻断，其他为异常：`plugins/plugin-dev/skills/hook-development/SKILL.md:296`
4. `session-start.sh` 固定 `exit 0`：`plugins/learning-output-style/hooks-handlers/session-start.sh:15`

### 4) 关键命令与实测
本次针对目标目录做了可复现实测：

1. 生成 `SessionStart` 样例输入
```bash
bash plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh --create-sample SessionStart > /tmp/learning_sessionstart_input.json
jq -r '.hook_event_name' /tmp/learning_sessionstart_input.json
```
结果：`SessionStart`。

2. 执行目标脚本（通过官方测试辅助脚本）
```bash
bash plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh \
  plugins/learning-output-style/hooks-handlers/session-start.sh \
  /tmp/learning_sessionstart_input.json
```
结果：`Exit Code: 0`，输出 JSON 可解析，含 `hookSpecificOutput.additionalContext`。

3. 对目标脚本执行 linter
```bash
bash plugins/plugin-dev/skills/hook-development/scripts/hook-linter.sh \
  plugins/learning-output-style/hooks-handlers/session-start.sh
```
结果：通过但有 2 条 warning（缺少 `set -euo pipefail`、stderr 提示规范）。

4. 校验关联配置 `hooks/hooks.json`
```bash
bash plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh \
  plugins/learning-output-style/hooks/hooks.json
```
结果：脚本对插件包装格式 `{"description": "...", "hooks": {...}}` 不兼容，`jq` 报错并退出码 `5`（并非目标目录脚本本身故障）。

## 关键代码路径与文件引用
### 目标目录（直接对象）
- `plugins/learning-output-style/hooks-handlers/session-start.sh:1`

### 调用方（上游）
- `plugins/learning-output-style/hooks/hooks.json:4`
- `plugins/learning-output-style/hooks/hooks.json:9`
- `plugins/learning-output-style/.claude-plugin/plugin.json:1`
- `.claude-plugin/marketplace.json:95`

### 被调用方与同类参照（下游/对照）
- Claude Hook runtime 输出消费协议：`plugins/plugin-dev/skills/hook-development/SKILL.md:278`
- 对照脚本（explanatory）：`plugins/explanatory-output-style/hooks-handlers/session-start.sh:10`
- 插件说明文档：`plugins/learning-output-style/README.md:24`
- 仓库插件总览定位：`plugins/README.md:23`

### 配置、测试、脚本、文档上下文
- 配置：`plugins/learning-output-style/hooks/hooks.json:1`
- 测试脚本：`plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh:29`
- Lint 脚本：`plugins/plugin-dev/skills/hook-development/scripts/hook-linter.sh:1`
- Schema 校验脚本：`plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh:41`
- Hook 开发规范：`plugins/plugin-dev/skills/hook-development/SKILL.md:62`

## 依赖与外部交互
### 运行时依赖
1. Claude Code Hook 事件系统（`SessionStart`）。
2. Shell 解释器（`#!/usr/bin/env bash`）。
3. 路径变量展开（`${CLAUDE_PLUGIN_ROOT}` 在上游配置里提供）。

### 开发与验证依赖
1. `jq`（测试脚本/校验脚本解析 JSON）。
2. `timeout`（`test-hook.sh` 超时控制）。
3. `bash`（脚本执行与工具链）。

### 外部交互特征
1. 本目录脚本不访问网络、不读写项目业务文件。
2. 主要副作用是“增加会话系统上下文文本”，从而改变模型行为和 token 消耗。
3. 与用户交互是间接的：通过后续回答风格体现，而非直接输出终端提示。

## 风险、边界与改进建议
### 风险
1. 指令体积较大，固定增加会话 token 成本  
- README 已给出成本提醒：`plugins/learning-output-style/README.md:7`

2. `additionalContext` 为单一长字符串，回归风险集中  
- 任意修改可能同时影响“请求用户贡献策略 + explanatory 样式”，缺少自动化断言。

3. 维护漂移风险  
- 与 `explanatory-output-style` 存在部分同构文案，长期可能出现不一致行为：`plugins/explanatory-output-style/hooks-handlers/session-start.sh:10`

4. 质量门禁工具错配风险  
- 当前通用 `validate-hook-schema.sh` 不能直接验证本插件 hooks 包装格式，容易误判。

### 边界
1. 本目录只负责静态上下文注入，不做条件判断、用户状态判断、项目类型分流。
2. 不消费 hook 输入 JSON，因此无法基于 `agent_type`、`cwd`、`permission_mode` 动态调整策略。
3. 不提供独立测试目录，验证依赖外部通用脚本。

### 改进建议
1. 增加目录级 smoke test  
- 断言输出可被 `jq` 解析、`hookEventName == "SessionStart"`、`additionalContext` 非空。

2. 增加“短指令版”上下文模板  
- 在保持学习目标的同时降低 token 负担，供低成本场景使用。

3. 结构化维护文案片段  
- 将 explanatory 共享段落抽成可复用模板，减少双插件文案漂移。

4. 提升脚本防御性  
- 加 `set -euo pipefail`，并在异常路径输出 stderr（虽然当前脚本简单，仍可提高一致性）。

5. 修复 `validate-hook-schema.sh` 对插件 wrapper 的兼容  
- 先下钻 `.hooks` 再做事件校验，可覆盖本插件与 explanatory 插件。
