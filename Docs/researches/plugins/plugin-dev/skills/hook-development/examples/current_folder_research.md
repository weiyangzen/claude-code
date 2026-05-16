# DIR 研究：plugins/plugin-dev/skills/hook-development/examples

## 场景与职责

`plugins/plugin-dev/skills/hook-development/examples` 是 `hook-development` 技能的“可运行样例层”，职责不是提供生产级 Hook 框架，而是提供最小可执行脚本，帮助插件作者快速理解 Claude Code Hook 的命令式实现模式。

在仓库内的上下文定位如下：

- 上游调用方（文档/流程级）：
  - `plugins/plugin-dev/skills/hook-development/SKILL.md:264`（引用 `examples/load-context.sh`）
  - `plugins/plugin-dev/skills/hook-development/SKILL.md:467`（引用 `validate-write.sh` / `validate-bash.sh`）
  - `plugins/plugin-dev/skills/hook-development/SKILL.md:677-681`（将本目录标注为 working examples）
  - `plugins/plugin-dev/README.md:286-290`（统计 Hook Development 的 3 个完整示例）
  - `plugins/plugin-dev/commands/create-plugin.md:201-209`（create-plugin 流程中要求创建并测试 Hook 脚本）
- 同级协作脚本（测试/校验）：
  - `plugins/plugin-dev/skills/hook-development/scripts/README.md:83-90`（直接示例 lint 本目录脚本）
  - `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh:17-18`（示例命令直接点名 `validate-bash.sh` / `validate-write.sh`）
- 下游被调用方（运行时依赖）：
  - Claude Code Hook 运行时 stdin JSON（`tool_input`、`hook_event_name` 等）
  - 环境变量 `CLAUDE_PROJECT_DIR` / `CLAUDE_ENV_FILE`
  - Shell 工具链（`bash`、`jq`）

结论：该目录承担“Hook 协议样板 + 安全基线示例 + SessionStart 上下文注入示例”的教学与模板职责。

## 功能点目的

本目录包含 3 个脚本，对应 3 类典型 Hook 场景：

1. `validate-write.sh`（PreToolUse 写入校验）
- 目的：在 `Write/Edit` 等写入场景中做路径与敏感目标的快速风险分级。
- 结果：将路径分为 `deny`（阻断）、`ask`（请求确认）与默认放行。

2. `validate-bash.sh`（PreToolUse 命令校验）
- 目的：对 Bash 命令做启发式风险判断，优先快速放行白名单命令，再拦截高危命令。
- 结果：覆盖 destructive command、磁盘破坏、提权命令三类风险。

3. `load-context.sh`（SessionStart 上下文加载）
- 目的：在会话启动阶段识别项目技术栈，并写入 `CLAUDE_ENV_FILE` 供会话后续步骤读取。
- 结果：对 Node/Rust/Go/Python/Java 生态进行基础识别，并设置 `PROJECT_TYPE` / `BUILD_SYSTEM` 等变量。

目录级目标：以最小实现演示 command hook 能力边界，并与 `references/migration.md` 中“迁移到 prompt hook”的策略形成对照（command hook 保留在确定性快速检查场景）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 共享协议与输入输出约定

- 脚本风格：3 个示例均使用 `#!/bin/bash` 与 `set -euo pipefail`（`examples/*.sh`）。
- Hook 输入：
  - `validate-write.sh` 读取 `.tool_input.file_path`（`validate-write.sh:11`）
  - `validate-bash.sh` 读取 `.tool_input.command`（`validate-bash.sh:11`）
  - `load-context.sh` 不解析 stdin，主要依赖环境变量（`load-context.sh:8,15,19...`）
- Hook 决策输出：
  - 通过 stderr 输出 JSON + `exit 2` 表示拦截/确认分支（`validate-write.sh:21-35`，`validate-bash.sh:26-39`）
  - 通过 `exit 0` 表示放行。

### 2) `validate-write.sh` 关键流程

`examples/validate-write.sh`：

1. `input=$(cat)` 读取 hook 输入（`8`）。
2. `jq` 解析 `file_path`，为空则直接 `{"continue": true}` 并放行（`11-17`）。
3. 规则链：
- 路径穿越（`..`）→ `permissionDecision=deny`（`20-23`）
- 系统目录（`/etc` `/sys` `/usr`）→ `deny`（`26-29`）
- 敏感目标（`.env`/`secret`/`credentials`）→ `ask`（`32-35`）
4. 无命中则 `exit 0`（`37-38`）。

实现特点：逻辑简单、成本低，但属于字符串模式匹配，覆盖深度有限。

### 3) `validate-bash.sh` 关键流程

`examples/validate-bash.sh`：

1. 读取 stdin 并解析 command（`8-11`）。
2. 空命令时返回 `{"continue": true}`（`14-17`）。
3. 快速白名单：`ls|pwd|echo|date|whoami` 直接放行（`20-22`）。
4. 风险分支：
- `rm -rf` / `rm -fr` → `deny`（`25-28`）
- `dd if=` / `mkfs` / `> /dev/` → `deny`（`31-34`）
- `sudo` / `su` → `ask`（`37-40`）
5. 默认放行（`42-43`）。

实现特点：体现“先快后严”的判定顺序，适合低延迟场景，但对变体绕过较敏感。

### 4) `load-context.sh` 关键流程

`examples/load-context.sh`：

1. 切换到 `CLAUDE_PROJECT_DIR`（`8`）。
2. 按项目文件探测技术栈并写入 `CLAUDE_ENV_FILE`（`13-47`）：
- `package.json` → `PROJECT_TYPE=nodejs`（可叠加 `USES_TYPESCRIPT=true`）
- `Cargo.toml` / `go.mod` / `pyproject.toml|setup.py`
- `pom.xml` / `build.gradle|build.gradle.kts`（含 `BUILD_SYSTEM`）
3. 附加 CI 探测，命中则写 `HAS_CI=true`（`49-52`）。
4. 结束输出日志并 `exit 0`（`54-55`）。

实现特点：通过 `CLAUDE_ENV_FILE` 把一次性探测结果转为会话可复用变量，是 SessionStart hook 的关键模式。

### 5) 关键数据结构与命令协议

- 输入 JSON 关键字段：
  - `tool_input.file_path`（写入类）
  - `tool_input.command`（Bash 类）
- 决策输出结构（示例脚本使用）：
  - `{"hookSpecificOutput": {"permissionDecision": "deny|ask"}, "systemMessage": "..."}`
- 常用验证命令（实测可运行）：

```bash
cd plugins/plugin-dev/skills/hook-development/scripts
./hook-linter.sh ../examples/*.sh
./test-hook.sh --create-sample PreToolUse > /tmp/hook_pretool.json
./test-hook.sh ../examples/validate-write.sh /tmp/hook_pretool.json
./test-hook.sh ../examples/validate-bash.sh /tmp/hook_pretool.json
./test-hook.sh --create-sample SessionStart > /tmp/hook_sessionstart.json
./test-hook.sh ../examples/load-context.sh /tmp/hook_sessionstart.json
```

### 6) 实测观察（本次研究执行）

- `hook-linter.sh` 对 3 个示例均给出 warning，但整体判定通过（exit 0）。
- `validate-write.sh` 在 PreToolUse 样例输入下返回 exit 0（放行，无输出）。
- `validate-bash.sh` 在同一输入下返回 `{"continue": true}`（因为输入是 Write 场景、无 command）。
- `load-context.sh` 在临时 Node+TS 项目中可写出：
  - `export PROJECT_TYPE=nodejs`
  - `export USES_TYPESCRIPT=true`

## 关键代码路径与文件引用

核心对象（本目录）：

- `plugins/plugin-dev/skills/hook-development/examples/validate-write.sh:1-38`
- `plugins/plugin-dev/skills/hook-development/examples/validate-bash.sh:1-43`
- `plugins/plugin-dev/skills/hook-development/examples/load-context.sh:1-55`

关键上下文（调用方/配套）：

- `plugins/plugin-dev/skills/hook-development/SKILL.md:264,467,677-681`
- `plugins/plugin-dev/skills/hook-development/scripts/README.md:83-90`
- `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh:17-18,170-174,189-191,236-241`
- `plugins/plugin-dev/skills/hook-development/scripts/hook-linter.sh:67-79,106-117`
- `plugins/plugin-dev/skills/hook-development/references/migration.md:14-81,83-154`
- `plugins/plugin-dev/README.md:249-257,286-290`
- `plugins/plugin-dev/commands/create-plugin.md:201-209`
- `plugins/plugin-dev/agents/plugin-validator.md:107-115`

## 依赖与外部交互

运行时依赖：

- Shell/命令：`bash`、`jq`（脚本直接依赖），`timeout`（由 `test-hook.sh` 使用）。
- Claude Hook 环境变量：
  - `CLAUDE_PROJECT_DIR`（`load-context.sh` 的目录探测根）
  - `CLAUDE_ENV_FILE`（会话环境导出目标）
  - `CLAUDE_PLUGIN_ROOT`（在同技能其它文档/脚本中作为推荐可移植路径）

外部交互方式：

- stdin 协议：Hook runtime 把事件 JSON 通过 stdin 注入。
- stderr 协议：拒绝/确认建议通过 stderr + `exit 2` 回传。
- 文件系统：
  - 读取项目标志文件（`package.json`、`go.mod` 等）
  - 写入环境导出文件（`CLAUDE_ENV_FILE`）

测试链路依赖：

- `test-hook.sh` 提供样例输入生成、环境注入、超时控制、输出 JSON 解析。
- `hook-linter.sh` 提供静态规则检查，但部分规则是启发式正则，存在误报可能。

## 风险、边界与改进建议

1. 路径约定存在文档不一致风险
- 现象：`SKILL.md` 示例多处使用 `${CLAUDE_PLUGIN_ROOT}/scripts/*.sh`，而 `create-plugin.md:207` 又强调“hook scripts 放在 examples/”，本目录实际也在 `examples/`。
- 影响：新建插件时可能出现目录放置混乱。
- 建议：统一约定（建议“运行脚本放 `hooks/scripts/`，技能内仅 `examples/` 演示”），并在 `SKILL.md` 与 `create-plugin.md` 同步修订。

2. `load-context.sh` 的 CI 检测条件可能漏判
- 现象：`load-context.sh:50` 用 `-f ".github/workflows"` 检测，该路径通常是目录。
- 影响：GitHub Actions 项目可能无法设置 `HAS_CI=true`。
- 建议：改为 `-d ".github/workflows"`，并保留对 `.gitlab-ci.yml` / `.circleci/config.yml` 的文件检测。

3. 风险判定主要是字符串匹配，易被变体绕过
- 现象：`validate-bash.sh` 仅覆盖部分危险模式（如 `rm -r -f`、编码/拼接变体未完全覆盖）。
- 影响：作为安全控制时防护深度不足。
- 建议：将此类脚本定位为“快速预筛”，复杂判断迁移到 prompt hook（与 `references/migration.md` 保持一致）。

4. `ask` 分支与退出码语义耦合较强
- 现象：示例对 `ask` 也使用 `exit 2`（`validate-write.sh:33-35`，`validate-bash.sh:38-40`）。
- 影响：依赖运行时对 `permissionDecision=ask` 与阻断退出码的组合解释，跨版本兼容性需验证。
- 建议：在 `SKILL.md` 增补“ask 场景推荐输出与退出码”专门说明，并在 `test-hook.sh` 增加该分支的断言样例。

5. 示例可运行但缺少自动化回归用例
- 现象：当前验证主要靠手工运行 `test-hook.sh` 和 linter。
- 影响：规则调整后容易回归。
- 建议：新增最小回归脚本（safe/deny/ask 三组输入），纳入 CI 的 shell test 任务。

边界说明：本目录设计目标是教学与模板，不是生产级安全网；在真实插件中需要结合 prompt hook、组织策略和更严格的测试体系。
