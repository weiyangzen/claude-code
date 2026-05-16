# FILE `plugins/explanatory-output-style/hooks-handlers/session-start.sh` 研究文档

## 场景与职责

`plugins/explanatory-output-style/hooks-handlers/session-start.sh` 是 `explanatory-output-style` 插件在运行时的唯一执行脚本，定位为 **SessionStart 阶段的上下文注入器**，不是业务计算脚本。

该文件所处场景：
- 插件目标是复刻已废弃的 Explanatory output style（`plugins/explanatory-output-style/README.md:3-4`）。
- 启用后每次会话启动自动生效，注入教育性输出指令（`plugins/explanatory-output-style/README.md:20-23,32-33`）。
- 插件在总览中被定义为 SessionStart Hook 型插件（`plugins/README.md:19`）。

该文件承担职责：
- 按 Hook 协议输出 JSON（stdout），声明事件为 `SessionStart`，并提供 `hookSpecificOutput.additionalContext`（`plugins/explanatory-output-style/hooks-handlers/session-start.sh:8-11`）。
- 用一段固定长文本指令约束模型行为（解释性、教学化、代码前后 insight 格式）。
- 以 `exit 0` 返回成功，确保会话初始化流程不中断（`plugins/explanatory-output-style/hooks-handlers/session-start.sh:15`）。

## 功能点目的

该脚本的功能点可以拆成 5 个目标：

1. 兼容“已废弃 output style”的迁移路径
- README 明确说明本插件用于替代旧配置 `"outputStyle": "Explanatory"`（`plugins/explanatory-output-style/README.md:42-54`）。
- CHANGELOG 明确 `/output-style` 已弃用并固定在 session start 生效（`CHANGELOG.md:206`）。

2. 在会话启动时一次性注入全局行为偏好
- Hook 配置把 `SessionStart` 路由到本脚本：`${CLAUDE_PLUGIN_ROOT}/hooks-handlers/session-start.sh`（`plugins/explanatory-output-style/hooks/hooks.json:4-10`）。
- 避免每次工具调用重复注入，符合“会话级风格设定”的语义。

3. 强制“任务完成 + 教学解释”的平衡
- 指令要求保持任务导向，但给出解释性洞察，防止只给结果不讲原因（`plugins/explanatory-output-style/hooks-handlers/session-start.sh:10`）。

4. 规范 insight 的展示格式
- 通过固定的 `★ Insight` 模板，提高输出一致性与可识别性（`plugins/explanatory-output-style/README.md:24-28` 与脚本内同款模板）。

5. 保持实现最小化和可移植
- 单文件、纯 Bash heredoc，无外部依赖命令链。
- 不读写工程文件，不访问网络，不依赖项目语言生态。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 关键流程

运行链路如下：

1. Claude Code 插件系统加载插件元数据（`plugins/explanatory-output-style/.claude-plugin/plugin.json:1-9`）。
2. Hook 配置文件声明 `SessionStart` 事件对应 command hook（`plugins/explanatory-output-style/hooks/hooks.json:3-13`）。
3. 会话启动触发 `SessionStart`，执行 `${CLAUDE_PLUGIN_ROOT}/hooks-handlers/session-start.sh`（`plugins/explanatory-output-style/hooks/hooks.json:9`）。
4. 脚本通过 `cat << 'EOF'` 输出固定 JSON 到 stdout（`plugins/explanatory-output-style/hooks-handlers/session-start.sh:6-13`）。
5. 运行时消费 `hookSpecificOutput.additionalContext` 并并入会话上下文。
6. `exit 0` 结束（`plugins/explanatory-output-style/hooks-handlers/session-start.sh:15`）。

### 2) 数据结构与协议

脚本输出结构：

```json
{
  "hookSpecificOutput": {
    "hookEventName": "SessionStart",
    "additionalContext": "..."
  }
}
```

协议要点：
- `hookEventName` 与触发事件一致（`SessionStart`）。
- `additionalContext` 为长字符串，包含行为约束、格式要求、时机要求。
- 对 SessionStart，此脚本不返回 `permissionDecision/decision`，而是上下文增强型输出。

与 Hook 开发规范的对应关系：
- SessionStart 用于“加载上下文/设置环境”（`plugins/plugin-dev/skills/hook-development/SKILL.md:238-264`）。
- 插件 hooks 文件采用 wrapper 格式 `{ "description": ..., "hooks": { ... } }`（`plugins/plugin-dev/skills/hook-development/SKILL.md:64-80`）。
- 退出码 `0` 表示成功（`plugins/plugin-dev/skills/hook-development/SKILL.md:294-299`）。

### 3) 关键命令与验证结果（本次实测）

1. 直接检查输出协议：
```bash
plugins/explanatory-output-style/hooks-handlers/session-start.sh | jq -r '.hookSpecificOutput.hookEventName'
```
结果：`SessionStart`。

2. 用官方测试脚本生成 SessionStart 样例输入：
```bash
bash plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh --create-sample SessionStart > /tmp/explanatory-sessionstart-input.json
```
结果：生成 JSON，`hook_event_name` 为 `SessionStart`。

3. 执行 hook 测试：
```bash
bash plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh plugins/explanatory-output-style/hooks-handlers/session-start.sh /tmp/explanatory-sessionstart-input.json
```
结果：Exit Code 0，输出 JSON 可被 `jq` 解析。

4. 执行脚本级 lint：
```bash
bash plugins/plugin-dev/skills/hook-development/scripts/hook-linter.sh plugins/explanatory-output-style/hooks-handlers/session-start.sh
```
结果：通过，但有 1 条 warning（缺少 `set -euo pipefail`）。

5. 执行 hooks 配置 schema 校验：
```bash
bash plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh plugins/explanatory-output-style/hooks/hooks.json
```
结果：脚本对 plugin wrapper 格式处理不完整，遍历顶层键时将 `description/hooks` 当事件处理，触发 jq 报错并以非零退出（`plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh:34-71`）。

## 关键代码路径与文件引用

主路径（调用方 -> 被调用方）：
- `plugins/explanatory-output-style/hooks/hooks.json:4-10`
- `plugins/explanatory-output-style/hooks-handlers/session-start.sh:1-15`

插件语义与迁移说明：
- `plugins/explanatory-output-style/README.md:3-4`
- `plugins/explanatory-output-style/README.md:20-23`
- `plugins/explanatory-output-style/README.md:42-56`

插件身份与分发元数据：
- `plugins/explanatory-output-style/.claude-plugin/plugin.json:2-4`

横向对照（同构实现）：
- `plugins/learning-output-style/hooks-handlers/session-start.sh:1-15`

Hook 规范与测试工具：
- `plugins/plugin-dev/skills/hook-development/SKILL.md:64-80`（plugin hooks.json wrapper）
- `plugins/plugin-dev/skills/hook-development/SKILL.md:238-264`（SessionStart 用途）
- `plugins/plugin-dev/skills/hook-development/SKILL.md:294-299`（退出码）
- `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh:23-92,175-240`
- `plugins/plugin-dev/skills/hook-development/scripts/hook-linter.sh:45-59`
- `plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh:34-71`

运行策略/演进背景：
- `CHANGELOG.md:192`（修复 resume 时 SessionStart 重复触发）
- `CHANGELOG.md:206`（output-style 固定在 session start）
- `CHANGELOG.md:597`（延后 SessionStart 以改善启动性能）

## 依赖与外部交互

直接依赖：
- shell 解释器：`#!/usr/bin/env bash`（`plugins/explanatory-output-style/hooks-handlers/session-start.sh:1`）。
- Claude Code Hook runtime（提供 SessionStart 生命周期并消费 stdout JSON）。

间接依赖：
- `hooks/hooks.json` 的路由声明（否则脚本不会被调用）。
- 插件安装/启用机制（README 所述）。

外部交互特征：
- 不读取 stdin。
- 不读取/写入 `$CLAUDE_ENV_FILE`。
- 不访问网络、不调用外部 API。
- 不触发文件系统写操作。

可观测交互面：
- 唯一输出通道：stdout JSON。
- 唯一状态信号：退出码（0）。

## 风险、边界与改进建议

### 风险与边界

1. Token 成本与短任务噪音
- 每次 SessionStart 都注入长文本，README 已明确 token 成本风险（`plugins/explanatory-output-style/README.md:6-7`）。
- 对极短任务/纯查询任务，教育性模板可能增加上下文和输出负担。

2. 全会话强制生效，缺少动态分流
- 当前仅 SessionStart 全局注入，`hooks.json` 没有场景条件（`plugins/explanatory-output-style/hooks/hooks.json:4-12`）。
- 无法按任务类型、仓库状态、用户偏好动态开关。

3. 文本硬编码与可维护性
- `additionalContext` 为单行超长字符串（`session-start.sh:10`），可读性和 diff 友好性较差。
- 与 `learning-output-style` 存在同构/部分重叠文案，长期可能产生行为漂移。

4. 工具链一致性问题
- `hook-linter.sh` 按通用最佳实践提示缺少 `set -euo pipefail`。
- `validate-hook-schema.sh` 当前不兼容插件 wrapper 格式，导致“官方样式配置 + 官方校验器”存在错配。

### 改进建议

1. 增加最小回归测试（建议优先）
- 新增脚本化 smoke test，断言：
  - 输出是合法 JSON
  - `hookSpecificOutput.hookEventName == "SessionStart"`
  - `hookSpecificOutput.additionalContext` 非空
- 可复用 `test-hook.sh` 作为执行器。

2. 优化配置校验脚本
- 在 `validate-hook-schema.sh` 中先判断并下钻 `.hooks`（若存在）再做事件遍历，兼容 plugin wrapper。
- 避免将 `description` 等元字段误判为事件。

3. 结构化维护 `additionalContext`
- 保持语义不变前提下，把长文本拆为多段变量或外部模板文件，再组合输出 JSON。
- 好处：审阅可读性更高，便于与 `learning-output-style` 同步维护。

4. 根据场景设计可选降噪策略
- 例如提供轻量版 explanatory 插件，或在文案中允许“超短任务减少 insight 频次”。
- 维持教学价值同时降低冗余输出。

5. 安全基线细化（可选）
- 该脚本虽不处理输入，但可考虑补充 `set -euo pipefail` 以统一 hook 脚本风格。

