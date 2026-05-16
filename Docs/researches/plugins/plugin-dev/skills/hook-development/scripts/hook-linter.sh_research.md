# FILE 研究：plugins/plugin-dev/skills/hook-development/scripts/hook-linter.sh

## 场景与职责

`hook-linter.sh` 是 hook 开发阶段的“静态前置闸门”，目标是在 hook 真正接入 Claude 事件前，快速发现 bash 脚本层面的常见问题（可执行性、安全习惯、输出约定、可移植性）。

它不负责：

- 校验 `hooks.json` 结构（由 `validate-hook-schema.sh` 负责）
- 模拟 hook 运行（由 `test-hook.sh` 负责）

调用链位置：

- 文档入口：`plugins/plugin-dev/skills/hook-development/scripts/README.md:62-90`
- 工具链定位：`plugins/plugin-dev/README.md:273-284`
- 流程要求：`plugins/plugin-dev/commands/create-plugin.md:208,259`

## 功能点目的

脚本提供 13 条检查（`hook-linter.sh:36-117`），可归纳为三类：

1. 可执行与健壮性
- 可执行位、shebang、`set -euo pipefail`、显式退出码。

2. 输入输出与协议习惯
- 是否读 stdin、是否使用 `jq` 解析 hook 输入、错误信息是否写 stderr。

3. 安全与可移植性
- 未引用变量风险、硬编码绝对路径、潜在长耗时逻辑（`sleep 100+`、`while true`）。

目的不是“证明脚本正确”，而是“以低成本筛掉明显 bad smell”。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 参数入口与主循环

- 无参数时输出 usage 并 `exit 1`（`8-21`）。
- 对每个入参文件调用 `check_script`（`140-145`）。
- 汇总维度是“有无 error”，warning 不影响总退出码（`147-153`）。

### 2) 单文件检查函数

`check_script()`（`23-132`）核心是 grep/head 驱动的启发式静态分析：

- 文件存在性：`[ -f ]`
- 首行 shebang：`head -1`
- 文本规则：`grep -q` / `grep -E`

典型正则规则：

- 未引用变量：`\$[A-Za-z_][A-Za-z0-9_]*[^"]`（`68`）
- 绝对路径：`^[^#]*/home/|^[^#]*/usr/|^[^#]*/opt/`（`75`）
- 长耗时：`sleep [0-9]{3,}|while true`（`100`）

### 3) 退出语义

- 单文件：有 error 则返回 1；仅 warning 仍返回 0（`122-131`）。
- 多文件：任一文件有 error，最终 `exit 1`（`147-153`）。

这与 README 的定位一致：lint 是“建议+阻断错误”的混合模式。

## 关键代码路径与文件引用

核心实现：

- `plugins/plugin-dev/skills/hook-development/scripts/hook-linter.sh:1-153`

文档与调用上下文：

- `plugins/plugin-dev/skills/hook-development/scripts/README.md:62-102`
- `plugins/plugin-dev/README.md:273-284`
- `plugins/plugin-dev/skills/hook-development/SKILL.md:685-709`
- `plugins/plugin-dev/commands/create-plugin.md:208,259`

典型被检对象：

- `plugins/plugin-dev/skills/hook-development/examples/validate-write.sh`
- `plugins/plugin-dev/skills/hook-development/examples/validate-bash.sh`
- `plugins/plugin-dev/skills/hook-development/examples/load-context.sh`

## 依赖与外部交互

运行依赖：

- `bash`
- `head`
- `grep`

输入输出交互：

- 输入：一到多个本地脚本路径
- 输出：stdout 打印 lint 结果
- 无网络调用、无外部 API

副作用：

- 无文件写入（纯读 + 输出）

## 风险、边界与改进建议

1. 规则是启发式，存在误报/漏报。
- 未引用变量正则（`68`）无法理解 shell 语法上下文，容易把合法场景判成风险。
- “是否读 stdin”仅查 `cat|read`（`56`），对复杂读取方式可能漏检。

2. 某些规则的语义匹配较宽。
- stderr 检查正则（`107`）混合了多个 `|` 分支，且未做严格分组，可能匹配到与错误无关的文本。

3. 缺少机器可消费输出。
- 当前仅人类可读文本，不便于 CI 聚合（如 JSON/SARIF）。

4. 与 ShellCheck 没有形成互补联动。
- 该脚本覆盖的是项目约定；ShellCheck 覆盖语法/可移植性。两者并行更稳。

建议：

- 增加 `--format json` 输出，便于 CI 门禁和报告归档。
- 把高误报规则分级（info/warn/error）并支持 `--strict`。
- 可选集成 `shellcheck`：检测到命令存在时自动补充检查。
