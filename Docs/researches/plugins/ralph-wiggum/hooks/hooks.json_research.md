# FILE `plugins/ralph-wiggum/hooks/hooks.json` 研究文档

## 场景与职责

`plugins/ralph-wiggum/hooks/hooks.json` 是 `ralph-wiggum` 插件的 Hook 路由入口，职责是把 Claude Code 的 `Stop` 生命周期事件绑定到本插件的循环控制脚本 `hooks/stop-hook.sh`。

它在链路中的位置是“声明层”，不承载业务逻辑：

1. 上游调用方
- Claude Code 插件运行时在加载插件后读取该配置，并在主代理准备结束时触发 `Stop` 事件。
- 插件整体功能在目录总览中定义为“Stop hook 拦截退出并继续迭代”（`plugins/README.md:26`）。

2. 下游被调用方
- 命令型 Hook 脚本：`plugins/ralph-wiggum/hooks/stop-hook.sh:1-177`。
- 间接依赖状态文件 `.claude/ralph-loop.local.md`（由 `plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:140-150` 创建）。

3. 同目录边界
- `hooks.json` 只做事件映射与命令声明。
- “是否放行停止/是否阻断并续跑”的判定全部在 `stop-hook.sh` 内执行。

## 功能点目的

1. 把 Ralph 循环接入 Stop 事件
- 通过 `hooks.Stop` 注册命令，使每次代理尝试停止时都能进入 Ralph 判定逻辑。

2. 保障插件路径可移植
- 使用 `${CLAUDE_PLUGIN_ROOT}/hooks/stop-hook.sh`，避免硬编码绝对路径导致安装目录变化后失效（`plugins/ralph-wiggum/hooks/hooks.json:9`）。

3. 维持最小配置面
- 当前仅注册单一事件 `Stop`，符合插件设计意图：Ralph 不做工具级审计，不在 `PreToolUse/PostToolUse` 插手，只在“是否结束会话”这一关键点仲裁。

4. 与插件 wrapper 格式一致
- 顶层采用 `{ "description": "...", "hooks": { ... } }`，符合仓库中插件 Hook 约定格式（`plugins/plugin-dev/skills/hook-development/SKILL.md:62-80`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 配置结构

目标文件结构如下（等价简化）：

```json
{
  "description": "Ralph Wiggum plugin stop hook for self-referential loops",
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "${CLAUDE_PLUGIN_ROOT}/hooks/stop-hook.sh"
          }
        ]
      }
    ]
  }
}
```

关键点：
- 事件键在 `.hooks.Stop` 下，不在顶层。
- `type=command` 表示由 shell 脚本处理 stdin JSON。
- 未显式配置 `matcher` 与 `timeout`，由运行时默认行为兜底。

### 2) 关键流程

1. `/ralph-loop` 命令执行 `setup-ralph-loop.sh` 写入 `.claude/ralph-loop.local.md`（`plugins/ralph-wiggum/commands/ralph-loop.md:12-14`，`plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:140-150`）。
2. 代理完成一轮后尝试停止，触发 `Stop` 事件。
3. 运行时读取 `hooks.json`，命中 `Stop -> command` 路由。
4. 执行 `${CLAUDE_PLUGIN_ROOT}/hooks/stop-hook.sh`。
5. 脚本返回：
- 放行：仅 `exit 0`（不输出阻断决策）。
- 继续循环：输出 `{"decision":"block","reason":"<原prompt>","systemMessage":"<迭代提示>"}`（`plugins/ralph-wiggum/hooks/stop-hook.sh:167-174`）。

### 3) 协议与命令

1. 输入协议
- command hook 从 stdin 接收 JSON，`Stop` 场景可用字段包括 `transcript_path`、`reason`（参考 `plugins/plugin-dev/skills/hook-development/SKILL.md:300-319`，`plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh:59-69`）。

2. 输出协议
- Stop 阻断输出采用 `decision=block` 协议（`plugins/plugin-dev/skills/hook-development/SKILL.md:202-209`）。
- Ralph 利用 `reason` 字段承载“下一轮继续使用的原始 prompt”。

### 4) 本次验证结果（实测）

1. JSON 语法
- `jq empty plugins/ralph-wiggum/hooks/hooks.json`：通过。

2. 仓库自带校验脚本
- `bash plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh plugins/ralph-wiggum/hooks/hooks.json`
- 结果：警告 `Unknown event type: description/hooks`，随后 `jq: Cannot index string with number`，退出码 `5`。
- 原因：该校验脚本按“顶层键即事件”遍历（`validate-hook-schema.sh:43-70`），与本文件采用的 plugin wrapper 格式不兼容。

3. 运行链路可用性（通过 stop-hook 实测）
- 在临时目录中以 Stop 输入触发脚本，配置路由对应命令可正确返回 block JSON 或放行（详见 `stop-hook.sh_research.md`）。

## 关键代码路径与文件引用

核心配置与实现：
- `plugins/ralph-wiggum/hooks/hooks.json:1-15`
- `plugins/ralph-wiggum/hooks/stop-hook.sh:1-177`

上游调用入口：
- `plugins/ralph-wiggum/commands/ralph-loop.md:1-18`
- `plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:130-203`
- `plugins/README.md:26`

相关命令与状态控制：
- `plugins/ralph-wiggum/commands/cancel-ralph.md:3,11-18`
- `plugins/ralph-wiggum/README.md:13-27,50-70`

Hook 规范与验证脚本（上下文依赖）：
- `plugins/plugin-dev/skills/hook-development/SKILL.md:62-80,181-209,300-319`
- `plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh:41-75`
- `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh:59-69`

## 依赖与外部交互

1. 运行时依赖
- Claude Code Hook Runtime（事件触发与命令执行）。
- 环境变量 `${CLAUDE_PLUGIN_ROOT}`（路径解析）。

2. 间接脚本依赖
- `stop-hook.sh` 依赖 `bash`, `jq`, `sed`, `awk`, `grep`, `tail`, `perl`, `mv`, `rm`。

3. 文件系统交互（由下游脚本执行）
- 读写 `.claude/ralph-loop.local.md`。
- 读取 stdin 提供的 `transcript_path` 文件。

4. 网络交互
- 本文件无网络调用；纯本地配置声明。

## 风险、边界与改进建议

1. 风险：校验脚本错配
- 当前仓库的 `validate-hook-schema.sh` 对 plugin wrapper 结构不兼容，会误报并异常退出。
- 建议：先检测是否存在顶层 `.hooks`，存在时下钻 `.hooks` 再校验。

2. 风险：未显式 `timeout`
- 若下游脚本处理 transcript 变慢，可能受默认超时影响但不可见。
- 建议：在命令条目上显式设置合理 `timeout`（例如 `10-30s`），便于维护预期。

3. 边界：只治理 Stop 阶段
- 该文件不处理 `PreToolUse/PostToolUse`，也不做输入安全检查，仅控制“会话是否结束”。

4. 改进：增强文档一致性
- `commands/help.md` 内仍写 `.claude/.ralph-loop.local.md`（多了一个点，`help.md:49,69`），与真实路径 `.claude/ralph-loop.local.md` 不一致。
- 建议统一文档路径，避免用户按帮助文案排障时找错文件。
