# DIR 研究：plugins/plugin-dev/skills/plugin-structure/examples

## 场景与职责

`plugins/plugin-dev/skills/plugin-structure/examples` 是 `plugin-structure` skill 的“可复制示例层”，职责是把结构规范从抽象规则变成三种可落地模板：`minimal`、`standard`、`advanced`。

它在上下文中的位置如下：

1. 上游调用方（谁把用户引到这里）
- `plugins/plugin-dev/skills/plugin-structure/README.md:50-72`：明确列出 3 个示例并声明用途。
- `plugins/plugin-dev/skills/plugin-structure/README.md:87-93`：将 `examples/` 设为 progressive disclosure 的第 3 层。
- `plugins/plugin-dev/README.md:286-293`：将该目录描述为 “3 plugin layouts (minimal, standard, advanced)”。
- `plugins/plugin-dev/commands/create-plugin.md:48-52,116-149`：创建插件流程在 Phase 2/4 依赖 `plugin-structure` 规范，示例是用户常见落地参照。

2. 下游被调用方（示例落地后由谁消费）
- 插件开发者：直接复制目录结构与片段实现新插件。
- Claude Code 运行时：按 manifest + 自动发现机制加载组件（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:343-349`，`plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:11-16`）。
- 校验链路：`plugin-validator` 和 `skill-reviewer` 会按结构/frontmatter/资源完整性进行验收（`plugins/plugin-dev/agents/plugin-validator.md:67-123`，`plugins/plugin-dev/agents/skill-reviewer.md:52-83`）。

3. 目录职责边界
- 该目录是文档样例，不是可执行实现目录；本身无脚本与测试文件（`find plugins/plugin-dev/skills/plugin-structure -maxdepth 3 -type f` 仅包含 markdown）。
- 运行行为由“被示例引用的脚本和配置协议”决定，不在本目录内直接执行。

## 功能点目的

### 1) `minimal-plugin.md`：验证最小可运行闭环

- 只保留 `.claude-plugin/plugin.json` 的 `name` 与一个 `commands/hello.md`（`minimal-plugin.md:7-23,25-46`）。
- 目的：证明“最小 manifest + 默认目录自动发现”即可生成可用命令（`minimal-plugin.md:64-67`）。
- 适用：快速原型、单功能工具、教学入门（`minimal-plugin.md:69-75`）。

### 2) `standard-plugin.md`：给出生产常见组合

- 展示 commands/agents/skills/hooks/scripts 组合结构（`standard-plugin.md:7-35`）。
- 给出较完整的 manifest 元数据（`standard-plugin.md:41-55`）。
- 演示跨组件协作：命令 -> 脚本、agent -> skill、hook -> 脚本（`standard-plugin.md:57-128,130-222,426-504`）。
- 目的：作为“中等复杂度团队插件”的通用蓝本（`standard-plugin.md:574-587`）。

### 3) `advanced-plugin.md`：演示企业级复杂组织

- 使用多层目录并通过 manifest 显式声明 `commands/agents/hooks/mcpServers`（`advanced-plugin.md:131-142`）。
- 展示多 MCP server、分层 hooks、共享 `lib/`、环境配置管理（`advanced-plugin.md:145-175,629-697,714-765`）。
- 目的：覆盖多环境部署、CI/CD、安全与可观测性的复杂场景（`advanced-plugin.md:741-757`）。

### 4) 三层示例的组合价值

- 形成“复杂度阶梯”：最小可用 -> 生产组合 -> 企业级扩展。
- 与 `plugin-structure` 的“标准布局 + 自动发现 + 可移植路径”核心叙事一致（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:22-45,339-355`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 关键流程（示例被消费的真实链路）

1. 开发者在 plugin 设计阶段触发 `plugin-structure`（`create-plugin.md:48-52`）。
2. 根据示例确定目录与 manifest 形态（`create-plugin.md:116-149` + `examples/*.md`）。
3. Claude Code 启动/启用插件时读取 `.claude-plugin/plugin.json`，扫描默认与自定义路径，注册组件并初始化 hooks/MCP（`component-patterns.md:11-16`，`manifest-reference.md:354-371`）。
4. 运行时按组件类型激活：命令调用、agent 选择、skill 匹配、hook 事件触发、MCP 工具转发（`component-patterns.md:23-27`）。

### B. 关键数据结构

1. Manifest 结构分层
- Minimal：`{ "name": "hello-world" }`（`minimal-plugin.md:19-23`）。
- Standard：元数据增强但不覆盖默认路径（`standard-plugin.md:42-54`）。
- Advanced：显式组件路径 + hooks/MCP 文件入口（`advanced-plugin.md:131-142`）。
- 与参考规范对应：`commands/agents` 可 string/array，`hooks/mcpServers` 可 path/object（`manifest-reference.md:211-331`）。

2. Command/Agent/Skill 文件协议
- Commands：markdown + YAML frontmatter（`standard-plugin.md:60-63,98-101`；`SKILL.md:112-132`）。
- Agents（示例中）：`description + capabilities` 风格 frontmatter（`standard-plugin.md:133-141`，`advanced-plugin.md:228-236`）。
- Skills：`SKILL.md` frontmatter + references/examples/scripts 资源层（`standard-plugin.md:227-231,323-424`，`advanced-plugin.md:313-317`）。

3. Hook 配置协议
- 事件结构为 `EventName[]`，每项含 `matcher` 与 `hooks` 列表，hook 项含 `type/prompt|command/timeout`（`standard-plugin.md:429-454`，`advanced-plugin.md:631-696`）。
- 事件覆盖：`PreToolUse`、`PostToolUse`、`Stop`、`SessionStart`（`advanced-plugin.md:633-695`）。

4. MCP 配置协议
- advanced 示例的 `.mcp.json` 使用 `{ "mcpServers": { ... } }` 包装结构（`advanced-plugin.md:147-175`）。
- server 配置包含 `command/args/env`，并演示环境变量默认值语法 `${K8S_NAMESPACE:-default}`（`advanced-plugin.md:151-156`）。

### C. 关键命令与脚本执行语义

1. 便携路径
- 命令与 hooks 均通过 `${CLAUDE_PLUGIN_ROOT}` 引用脚本，避免安装路径耦合（`standard-plugin.md:81,448`，`advanced-plugin.md:152,160,168,639,661,673,678,690`）。

2. 质量校验脚本（standard）
- `validate-commit.sh` 通过 `git status`、`git diff --name-only --cached` 获取变更文件（`standard-plugin.md:466-473`）。
- 对 JS/TS 执行 `npx eslint`，对 Python 执行 `python -m pylint`（`standard-plugin.md:485-492`）。
- 通过 `exit 0/1` + JSON `systemMessage` 回传结果（`standard-plugin.md:467,475,498-503`）。

3. MCP 工具调用（advanced）
- 命令侧示例：`tools.github_actions_trigger_workflow(...)`（`advanced-plugin.md:200-203`）。
- agent 侧示例：引用 Datadog/Slack 集成库并发送部署状态（`advanced-plugin.md:293-307`）。

4. 运维命令样例（advanced skill 内容）
- 含 `kubectl describe/logs/exec/top` 等排障命令（`advanced-plugin.md:491-505`）。
- 这些命令属于 skill 知识样例，不是本目录可执行脚本。

### D. 与“配置/测试/脚本”上下文的关系

- 配置：`manifest-reference.md` 提供字段与默认路径契约；examples 负责可读可抄模板化（`manifest-reference.md:214-237,247-263,301-309,356-367`）。
- 测试：本目录无独立测试；验证依赖 create-plugin Phase 6 的外部工具链（`create-plugin.md:255-260`）。
- 脚本：示例内仅展示脚本内容/调用方式，真实脚本并不在该目录落盘。

## 关键代码路径与文件引用

### 目标目录（研究对象）
- `plugins/plugin-dev/skills/plugin-structure/examples/minimal-plugin.md`
- `plugins/plugin-dev/skills/plugin-structure/examples/standard-plugin.md`
- `plugins/plugin-dev/skills/plugin-structure/examples/advanced-plugin.md`

### 上游调用方（流程入口）
- `plugins/plugin-dev/skills/plugin-structure/README.md:50-72,87-93`
- `plugins/plugin-dev/skills/plugin-structure/SKILL.md:339-349,476`
- `plugins/plugin-dev/README.md:286-293`
- `plugins/plugin-dev/commands/create-plugin.md:48-52,116-149,157-164`

### 下游被调用方（验收与运行）
- `plugins/plugin-dev/agents/plugin-validator.md:67-123`
- `plugins/plugin-dev/agents/skill-reviewer.md:52-83`
- `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:211-237,259-331,352-371`
- `plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:11-27,93-121,291-374,448-474`

### 横向耦合（规范一致性）
- `plugins/plugin-dev/skills/agent-development/SKILL.md:60-145,353-357`（agent frontmatter required 字段）
- `plugins/plugin-dev/skills/mcp-integration/SKILL.md:23-37,44-59`（`.mcp.json` 与 inline `mcpServers` 形态）
- `plugins/plugin-dev/skills/mcp-integration/examples/stdio-server.json:1-26`（顶层 server map 示例）

## 依赖与外部交互

### 1) 本地依赖

- 文档资产依赖 markdown 解析与人工复制，不依赖本目录脚本执行。
- 运行时依赖 Claude Code 插件发现/注册机制（manifest + 自动发现目录）。

### 2) 工具链与命令依赖（由示例引入）

- Git：`git status`、`git diff --name-only --cached`（`standard-plugin.md:466-473`）。
- Node 工具链：`npx eslint`（`standard-plugin.md:485`）。
- Python 工具链：`python -m pylint`（`standard-plugin.md:490`）。
- Shell：`bash ${CLAUDE_PLUGIN_ROOT}/...`（standard/advanced 多处）。

### 3) 外部系统交互（由示例建模）

- MCP server：GitHub Actions、Kubernetes、Terraform（`advanced-plugin.md:149-173,709-712`）。
- 观测与通知：Datadog、Slack、PagerDuty（`advanced-plugin.md:291-307,736-739`）。
- 集群操作：`kubectl` 命令序列（`advanced-plugin.md:491-505`）。

### 4) 测试与验证边界

- 本目录没有自动化测试或 schema 校验脚本。
- 质量保障主要依赖外部流程：`validate-agent.sh`、`validate-hook-schema.sh`、`test-hook.sh`、plugin-validator agent（`create-plugin.md:199,255-260`，`plugin-validator.md:89,108`）。

## 风险、边界与改进建议

1. 高风险：agent 示例 frontmatter 与当前 agent 规范漂移  
现象：
- examples 中 agent 使用 `description + capabilities`（`standard-plugin.md:133-141`，`advanced-plugin.md:228-236`）。
- agent-development 规范要求 `name/description/model/color` 为必填（`agent-development/SKILL.md:60-145,353-357`）。  
影响：
- 用户按示例复制后可能被 validator/运行时规则拦截或降级。  
建议：
- 将两个示例 agent 升级到当前规范；至少加“旧格式示例”显式注释。

2. 中风险：`.mcp.json` canonical 形态跨技能不一致  
现象：
- advanced 示例使用 `{ "mcpServers": {...} }`（`advanced-plugin.md:147-175`）。
- mcp-integration 示例常用顶层 server map（`mcp-integration/examples/stdio-server.json:1-26`，`mcp-integration/SKILL.md:23-37`）。  
影响：
- 用户跨技能复制时可能产生格式不确定性。  
建议：
- 在两个技能中统一“推荐形态 + 兼容形态”并给单一 copy-paste 首选模板。

3. 中风险：hooks 脚本目录约定跨文档冲突  
现象：
- plugin-structure 示例与 SKILL 明确使用 `hooks/scripts/`（`standard-plugin.md:448`，`advanced-plugin.md:639-690`，`plugin-structure/SKILL.md:208-213`）。
- create-plugin 文档写“hook scripts if needed (in examples/ not scripts/)”（`create-plugin.md:207`）。  
影响：
- 开发者会在 `hooks/scripts` 与 `examples` 之间摇摆，影响自动发现与维护一致性。  
建议：
- 统一到“运行脚本放 hooks/scripts，examples 仅放教学样例”并修正文档冲突。

4. 中风险：示例宣称 complete，但文件内容是“选段式”  
现象：
- standard/advanced 目录树列出了大量文件，但 File Contents 仅展示关键片段（如 standard 只给 `lint.md`/`test.md`，advanced 只给 `build.md` 等）。  
影响：
- 用户误以为“完整拷贝即可运行”，实际仍需补齐缺失文件。  
建议：
- 在三个示例顶部标注“示意样例（partial excerpts）”；或提供可下载的完整模板仓库链接。

5. 中风险：示例目录缺少自动回归检查  
现象：
- 无脚本校验 agent frontmatter、MCP 结构、hooks 命令路径与 `${CLAUDE_PLUGIN_ROOT}` 覆盖率。  
影响：
- 文档漂移难以在 CI/日常维护中提前发现。  
建议：
- 新增轻量脚本（例如 `scripts/validate-plugin-structure-examples.sh`）做静态检查，并接入文档维护流程。

6. 边界结论  
- 本目录核心价值是“样例教学与结构对齐”，不是运行时执行效率优化点。
- 其质量关键在于：示例可复制性、跨技能协议一致性、与 validator 规则同步。
