# FILE 研究：plugins/plugin-dev/skills/hook-development/examples/load-context.sh

## 场景与职责
`load-context.sh` 是 `hook-development` 技能中的 `SessionStart` 命令型 Hook 示例脚本。它在 Claude Code 会话启动时执行，用于基于项目根目录中的标记文件推断技术栈，并将推断结果写入 `CLAUDE_ENV_FILE`，供后续会话命令继承。

该脚本本身不承担业务逻辑执行，核心职责是“会话初始化上下文注入”：
- 将项目类型抽象成统一环境变量（`PROJECT_TYPE`、`BUILD_SYSTEM`、`USES_TYPESCRIPT`）。
- 提供可移植、低成本的语言/构建系统探测模板。
- 示范 `SessionStart` 钩子如何持久化环境变量。

## 功能点目的
1. 项目类型探测：基于常见生态文件（`package.json`、`Cargo.toml`、`go.mod` 等）识别语言/生态。
2. TypeScript 补充识别：在 Node 项目分支下检查 `tsconfig.json` 并设置 `USES_TYPESCRIPT=true`。
3. Java 构建体系识别：区分 Maven 与 Gradle，额外写入 `BUILD_SYSTEM`。
4. 未识别兜底：写入 `PROJECT_TYPE=unknown`，避免下游变量缺失。
5. CI 能力标记：若检测到 CI 配置，写入 `HAS_CI=true`。

## 具体技术实现（关键流程/数据结构/协议/命令）
脚本实现位于 `plugins/plugin-dev/skills/hook-development/examples/load-context.sh:1-55`，流程如下：

1. 运行安全基线
- `set -euo pipefail`：命令失败即退出、未定义变量报错、管道失败可见。

2. 上下文切换
- `cd "$CLAUDE_PROJECT_DIR" || exit 1`：在项目根执行文件探测，依赖 `SessionStart` 提供的环境变量。

3. 规则链判定（顺序优先）
- 按 `if/elif` 顺序探测：Node -> Rust -> Go -> Python -> Java(Maven) -> Java/Kotlin(Gradle) -> unknown。
- 命中后通过 `echo "export KEY=VALUE" >> "$CLAUDE_ENV_FILE"` 持久化。
- 这是“先命中先返回语义”的规则链，不是多标签并存模型。

4. CI 标记
- 使用 `-f` 检测 `.github/workflows`、`.gitlab-ci.yml`、`.circleci/config.yml`。
- 命中后写入 `HAS_CI=true`。

5. 协议行为
- 成功返回 `exit 0`。
- 此类 `SessionStart` 示例不消费 stdin JSON，主要基于环境变量和文件系统状态。

实测（本次研究执行）：
- 在临时 Node+TS 项目中（`package.json` + `tsconfig.json` + `.github/workflows/` 目录）运行，`env` 文件写入：
  - `export PROJECT_TYPE=nodejs`
  - `export USES_TYPESCRIPT=true`
- 未写入 `HAS_CI`，与脚本中 `.github/workflows` 使用 `-f`（文件）检测一致，目录场景漏判。

## 关键代码路径与文件引用
- 主脚本：`plugins/plugin-dev/skills/hook-development/examples/load-context.sh:1-55`
- Hook 事件与环境变量协议：`plugins/plugin-dev/skills/hook-development/SKILL.md:238-264`、`plugins/plugin-dev/skills/hook-development/SKILL.md:322-329`
- 插件配置示例（SessionStart 调用 command hook）：`plugins/plugin-dev/skills/hook-development/SKILL.md:368-376`
- 模式文档中的同类示例：`plugins/plugin-dev/skills/hook-development/references/patterns.md:49-84`
- 测试脚本（SessionStart 样例输入与环境注入）：`plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh:83-93`、`plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh:170-174`、`plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh:235-241`
- 技能总览中对该示例脚本的资源声明：`plugins/plugin-dev/README.md:68-72`

## 依赖与外部交互
1. Shell 与运行时
- 依赖 `/bin/bash`，并要求脚本在 POSIX-like 文件系统环境中运行。

2. Claude Code Hook 环境变量
- `CLAUDE_PROJECT_DIR`：项目根目录。
- `CLAUDE_ENV_FILE`：SessionStart 可写环境文件。
- 脚本未使用 `CLAUDE_PLUGIN_ROOT`，但上层配置通常用它定位脚本路径。

3. 文件系统交互
- 读取项目根标记文件：`package.json`、`Cargo.toml`、`go.mod`、`pyproject.toml`、`setup.py`、`pom.xml`、`build.gradle`、`build.gradle.kts`。
- 读取 CI 标记路径：`.github/workflows`、`.gitlab-ci.yml`、`.circleci/config.yml`。
- 追加写入：`$CLAUDE_ENV_FILE`。

4. 上游调用方
- 典型由 `hooks/hooks.json` 的 `SessionStart` 事件触发（文档示例为 `bash ${CLAUDE_PLUGIN_ROOT}/scripts/load-context.sh`）。

## 风险、边界与改进建议
1. CI 检测误判
- 风险：`.github/workflows` 实际通常是目录，当前 `-f` 会漏判。
- 建议：改为 `[ -d ".github/workflows" ]`，或同时兼容 `-d/-f`。

2. 多生态项目识别偏置
- 风险：`if/elif` 顺序导致多语言仓库只保留首个命中标签（例如 Node + Go monorepo）。
- 建议：支持多标签写入（如 `PROJECT_STACK=nodejs,go`），或允许配置优先级。

3. 环境文件幂等性
- 风险：每次 SessionStart 追加写入，可能出现重复 `export`。
- 建议：写入前去重，或重建临时 env 文件后原子替换。

4. 异常路径缺乏可观测性
- 风险：`cd` 失败直接退出，错误上下文较弱。
- 建议：在失败前输出带路径的 stderr 诊断信息，便于定位。

5. 规则可扩展性
- 风险：当前规则硬编码在脚本中，扩展语言时需要改脚本。
- 建议：将“标记文件 -> 变量集”外置为配置映射（JSON/YAML），脚本仅负责解释执行。
