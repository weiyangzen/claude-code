# plugins/plugin-dev/skills/mcp-integration/references 目录研究（DIR）

## 场景与职责

该目录是 `mcp-integration` 技能的“深度参考层”，在 plugin-dev 的 progressive disclosure 体系里负责承载核心 `SKILL.md` 之外的细节规范。

- 在技能内的职责定位：`SKILL.md` 在“Reference Files”章节显式把本目录 3 份文档作为二级知识入口（`plugins/plugin-dev/skills/mcp-integration/SKILL.md:521-523`）。
- 在插件工具包内的职责定位：`plugin-dev` README 把 MCP 能力拆成“核心 SKILL + 3 份 references + 3 份 examples”，其中 references 覆盖 server type、认证、tool 使用（`plugins/plugin-dev/README.md:88-93`）。
- 在工作流中的职责定位：`/plugin-dev:create-plugin` 的 Phase 5 要求先加载 `mcp-integration` 技能再实施 MCP 配置，references 是该技能深入实施时的规则来源（`plugins/plugin-dev/commands/create-plugin.md:157-163,210-218`）。

按文件划分的具体职责：

1. `server-types.md`：定义 4 类 MCP server（stdio/SSE/HTTP/ws）的配置模型、连接生命周期、迁移路径与安全要点（`plugins/plugin-dev/skills/mcp-integration/references/server-types.md:5-44,101-140,204-243,302-341,410-457,504-527`）。
2. `authentication.md`：定义 OAuth 自动流、Token/API Key、自定义 headers、`headersHelper` 动态认证，以及租户化和高级认证（mTLS/JWT/HMAC）处理方式（`plugins/plugin-dev/skills/mcp-integration/references/authentication.md:9-33,80-139,227-240,311-337,458-518`）。
3. `tool-usage.md`：定义 MCP 工具命名协议、command/agent 调用策略、参数与错误处理、性能模式和测试清单（`plugins/plugin-dev/skills/mcp-integration/references/tool-usage.md:7-15,42-85,120-156,210-273,313-357,410-446,502-538`）。

## 功能点目的

### 1) 服务类型决策支持

目的：让插件作者根据部署位置、交互模式、鉴权方式选择正确 transport。

- `server-types.md` 把 stdio/SSE/HTTP/ws 的生命周期、适用场景与迁移方案集中化，避免在单个 SKILL 中信息过载（`plugins/plugin-dev/skills/mcp-integration/references/server-types.md:38-44,133-140,237-243,334-341,371-409`）。

### 2) 认证策略标准化

目的：把“先能连上”升级为“可持续、安全、可维护的认证模型”。

- OAuth 自动流覆盖首次授权、token 刷新与故障排查（`plugins/plugin-dev/skills/mcp-integration/references/authentication.md:13-21,57-79`）。
- Token/API Key/自定义 Header 规范强调环境变量注入而非硬编码（`plugins/plugin-dev/skills/mcp-integration/references/authentication.md:86-120,141-171,267-309`）。
- `headersHelper` 机制用于短期 token、签名等动态凭据场景（`plugins/plugin-dev/skills/mcp-integration/references/authentication.md:227-258,482-518`）。

### 3) MCP 工具治理与最小权限

目的：让外部能力在 command/agent 中可控、可审计、可测试。

- 命名协议标准化：`mcp__plugin_<plugin-name>_<server-name>__<tool-name>`（`plugins/plugin-dev/skills/mcp-integration/references/tool-usage.md:13-15`）。
- command 场景通过 `allowed-tools` 精细白名单限制权限；wildcard 仅建议谨慎使用（`plugins/plugin-dev/skills/mcp-integration/references/tool-usage.md:44-85`）。
- 与 command-development 的 frontmatter 权限约束形成配套（`plugins/plugin-dev/skills/command-development/references/frontmatter-reference.md:60-67,124-128`）。

### 4) 运行期可观测性与验证闭环

目的：保证接入可验证、问题可定位。

- `/mcp` 用于 server/tool/schema 发现；`claude --debug` 用于连接与调用问题排障（`plugins/plugin-dev/skills/mcp-integration/references/tool-usage.md:29-41,414-419,512-519`）。
- 与测试文档联动存在“Command + MCP Integration”场景验证（`plugins/plugin-dev/skills/command-development/references/testing-strategies.md:328-339`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 关键流程

1. `SKILL.md` 触发后，开发者在插件根配置 `.mcp.json` 或在 manifest 中配置 `mcpServers`（`plugins/plugin-dev/skills/mcp-integration/SKILL.md:23-59`）。
2. Claude Code 启动时读取 plugin manifest 与默认路径，加载 `.mcp.json`/`mcpServers`（`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:298-309,354-367`）。
3. 按 server 类型建连：
   - stdio：spawn 子进程并经 stdin/stdout 传输 JSON-RPC（`plugins/plugin-dev/skills/mcp-integration/references/server-types.md:40-43`）。
   - SSE：SSE 流 + HTTP 请求、自动重连（`plugins/plugin-dev/skills/mcp-integration/references/server-types.md:135-140`）。
   - HTTP：无状态请求响应模型（`plugins/plugin-dev/skills/mcp-integration/references/server-types.md:237-243`）。
   - ws：WebSocket 持久双向通道 + 心跳/重连（`plugins/plugin-dev/skills/mcp-integration/references/server-types.md:336-341,352-357`）。
4. 工具注册后，command 通过 frontmatter 预授权调用，agent 根据任务自主编排调用（`plugins/plugin-dev/skills/mcp-integration/references/tool-usage.md:46-74,124-156`）。
5. 使用 `/mcp` 与 `claude --debug` 进行可见性与链路诊断（`plugins/plugin-dev/skills/mcp-integration/references/tool-usage.md:29-40,414-419`）。

### B. 关键数据结构

1. server 配置结构（按类型分支）：
- stdio: `{"command": string, "args"?: string[], "env"?: object}`（`plugins/plugin-dev/skills/mcp-integration/references/server-types.md:13-36`）。
- 网络型: `{"type": "sse|http|ws", "url": string, "headers"?: object}`（`plugins/plugin-dev/skills/mcp-integration/references/server-types.md:109-131,212-235,310-332`）。

2. 认证增强结构：
- `headersHelper` 指向脚本，脚本需输出 JSON header 对象（`plugins/plugin-dev/skills/mcp-integration/references/authentication.md:233-258`）。

3. 工具命名协议：
- `mcp__plugin_<plugin-name>_<server-name>__<tool-name>`，作为 `allowed-tools` 精确授权单位（`plugins/plugin-dev/skills/mcp-integration/references/tool-usage.md:13-15,46-74`）。

### C. 关键协议与命令

- 协议：stdio/WS 通道下使用 JSON-RPC；SSE/HTTP 走 HTTP 语义并附带 headers（`plugins/plugin-dev/skills/mcp-integration/references/server-types.md:41,137-139,239-242,338-339`）。
- 运行命令：
  - `/mcp`：查看 server 与工具清单（`plugins/plugin-dev/skills/mcp-integration/references/tool-usage.md:31-41`）。
  - `claude --debug`：抓连接/认证/调用日志（`plugins/plugin-dev/skills/mcp-integration/references/authentication.md:384-393`）。
  - `curl -H "Authorization: Bearer $API_TOKEN" .../health`：外部鉴权链路冒烟测试（`plugins/plugin-dev/skills/mcp-integration/references/authentication.md:395-402`）。

## 关键代码路径与文件引用

### 目标目录（核心研究对象）

1. `plugins/plugin-dev/skills/mcp-integration/references/server-types.md`
2. `plugins/plugin-dev/skills/mcp-integration/references/authentication.md`
3. `plugins/plugin-dev/skills/mcp-integration/references/tool-usage.md`

### 调用方（谁消费该目录）

1. `plugins/plugin-dev/skills/mcp-integration/SKILL.md:521-523`
- 直接把 3 份 references 暴露给技能使用流程。

2. `plugins/plugin-dev/README.md:88-93,344-349`
- 在 toolkit 总览与用例中把 references 作为 MCP 实操细节层。

3. `plugins/plugin-dev/commands/create-plugin.md:157-163,210-218`
- 通过“加载 mcp-integration skill”间接依赖 references 细则完成配置。

### 被调用方/上下文依赖（该目录依赖谁）

1. `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:298-330,354-367`
- 提供 `mcpServers` 字段语义、默认 `.mcp.json` 路径和加载顺序。

2. `plugins/plugin-dev/agents/plugin-validator.md:116-123`
- 提供 MCP 配置校验规则（stdio 需 `command`，网络型需 `url`，检查 `${CLAUDE_PLUGIN_ROOT}` 可移植性）。

3. `plugins/plugin-dev/skills/command-development/references/frontmatter-reference.md:60-67,124-128`
- 定义 command `allowed-tools` 权限约束，与 `tool-usage.md` 的 MCP 工具授权实践闭环。

4. `plugins/plugin-dev/skills/command-development/references/testing-strategies.md:328-339`
- 提供 command+MCP 联调测试场景。

5. `plugins/plugin-dev/skills/skill-development/SKILL.md:608-613`
- 把 mcp-integration 作为“综合 references 设计范例”进行横向复用。

## 依赖与外部交互

### 内部依赖

- 文档依赖：该目录纯文档资产，不含可执行脚本；依赖 `SKILL.md` 调度、plugin-structure 配置规范、command-development 权限语义、validator 校验语义形成闭环。
- 配置依赖：强依赖 `.mcp.json` / manifest `mcpServers` 的格式与加载规则（`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:300-309,361-367`）。

### 外部交互

1. 网络服务交互：
- SSE/HTTP/ws 指向外部 MCP endpoint（`plugins/plugin-dev/skills/mcp-integration/references/server-types.md:109-117,212-219,310-317`）。

2. 本地进程交互：
- stdio 模式以本地命令拉起服务并通过 stdin/stdout 通信（`plugins/plugin-dev/skills/mcp-integration/references/server-types.md:40-43`）。

3. 认证系统交互：
- OAuth 浏览器授权、token 刷新、环境变量注入、动态 header 生成脚本（`plugins/plugin-dev/skills/mcp-integration/references/authentication.md:13-21,57-64,99-103,233-258`）。

### 测试与脚本现状

- 目录内无 `scripts/`，测试流程主要靠命令化手工验证（`/mcp`、`claude --debug`、`curl`）和跨目录测试策略文档支持（`plugins/plugin-dev/skills/mcp-integration/references/tool-usage.md:410-419`; `plugins/plugin-dev/skills/command-development/references/testing-strategies.md:328-339`）。

## 风险、边界与改进建议

### 风险 1（高）：`.mcp.json` 结构示例在跨技能文档中不一致

- 现状：`mcp-integration` 示例通常使用“顶层直接 server map”（`plugins/plugin-dev/skills/mcp-integration/SKILL.md:27-36`），而 `plugin-structure` 的 advanced 示例展示了 `{ "mcpServers": { ... } }` 包装结构（`plugins/plugin-dev/skills/plugin-structure/examples/advanced-plugin.md:147-175`）。
- 影响：用户跨文档复制时可能不确定 canonical schema，增加配置失败概率。
- 建议：在 `references/server-types.md` 或 `SKILL.md` 增加“`.mcp.json` 标准形态 + 兼容形态”明确说明，并在 plugin-structure 示例处加注释。

### 风险 2（中）：示例 JSON 的 `_comment` 字段可能被误当 server 配置

- 现状：3 个示例文件都包含顶层 `_comment`（`plugins/plugin-dev/skills/mcp-integration/examples/stdio-server.json:2`, `.../sse-server.json:2`, `.../http-server.json:2`）。
- 影响：若用户直接复制为 `.mcp.json`，运行时可能把 `_comment` 当 server key 解析。
- 建议：改为 README 注释或文档说明，不在可执行 JSON 内保留 `_comment`。

### 风险 3（中）：安全建议与示例存在局部冲突

- 现状：`server-types.md` 强调“始终 HTTPS/WSS”（`plugins/plugin-dev/skills/mcp-integration/references/server-types.md:283-287,515-518`），但“Conditional Configuration”示例含 `http://localhost`（`plugins/plugin-dev/skills/mcp-integration/references/server-types.md:500-502`）。
- 影响：初学者可能将开发态例外误解为生产可接受策略。
- 建议：在示例旁补充“仅限本地开发，不可用于生产”标记。

### 风险 4（中）：权限模型描述存在“command 与 agent”粒度差异，缺统一决策树

- 现状：`tool-usage.md` 说明 command 需前置白名单而 agent 更自治（`plugins/plugin-dev/skills/mcp-integration/references/tool-usage.md:124-156`），但未给出“何时选 command、何时选 agent”的权限/审计决策模板。
- 影响：实现者可能在敏感操作里过度依赖 agent 自主工具调用。
- 建议：新增“权限决策矩阵”（敏感写操作优先 command 白名单、批量自动化可 agent + 审核步骤）。

### 风险 5（中）：认证与配置指南缺少目录内自动化 lint 工具

- 现状：本目录仅有文档与配置示例，无 `scripts/validate-mcp-config.sh` 之类的静态检查工具。
- 影响：JSON 结构、headersHelper 输出格式、变量缺失等问题只能运行时发现。
- 建议：新增脚本最少检查项：字段组合合法性、URL 协议约束、`${CLAUDE_PLUGIN_ROOT}` 路径存在性、headersHelper 输出 JSON 校验。

### 边界说明

1. 本目录不实现 MCP 客户端/服务端代码，只定义“如何配置与使用”的知识规范。
2. OAuth token 存储/刷新、tool 注册与执行属于 Claude Code runtime 范畴，不在本目录控制面内（`plugins/plugin-dev/skills/mcp-integration/references/authentication.md:57-64`）。
3. 因此该目录的主要质量杠杆是：文档一致性、示例可复制性、与 plugin-structure/command-development/validator 的契约对齐。
