# plugins/ralph-wiggum/.claude-plugin 目录研究（DIR）

## 场景与职责

`plugins/ralph-wiggum/.claude-plugin` 是 `ralph-wiggum` 插件的 manifest 目录，当前仅包含 `plugin.json`（`plugins/ralph-wiggum/.claude-plugin/plugin.json:1-9`）。

该目录在插件生命周期中承担“被发现、被识别、被装配”的入口职责，而不直接执行业务逻辑：
1. 插件规范要求 manifest 必须位于 `.claude-plugin/plugin.json`，否则插件不会被识别（`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:7-10`）。
2. 组件发现阶段先读取 manifest，再扫描默认组件目录（`plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:11-16`）。
3. 仓库 marketplace 将 `ralph-wiggum` 的 `source` 路由到 `./plugins/ralph-wiggum`，运行时最终落到本目录读取 manifest（`.claude-plugin/marketplace.json:128-136`）。

因此本目录是 Ralph 插件能力链路（`/ralph-loop`、`/cancel-ralph`、Stop Hook）被加载的硬前置条件。

## 功能点目的

### 1) 声明插件身份与归属
`plugin.json` 声明 `name/version/description/author`（`plugins/ralph-wiggum/.claude-plugin/plugin.json:2-8`）：
- `name: ralph-wiggum`：插件唯一标识，用于识别与冲突检测。
- `version: 1.0.0`：语义化版本，配合 marketplace 展示。
- `description`：声明该插件实现“同 prompt 持续迭代”的 Ralph 技术。
- `author`：作者与联系方式。

### 2) 激活默认组件自动发现
manifest 未声明自定义 `commands/hooks/scripts` 路径，意味着依赖默认目录约定加载下游能力：
- 命令：`commands/ralph-loop.md`、`commands/cancel-ralph.md`、`commands/help.md`
- Hook 配置：`hooks/hooks.json`
- 执行脚本：`hooks/stop-hook.sh`、`scripts/setup-ralph-loop.sh`

对应文件分别见：
- `plugins/ralph-wiggum/commands/ralph-loop.md:1-18`
- `plugins/ralph-wiggum/commands/cancel-ralph.md:1-18`
- `plugins/ralph-wiggum/commands/help.md:1-126`
- `plugins/ralph-wiggum/hooks/hooks.json:1-15`
- `plugins/ralph-wiggum/hooks/stop-hook.sh:1-177`
- `plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:1-203`

### 3) 对齐 marketplace 与插件文档入口
- marketplace 中 `ralph-wiggum` 条目与本 manifest 在 `name/version/author` 上保持一致（`.claude-plugin/marketplace.json:128-136`）。
- 插件总览文档将其声明为“`/ralph-loop` + `/cancel-ralph` + Stop Hook”组合（`plugins/README.md:26`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 关键流程（调用方 -> 目标目录 -> 被调用方）

1. 上游调用方：marketplace 与插件发现机制
- marketplace 声明 `source: ./plugins/ralph-wiggum`（`.claude-plugin/marketplace.json:135`）。
- 发现阶段读取 `.claude-plugin/plugin.json`（`component-patterns.md:11-16`）。

2. 目标目录：manifest 入口
- `plugins/ralph-wiggum/.claude-plugin/plugin.json:1-9`

3. 下游被调用方：命令与 Hook 运行链
- `/ralph-loop` 仅允许调用 setup 脚本（`plugins/ralph-wiggum/commands/ralph-loop.md:2-5,12-14`）。
- `setup-ralph-loop.sh` 写入状态文件 `.claude/ralph-loop.local.md`，初始化 frontmatter 字段 `iteration/max_iterations/completion_promise/started_at` + prompt body（`plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:140-150`）。
- Stop 事件触发 `hooks/hooks.json`，执行 `hooks/stop-hook.sh`（`plugins/ralph-wiggum/hooks/hooks.json:3-10`）。
- `stop-hook.sh` 读取状态文件和 transcript，判断三种结论：
  - 允许退出：无状态文件、达最大迭代、命中 completion promise（`plugins/ralph-wiggum/hooks/stop-hook.sh:15-18,50-55,114-127`）
  - 阻断退出并继续：输出 `{"decision":"block","reason":"<原始prompt>"...}`（`plugins/ralph-wiggum/hooks/stop-hook.sh:165-174`）
  - 异常兜底：状态/转录异常时清理状态文件并放行退出（`plugins/ralph-wiggum/hooks/stop-hook.sh:28-37,39-48,60-67,71-77,98-105,138-149`）
- `/cancel-ralph` 通过读状态 + 删除状态文件强制停止循环（`plugins/ralph-wiggum/commands/cancel-ralph.md:11-18`）。

### B. 关键数据结构

1. manifest 结构（本目录核心）
```json
{
  "name": "ralph-wiggum",
  "version": "1.0.0",
  "description": "...",
  "author": {
    "name": "Daisy Hollman",
    "email": "daisy@anthropic.com"
  }
}
```
来源：`plugins/ralph-wiggum/.claude-plugin/plugin.json:1-9`

2. 循环状态文件结构（由下游脚本创建）
- 文件：`.claude/ralph-loop.local.md`
- frontmatter 字段：`active/iteration/max_iterations/completion_promise/started_at`
- body：原始 prompt
来源：`plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:140-150`

3. Stop Hook 返回协议
- 继续循环：`decision=block` + `reason=<原始prompt>` + `systemMessage`
来源：`plugins/ralph-wiggum/hooks/stop-hook.sh:167-174`

### C. 关键协议与命令

1. 插件 manifest 协议
- 路径与识别规则：`manifest-reference.md:7-10`
- `name` 格式与语义：`manifest-reference.md:15-36`

2. 命令协议
- `/ralph-loop` frontmatter 使用 `allowed-tools` 限定为 setup 脚本（`plugins/ralph-wiggum/commands/ralph-loop.md:4`）。
- `/cancel-ralph` 仅允许 `test/rm/read` 相关工具（`plugins/ralph-wiggum/commands/cancel-ralph.md:3`）。

3. Hook 协议
- `hooks/hooks.json` 将 Stop 事件绑定到 command hook（`plugins/ralph-wiggum/hooks/hooks.json:3-10`）。
- `stop-hook.sh` 从 stdin 读取 hook 输入 JSON，并解析 `transcript_path`（`plugins/ralph-wiggum/hooks/stop-hook.sh:9-10,57-58`）。

## 关键代码路径与文件引用

### 目标目录（直接研究对象）
- `plugins/ralph-wiggum/.claude-plugin/plugin.json:1-9`

### 调用方（上游）
- `.claude-plugin/marketplace.json:128-136`（插件注册与 source 路由）
- `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:7-10,15-36`（manifest 必需路径与字段约束）
- `plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:11-16`（组件发现生命周期）
- `plugins/README.md:26,49-60`（插件能力总览与结构约定）

### 被调用方（下游）
- `plugins/ralph-wiggum/commands/ralph-loop.md:1-18`
- `plugins/ralph-wiggum/commands/cancel-ralph.md:1-18`
- `plugins/ralph-wiggum/commands/help.md:1-126`
- `plugins/ralph-wiggum/hooks/hooks.json:1-15`
- `plugins/ralph-wiggum/hooks/stop-hook.sh:1-177`
- `plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:1-203`
- `plugins/ralph-wiggum/README.md:1-179`

### 配置/测试/脚本/文档上下文
- 配置：manifest（本目录）+ marketplace + hooks.json + command frontmatter。
- 测试：插件目录无 `tests/`、`spec/`、`__tests__/`；`find plugins/ralph-wiggum -maxdepth 4 -type d` 仅见 `.claude-plugin/commands/hooks/scripts` 结构。
- 脚本：两个 shell 脚本承担初始化与循环控制。
- 文档：`plugins/ralph-wiggum/README.md` 与 `commands/help.md` 解释机制与使用方法。

补充校验（本次实测）：
- `bash -n plugins/ralph-wiggum/hooks/stop-hook.sh` -> 通过。
- `bash -n plugins/ralph-wiggum/scripts/setup-ralph-loop.sh` -> 通过。
- `bash plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh plugins/ralph-wiggum/hooks/hooks.json` -> 报错 `Cannot index string with number`，显示该通用校验脚本与插件包装格式（顶层 `description/hooks`）存在兼容缺口。

## 依赖与外部交互

### 1) 运行时依赖
- Shell 与核心命令：`bash/sed/grep/awk/tail/mv/rm/date`
- JSON 与文本处理：`jq`（解析 transcript 与生成返回 JSON），`perl`（提取 `<promise>`）
- 关键证据：`plugins/ralph-wiggum/hooks/stop-hook.sh:21-26,57-58,90-95,119,167-174`

### 2) 文件系统交互
- 写入与读取：`.claude/ralph-loop.local.md`（setup 创建、hook 更新、cancel 删除）
  - 创建：`setup-ralph-loop.sh:140-150`
  - 读取/更新/删除：`stop-hook.sh:13,21-26,152-156` + `cancel-ralph.md:11-18`
- 读取 transcript：由 hook 输入提供 `transcript_path`（`stop-hook.sh:57-60`）

### 3) 平台协议交互
- hook stdin 输入 JSON（含 `transcript_path`）
- hook stdout 输出控制 JSON（block/allow）
- 事件绑定由 `hooks/hooks.json` 声明（`hooks/hooks.json:3-10`）

### 4) 文档与实现交互一致性
- README、help 命令都描述了 Ralph 迭代机制（`plugins/ralph-wiggum/README.md:13-34`，`plugins/ralph-wiggum/commands/help.md:22-30`），实现由 setup + stop hook 承接。

## 风险、边界与改进建议

### 风险
1. `commands/help.md` 中状态文件路径与实现不一致
- help 文档写的是 `.claude/.ralph-loop.local.md`（`plugins/ralph-wiggum/commands/help.md:49,69`），实际实现是 `.claude/ralph-loop.local.md`（`setup-ralph-loop.sh:140`，`stop-hook.sh:13`，`cancel-ralph.md:11`）。
- 影响：用户按帮助文本排障时会误判“无活动循环”。

2. setup 脚本提示“不能手动停止”与实际能力冲突
- setup 输出 `This loop cannot be stopped manually!`（`setup-ralph-loop.sh:166-167`），但插件明确提供 `/cancel-ralph` 手动停止（`cancel-ralph.md:1-18`，`README.md:63-70`）。
- 影响：用户对控制手段认知错位。

3. completion promise 的 YAML 转义不完整
- 当前仅用双引号包裹 `completion_promise`（`setup-ralph-loop.sh:134-136`），未转义内嵌引号、反斜杠等字符。
- 影响：特殊字符可能破坏 frontmatter 或与 `stop-hook.sh` 的 grep/sed 解析产生偏差。

4. Stop hook 对 frontmatter 字段缺失的容错存在边界
- `set -euo pipefail` + `grep '^iteration:'`/`grep '^max_iterations:'`（`stop-hook.sh:7,22-23`）在字段缺失时可能提前退出，未进入后续友好错误提示路径。
- 影响：损坏状态文件时行为不一定稳定可预期。

5. 通用 hook schema 校验脚本与本插件格式不兼容
- 实测 `validate-hook-schema.sh` 在本插件 `hooks/hooks.json` 上报 `Cannot index string with number`。
- 影响：仓库层通用校验无法直接覆盖此插件，增加回归盲区。

### 边界
1. 本目录仅负责 manifest 与发现入口，不承载循环算法本体。
2. 真实行为由 `commands/`、`hooks/`、`scripts/` 决定。
3. 该插件不含专用自动化测试目录，当前质量保障主要依赖脚本健壮性和人工回归。

### 改进建议
1. 修正文档路径一致性：统一 help/README/命令文档中的状态文件路径为 `.claude/ralph-loop.local.md`。
2. 修正 setup 提示文案：明确“可通过 `/cancel-ralph` 手动停止”，避免与命令能力冲突。
3. 强化 `completion_promise` 的 YAML 安全编码，避免特殊字符破坏状态文件。
4. 在 `stop-hook.sh` 增加字段缺失的显式判空与降级处理，避免 `set -e` 导致的非预期硬退出。
5. 为 `plugins/*/hooks/hooks.json` 增加兼容插件包装格式的统一校验脚本，补齐 CI 覆盖。
