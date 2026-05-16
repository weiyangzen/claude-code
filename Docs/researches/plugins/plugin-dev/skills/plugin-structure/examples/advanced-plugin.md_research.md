# plugins/plugin-dev/skills/plugin-structure/examples/advanced-plugin.md 研究

## 场景与职责
`advanced-plugin.md` 是 `plugin-structure` 的企业级模板，职责是示范“复杂插件如何组织并可持续演进”，重点不是单一命令，而是平台化能力整合：

- 通过分层目录组织 commands/agents/skills/hooks/servers/lib/config，支持多人协作与大规模扩展（`advanced-plugin.md:7-99`）。
- 通过 manifest 显式声明多命令/多 agent 路径、hooks 文件、MCP 配置文件，适配嵌套目录而非默认扁平扫描（`advanced-plugin.md:131-142`，`component-patterns.md:93-121`）。
- 通过多 MCP server（kubernetes/terraform/github-actions）把 Claude Code 能力与外部 DevOps 系统对接（`advanced-plugin.md:147-175,709-712`）。
- 通过多事件 hooks + 分组脚本实现“安全、质量、流程”分层治理（`advanced-plugin.md:629-697,727-733`）。

文档调用关系：

- `plugin-structure/README.md` 将该文件作为 enterprise-grade 示例（`README.md:65-71`）。
- `plugin-structure/SKILL.md` 在 full-featured 与 portability 规则中可直接映射到本示例写法（`SKILL.md:86-107,256-301,417-432`）。

## 功能点目的
1. 显式路径配置支持“分组目录”
- `commands` 指向 `./commands/ci`、`./commands/monitoring`、`./commands/admin`；`agents` 指向两组子目录（`advanced-plugin.md:131-139`）。
- 目的：规避默认发现对深层目录支持不足的问题（`component-patterns.md:111-121`）。

2. 将 MCP 集成作为一等能力
- `mcpServers: "./.mcp.json"` 与 `.mcp.json` 内三服务定义形成“声明-实现”闭环（`advanced-plugin.md:141,147-175`）。
- 目的：让 build/deploy/infra 命令可以调用外部系统工具链（`advanced-plugin.md:197-204,707-713`）。

3. hooks 多阶段编排
- `SessionStart` 做权限校验，`PreToolUse` 做敏感操作前置检查，`PostToolUse` 更新状态，`Stop` 执行配置检查与团队通知（`advanced-plugin.md:633-695`）。
- 目的：把安全、流程、审计自动化嵌入会话生命周期。

4. 共享库与配置层治理
- `lib/core|integrations|utils` 提供复用基础能力；`config/environments|templates` 提供多环境参数与模板（`advanced-plugin.md:79-99,714-726`）。
- 目的：降低跨组件重复逻辑，支持环境隔离与标准化交付。

5. 面向企业运维知识沉淀
- `skills/kubernetes-ops` 展示大量操作策略、排障命令、安全规范与脚本入口（`advanced-plugin.md:310-627`）。

## 具体技术实现（关键流程/数据结构/协议/命令）
### 1) 关键流程：加载、执行、治理
- 加载阶段
1. 读取 `.claude-plugin/plugin.json`（`advanced-plugin.md:103-143`）。
2. 根据 `commands/agents` 数组路径加载嵌套目录组件（`advanced-plugin.md:131-139`）。
3. 加载 `hooks/hooks.json` 与 `.mcp.json`（`advanced-plugin.md:140-142`）。
4. 启动会话时触发 `SessionStart` hook 执行权限校验脚本（`advanced-plugin.md:684-693`）。

- 运行阶段（CI/build 典型链路）
1. 用户触发 `build` 命令。
2. 命令通过 MCP 工具调用 GitHub Actions workflow（`advanced-plugin.md:197-204`）。
3. Bash 工具执行后触发 `PostToolUse` 更新流程状态（`advanced-plugin.md:655-665`）。
4. 会话结束触发 `Stop`，执行配置检查与通知（`advanced-plugin.md:667-682`）。

### 2) 核心数据结构
- `plugin.json`（显式路径版）
```json
{
  "commands": ["./commands/ci", "./commands/monitoring", "./commands/admin"],
  "agents": ["./agents/orchestration", "./agents/specialized"],
  "hooks": "./hooks/hooks.json",
  "mcpServers": "./.mcp.json"
}
```
来源：`advanced-plugin.md:131-142`。

- `.mcp.json`（多 server）
- server 维度字段：`command`、`args`、`env`（`advanced-plugin.md:149-173`）。
- 使用 `${CLAUDE_PLUGIN_ROOT}` 保证安装位置无关（`advanced-plugin.md:152,160,168`）。
- 使用 shell 默认值语法 `${K8S_NAMESPACE:-default}` 体现环境变量容错（`advanced-plugin.md:155`）。

- `hooks/hooks.json`（多事件）
- 事件：`PreToolUse`、`PostToolUse`、`Stop`、`SessionStart`。
- matcher：`Write|Edit`、`Bash`、`.*`。
- hook 子项：`type=prompt|command` + `timeout`（`advanced-plugin.md:633-695`）。

### 3) 协议与命令约定
- 命令/agent/skill 使用 Markdown + YAML frontmatter 协议（`advanced-plugin.md:180-236,313-317`；`SKILL.md:124-133,150-160,185-194`）。
- hooks 使用事件映射 JSON 协议（`advanced-plugin.md:631-697`）。
- MCP 使用 server map 协议；每个服务绑定外部进程或远程连接配置（`mcp-integration/SKILL.md:23-37,67-82`，`references/server-types.md:11-36`）。

### 4) 关键实现命令示例
- MCP 触发 CI：`tools.github_actions_trigger_workflow(...)`（`advanced-plugin.md:200-203`）。
- K8s 运维：`kubectl describe/logs/exec/top` 系列（`advanced-plugin.md:491-505`）。
- Skill 脚本：`bash ${CLAUDE_PLUGIN_ROOT}/skills/kubernetes-ops/scripts/validate-manifest.sh ...`（`advanced-plugin.md:623-626`）。

## 关键代码路径与文件引用
- `plugins/plugin-dev/skills/plugin-structure/examples/advanced-plugin.md:7-99`：企业级目录分层全貌。
- `plugins/plugin-dev/skills/plugin-structure/examples/advanced-plugin.md:131-142`：manifest 显式路径配置。
- `plugins/plugin-dev/skills/plugin-structure/examples/advanced-plugin.md:147-175`：`.mcp.json` 三服务定义。
- `plugins/plugin-dev/skills/plugin-structure/examples/advanced-plugin.md:178-223`：CI build 命令与 MCP 工具调用。
- `plugins/plugin-dev/skills/plugin-structure/examples/advanced-plugin.md:225-308`：部署编排 agent 与集成调用。
- `plugins/plugin-dev/skills/plugin-structure/examples/advanced-plugin.md:310-627`：Kubernetes skill 知识体、脚本入口与命令。
- `plugins/plugin-dev/skills/plugin-structure/examples/advanced-plugin.md:629-697`：多事件 hooks 编排。
- `plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:93-121,481-500`：分层/层次化组织模式依据。
- `plugins/plugin-dev/skills/mcp-integration/SKILL.md:23-37,46-59`：`.mcp.json` 与 inline 两种配置基准。

## 依赖与外部交互
### 仓库内依赖（规范与文档）
- `plugin-structure/SKILL.md` 与 `references/*`：目录与 manifest path 规则。
- `mcp-integration`：MCP 连接类型、配置实践、错误处理建议。
- `hook-development`：hook 事件语义、matcher 与 timeout 策略。
- `agent-development`：agent frontmatter 新规范（与当前示例进行兼容性对照）。

### 外部系统交互
- CI/CD：GitHub Actions（workflow 触发）。
- 基础设施：Kubernetes、Terraform。
- 观测与通知：Datadog、Slack、PagerDuty（`advanced-plugin.md:291-307,736-739`）。
- 运维命令：`kubectl` 与 shell 脚本集合（安全/质量/流程）。

### 运行前置依赖
- Node、Python 运行时。
- 各 MCP server 的依赖包（例如 `package.json` / `requirements.txt`）。
- 环境变量：`KUBECONFIG`、`TF_STATE_BUCKET`、`AWS_REGION`、`GITHUB_TOKEN` 等（`advanced-plugin.md:154-172`）。

## 风险、边界与改进建议
### 风险与边界
1. `.mcp.json` 包装格式潜在歧义
- 本示例使用 `{ "mcpServers": {...} }` 包装（`advanced-plugin.md:147-175`），而 `mcp-integration` 的推荐 `.mcp.json` 示例是“顶层直接 server map”（`mcp-integration/SKILL.md:27-37`）。若运行时只接受其中一种格式，存在接入失败风险。

2. agent frontmatter 版本漂移
- 示例 agent 仍用 `description + capabilities`（`advanced-plugin.md:228-236`），与 `agent-development` 要求的 `name/model/color` 不一致，迁移时可能触发校验错误。

3. hook 失败传播范围大
- `Stop` 同时串联多个 command hook（`advanced-plugin.md:672-679`），任一脚本失败都可能阻断结束，需明确“硬失败/软失败”策略。

4. 外部依赖面广导致可用性波动
- 依赖多种外部系统与凭证，任何单点（网络、权限、token 过期）都可能引起链路中断。

5. 文档示例与真实实现边界
- 文件是“示例文档”而非可直接执行的插件仓库；其中 `servers/`、`lib/`、`hooks/scripts/` 多为结构展示，实际落地需补齐真实代码与测试。

### 改进建议
1. 统一并显式声明 `.mcp.json` 格式
- 在示例中注明支持格式（顶层 map 或 `mcpServers` 包装），并给出与 `mcp-integration` 一致的单一推荐，避免歧义。

2. 升级 agent 示例到当前规范
- 增加 `name/model/color`，必要时保留 `capabilities` 作为补充字段，并在文档中标注“兼容旧版”。

3. 将 Stop hooks 分级
- 把“阻断发布”的硬门禁与“通知类/审计类”软门禁拆分，减少误阻断。

4. 加入可观测性与降级策略
- 对每个外部调用统一日志格式与重试策略（例如 `lib/utils/retry.js` 约定化）；对非关键集成支持降级继续。

5. 补充验证与回归脚本
- 增加用于校验 manifest/hooks/mcp schema 的自动化测试与 CI 校验，避免示例与规范继续漂移。
