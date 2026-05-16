# plugins/plugin-dev/skills/plugin-settings/examples 研究

## 场景与职责

`plugins/plugin-dev/skills/plugin-settings/examples` 是 `plugin-settings` 技能的“可直接复用样例层”。它不是运行时入口目录，而是给插件作者提供三类落地模板：

1. 配置文件模板：`example-settings.md` 提供 `.claude/*.local.md` 的多种写法（基础配置、高级配置、Agent 状态、Feature Flag）。
2. 命令模板：`create-settings-command.md` 展示如何通过命令交互收集偏好并生成 `.local.md`。
3. Hook 模板：`read-settings-hook.sh` 展示 Hook 如何读取 frontmatter 并据此做阻断/放行。

其上游定位来自 `plugin-settings/SKILL.md` 的资源导航（明确列出 `examples/` 三个文件）与 `create-plugin` 流程在 Phase 5 中的 “Settings” 组件指引。其下游消费方是插件开发者：复制样例后接入自己的 `hooks/hooks.json`、commands 与 README。

## 功能点目的

### 1) 将“配置驱动插件行为”具体化

样例把抽象规范（`.claude/plugin-name.local.md` + YAML frontmatter + Markdown body）转换成可复制代码，降低实现门槛：

- 结构模板：`plugins/plugin-dev/skills/plugin-settings/examples/example-settings.md:3`
- 命令创建模板：`plugins/plugin-dev/skills/plugin-settings/examples/create-settings-command.md:12`
- Hook 读取模板：`plugins/plugin-dev/skills/plugin-settings/examples/read-settings-hook.sh:10`

### 2) 统一 quick-exit + enabled 开关模式

`read-settings-hook.sh` 先检查文件存在，再检查 `enabled`，快速返回，避免未配置时触发额外处理：

- 文件缺失直接退出：`read-settings-hook.sh:10-14`
- enabled 非 true 直接退出：`read-settings-hook.sh:24-27`

这与 `plugin-settings/SKILL.md` 的通用模式保持一致（quick-exit、enabled、重启生效）。

### 3) 提供“写配置”与“读配置”闭环

- `create-settings-command.md` 演示收集用户偏好并生成 settings 文件：`create-settings-command.md:12-81`
- `read-settings-hook.sh` 演示读取并应用这些字段：`read-settings-hook.sh:16-62`

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 数据结构：`.local.md` 双区模型

示例统一采用：

1. YAML frontmatter：结构化字段（布尔/数字/列表/字符串）。
2. Markdown body：上下文说明、任务描述或自由文本。

可见于：

- 基础模板：`example-settings.md:7-16`
- 高级模板：`example-settings.md:22-47`
- Agent 状态模板：`example-settings.md:53-89`
- Feature Flag 模板：`example-settings.md:95-113`

### B. 读取流程（Hook）

`read-settings-hook.sh` 的关键流程：

1. `set -euo pipefail` 强化失败感知：`read-settings-hook.sh:5`
2. quick-exit（文件不存在）：`read-settings-hook.sh:10-14`
3. 用 `sed` 提取 frontmatter：`read-settings-hook.sh:16-17`
4. 用 `grep + sed` 取字段（`enabled/strict_mode/max_file_size`）：`read-settings-hook.sh:20-22`
5. quick-exit（disabled）：`read-settings-hook.sh:24-27`
6. 从 stdin 读取 Hook 输入 JSON，并用 `jq` 取 `tool_input.file_path`：`read-settings-hook.sh:29-31`
7. 根据 strict_mode 执行不同路径策略，命中则输出 deny JSON 到 stderr 并 `exit 2`：`read-settings-hook.sh:34-50`
8. 可选文件大小限制（取 `tool_input.content` 长度）：`read-settings-hook.sh:53-61`
9. 全部通过则 `exit 0`：`read-settings-hook.sh:64-65`

### C. Hook 协议

`read-settings-hook.sh` 采用 PreToolUse 风格返回：

- 阻断时输出包含 `hookSpecificOutput.permissionDecision` 与 `systemMessage` 的 JSON（stderr）：`read-settings-hook.sh:37-49,59`
- 使用 `exit 2` 让 Claude 读取阻断决策：`read-settings-hook.sh:38,43,49,60`

该协议与 `hook-development/SKILL.md` 一致：

- 输出格式：`plugins/plugin-dev/skills/hook-development/SKILL.md:144-153`
- 退出码语义（`exit 2` 回传 stderr）：`plugins/plugin-dev/skills/hook-development/SKILL.md:176-179`

### D. 写入流程（命令）

`create-settings-command.md` 定义命令模板：

1. 通过 `AskUserQuestion` 收集偏好：`create-settings-command.md:12-55`
2. 解析回答映射（enabled/mode）：`create-settings-command.md:57-63`
3. 用 `Write` 生成 `.claude/my-plugin.local.md`：`create-settings-command.md:64-81`
4. 回告用户重启要求与 gitignore：`create-settings-command.md:83-90`
5. 要求输入校验与路径安全：`create-settings-command.md:92-98`

### E. 相关脚本命令（上下文依赖）

目录外配套脚本在 `scripts/`：

- `parse-frontmatter.sh`：提取 frontmatter 或单字段。`plugins/plugin-dev/skills/plugin-settings/scripts/parse-frontmatter.sh:37-58`
- `validate-settings.sh`：检查 marker、frontmatter、常见字段与正文存在性。`plugins/plugin-dev/skills/plugin-settings/scripts/validate-settings.sh:40-101`

本次实测（命令）：

```bash
bash plugins/plugin-dev/skills/plugin-settings/scripts/parse-frontmatter.sh \
  plugins/plugin-dev/skills/plugin-settings/examples/example-settings.md

bash plugins/plugin-dev/skills/plugin-settings/scripts/validate-settings.sh \
  plugins/plugin-dev/skills/plugin-settings/examples/example-settings.md
```

结果表明脚本会把 `example-settings.md` 中多个模板代码块的 `---` 段都纳入解析（不是单个真实 `.local.md` 文件语义）。

## 关键代码路径与文件引用

### 目标目录（被研究对象）

- `plugins/plugin-dev/skills/plugin-settings/examples/example-settings.md`
- `plugins/plugin-dev/skills/plugin-settings/examples/create-settings-command.md`
- `plugins/plugin-dev/skills/plugin-settings/examples/read-settings-hook.sh`

### 直接上游（调用方/规范来源）

- `plugins/plugin-dev/skills/plugin-settings/SKILL.md:97,519-523,527-530`
  - 把本目录声明为官方 examples，并指向 scripts。
- `plugins/plugin-dev/commands/create-plugin.md:157-163,220-226`
  - 在插件创建流程中要求加载 `plugin-settings` 并产出 settings 模板、读取逻辑、gitignore。
- `plugins/plugin-dev/README.md:114-133`
  - 文档层描述 plugin settings 能力与资源组成（examples/references/scripts）。

### 直接下游（被调用方/运行时宿主）

- 插件 Hook 引擎（PreToolUse）消费 `read-settings-hook.sh` 的 stdin JSON 与 stderr 决策 JSON，语义见：
  - `plugins/plugin-dev/skills/hook-development/SKILL.md:123-153,176-179`
- 插件命令系统消费 `create-settings-command.md` frontmatter（`allowed-tools`）与步骤规范。

### 相关配套（解析与参考）

- `plugins/plugin-dev/skills/plugin-settings/scripts/parse-frontmatter.sh`
- `plugins/plugin-dev/skills/plugin-settings/scripts/validate-settings.sh`
- `plugins/plugin-dev/skills/plugin-settings/references/parsing-techniques.md`
- `plugins/plugin-dev/skills/plugin-settings/references/real-world-examples.md`

### 测试与校验现状

- 该 `examples/` 目录下无独立自动化测试文件（未检索到 `*test*`）。
- 主要依赖文档化样例 + 配套脚本手工验证。
- 可借用 hook-development 的工具脚本进行补测（如 `validate-hook-schema.sh`、`test-hook.sh`），路径见：
  - `plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh`
  - `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh`

## 依赖与外部交互

### Shell/CLI 依赖

- `bash`（脚本运行）
- `sed`/`grep`/`awk`（frontmatter/body 解析）
- `jq`（解析 Hook 输入 JSON）

见：

- `read-settings-hook.sh:17,20-22,31,55`
- `parse-frontmatter.sh:37,51`
- `validate-settings.sh:41,55,71,86`

### Claude 运行时依赖

- Hook 协议：通过 stdin 获取事件输入，stderr 输出控制决策，`exit 2` 触发 deny/ask 语义。
- 命令工具权限：`create-settings-command.md` 仅允许 `Write` 与 `AskUserQuestion`。

### 配置与生命周期交互

- settings 文件建议放在项目根 `.claude/*.local.md`，且加入 `.gitignore`（用户本地态，不入库）。
- 修改后通常需要重启 Claude Code 才会稳定生效（Hook 加载时机限制）。

依据：

- `example-settings.md:136-159`
- `plugin-settings/SKILL.md:329-334,367-383`
- `hook-development/SKILL.md:574-583`

## 风险、边界与改进建议

### 风险 1：基于正则/行匹配的 YAML 解析能力有限

现状：

- `grep '^key:'` + `sed` 对嵌套对象、多行字符串、复杂数组不稳健。
- `example-settings.md` 的 feature 列表是块序列（多行），`read-settings-hook.sh` 并未处理这类字段。

建议：

1. 简单字段继续用当前方案，复杂结构切换 `yq`（作为可选依赖，带降级策略）。
2. 在样例中明确“支持字段形态”边界，避免误用。

### 风险 2：`parse-frontmatter.sh` / `validate-settings.sh` 对“文档型样例文件”会产生误判

实测现象：

- `example-settings.md` 含多个 fenced code frontmatter；脚本会提取并拼接多个块字段，导致重复 `enabled`。
- `validate-settings.sh` 仍给出结构有效结论，同时提示布尔字段异常（多行拼接）。

影响：

- 若开发者把文档样例当作真实配置文件输入脚本，结果可能与预期偏离。

建议：

1. 在脚本中增加“frontmatter 仅允许首段”或“必须文件开头即 `---`”校验。
2. 增加 `--strict` 模式：要求 marker 为开头两段、禁止多个 frontmatter 块。
3. 在 `example-settings.md` 顶部加提示：该文件是文档模板集合，不是单一可解析 settings 实例。

### 风险 3：Hook deny JSON 使用字符串拼接，存在转义边界

现状：

- `read-settings-hook.sh:59` 在 JSON 字符串中拼接 `$MAX_SIZE`，当前来源是数字字段，风险较低。

建议：

1. 统一改为 `jq -n --arg ...` 生成 JSON（`parsing-techniques.md` 已给出建议模式）。
2. 在样例中示范“所有动态输出都走 jq 构造”。

### 风险 4：目录缺少可执行验证闭环

现状：

- `examples/` 目录无自动化测试，主要依赖阅读与手工复制。

建议：

1. 增加 `examples/testdata/`（合法/非法 `.local.md`）。
2. 增加 `scripts/smoke-test-examples.sh`：
   - 校验 `read-settings-hook.sh` 的 allow/deny 路径与 exit code。
   - 校验 `create-settings-command.md` 示例输出字段完整性。
3. 在 CI 中至少跑一次 shellcheck + smoke test。

