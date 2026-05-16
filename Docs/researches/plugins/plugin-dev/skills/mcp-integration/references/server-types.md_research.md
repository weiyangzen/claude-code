# plugins/plugin-dev/skills/mcp-integration/references/server-types.md 研究

## 场景与职责

`server-types.md` 是 MCP 集成中的“传输与部署决策参考”，目标是帮助插件作者在 `stdio / sse / http / ws` 四种 server 类型中做正确选择，并把选择落地为可运行的 `.mcp.json`/`mcpServers` 配置。

在整体体系中的职责：

1. 作为 `mcp-integration/SKILL.md` 的深度参考，承接主技能里“先选类型再配置”的流程（`plugins/plugin-dev/skills/mcp-integration/SKILL.md:65-166,521-523`）。
2. 在 `create-plugin` 的 MCP 实施阶段，为“server type 选择 + command/url 字段完整性 + env vars”提供事实依据（`plugins/plugin-dev/commands/create-plugin.md:210-218`）。
3. 与 `plugin-validator` 的 MCP 校验规则直接对齐：stdio 需要 `command`，网络型需要 `url`，并强调可移植与安全约束（`plugins/plugin-dev/agents/plugin-validator.md:116-134`）。

该文件覆盖的是“配置语义 +生命周期 +最佳实践”，并不包含实际 MCP client/server 代码实现。

## 功能点目的

### 1) 定义四类 server 的配置模型与运行语义

文档按类型拆分配置格式、生命周期、用例：

1. `stdio`：本地子进程 + stdin/stdout JSON-RPC，适合本地工具和自建 server（`plugins/plugin-dev/skills/mcp-integration/references/server-types.md:5-99`）。
2. `sse`：HTTP 建连 + 事件流 + POST 调用，适合托管服务和 OAuth 场景（`plugins/plugin-dev/skills/mcp-integration/references/server-types.md:101-203`）。
3. `http`：无状态请求响应模型，适合 REST 风格后端（`plugins/plugin-dev/skills/mcp-integration/references/server-types.md:204-300`）。
4. `ws`：双向长连接，适合实时低延迟（`plugins/plugin-dev/skills/mcp-integration/references/server-types.md:302-370`）。

### 2) 提供决策矩阵与迁移路径

通过 comparison matrix、选择指南和迁移示例（stdio->SSE、HTTP->WS），降低早期错误选型带来的重构成本（`plugins/plugin-dev/skills/mcp-integration/references/server-types.md:371-457`）。

### 3) 提供安全与可运维基线

文档对网络协议（HTTPS/WSS）、token 管理、重试和超时做了最低要求，目的是让示例从“可运行”升级到“可上线”（`plugins/plugin-dev/skills/mcp-integration/references/server-types.md:281-287,504-527`）。

### 4) 支持多 server 与环境切换

“Multiple Servers / Conditional Configuration”章节给出同一插件接多种服务和 dev/prod 切换模式，减少在 plugin 侧写条件逻辑（`plugins/plugin-dev/skills/mcp-integration/references/server-types.md:458-503`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 关键流程

1. 插件加载时读取 `plugin.json` 的 `mcpServers` 字段或默认 `./.mcp.json`（`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:298-309,354-367`）。
2. 运行时按 server 条目分发：
   - 无 `type`（或本地配置）走 stdio：spawn 本地进程并维持 session 生命周期（`plugins/plugin-dev/skills/mcp-integration/references/server-types.md:38-44`）。
   - `type: sse/http/ws` 走网络连接路径：建连、握手、调用、重连/重试（`plugins/plugin-dev/skills/mcp-integration/references/server-types.md:133-140,237-243,334-341`）。
3. tool 发现后注册为 `mcp__plugin_<plugin>_<server>__<tool>`，进入 command/agent 调用面（`plugins/plugin-dev/skills/mcp-integration/SKILL.md:190-205`; `plugins/plugin-dev/skills/mcp-integration/references/tool-usage.md:7-15`）。
4. 开发期通过 `/mcp` 与 `claude --debug` 验证连通、工具可见性与调用日志（`plugins/plugin-dev/skills/mcp-integration/SKILL.md:427-431,447-455`）。

### B. 关键数据结构

1. `stdio` 结构：
   - `{"command": string, "args"?: string[], "env"?: object}`（`plugins/plugin-dev/skills/mcp-integration/references/server-types.md:13-36`）。
2. 网络型通用结构：
   - `{"type": "sse|http|ws", "url": string, "headers"?: object}`（`plugins/plugin-dev/skills/mcp-integration/references/server-types.md:109-131,212-235,310-332`）。
3. 条件切换结构：
   - URL/token 通过环境变量展开，支持 dev/prod 复用（`plugins/plugin-dev/skills/mcp-integration/references/server-types.md:486-503`）。
4. 多 server 聚合结构：
   - 单个 `.mcp.json` 中并列多条 server 定义（`plugins/plugin-dev/skills/mcp-integration/references/server-types.md:464-482`）。

### C. 协议与命令

1. 协议面：
   - stdio、ws 强调 JSON-RPC 双向消息语义；
   - sse 使用事件流 + HTTP 请求组合；
   - http 使用请求/响应无状态语义（`plugins/plugin-dev/skills/mcp-integration/references/server-types.md:41-42,137-139,239-242,338-339`）。
2. 运维命令：
   - `/mcp`：检查 server/tool 注册与 schema；
   - `claude --debug`：定位建连、认证、调用失败（`plugins/plugin-dev/skills/mcp-integration/SKILL.md:447-469`）。

### D. 与示例配置的映射

1. `examples/stdio-server.json` 展示本地命令、args、env（`plugins/plugin-dev/skills/mcp-integration/examples/stdio-server.json:3-25`）。
2. `examples/sse-server.json` 展示 hosted SSE + headers 扩展（`plugins/plugin-dev/skills/mcp-integration/examples/sse-server.json:3-18`）。
3. `examples/http-server.json` 展示 token header 与版本头（`plugins/plugin-dev/skills/mcp-integration/examples/http-server.json:3-19`）。

## 关键代码路径与文件引用

核心研究对象：

1. `plugins/plugin-dev/skills/mcp-integration/references/server-types.md:1-536`

直接调用方/入口：

1. `plugins/plugin-dev/skills/mcp-integration/SKILL.md:65-166,323-341,479-487,521-523`
2. `plugins/plugin-dev/README.md:80-90,244-246,319-321,344-349`
3. `plugins/plugin-dev/commands/create-plugin.md:162,210-218`

关键上下文依赖：

1. `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:298-330,354-367`
   - 定义 `mcpServers` 默认路径、内联写法、加载顺序。
2. `plugins/plugin-dev/agents/plugin-validator.md:116-123,130-134`
   - 提供 server 类型字段校验与安全检查。
3. `plugins/plugin-dev/skills/mcp-integration/references/authentication.md:13-21,80-139,227-258`
   - 补全 server 类型上的认证实现细节。
4. `plugins/plugin-dev/skills/mcp-integration/references/tool-usage.md:29-41,44-85,410-419`
   - 衔接 server 可用后如何在命令与 agent 调用。

测试/样例关联：

1. `plugins/plugin-dev/skills/mcp-integration/examples/stdio-server.json:1-26`
2. `plugins/plugin-dev/skills/mcp-integration/examples/sse-server.json:1-19`
3. `plugins/plugin-dev/skills/mcp-integration/examples/http-server.json:1-20`
4. `plugins/plugin-dev/skills/command-development/references/testing-strategies.md:328-339`

## 依赖与外部交互

### 内部依赖

1. 依赖 `plugin-structure` 对 `mcpServers` 路径解析与组件加载顺序的定义（`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:300-367`）。
2. 依赖 `mcp-integration/SKILL.md` 在实施流程里触发“类型选择 -> 配置 -> 测试”链路（`plugins/plugin-dev/skills/mcp-integration/SKILL.md:540-552`）。
3. 依赖 `tool-usage` 与 command frontmatter 体系完成权限限制和工具调用（`plugins/plugin-dev/skills/mcp-integration/references/tool-usage.md:44-85`; `plugins/plugin-dev/skills/command-development/references/frontmatter-reference.md:60-128`）。

### 外部交互

1. stdio：与本地可执行文件/解释器（`node`/`python`/`npx`）交互。
2. sse/http/ws：与远端 MCP endpoint、TLS 证书、网络策略交互。
3. 认证：与 OAuth 服务、API token 系统交互（由 `authentication.md` 细化）。
4. 调试：与 Claude Code 运行时命令接口（`/mcp`、`claude --debug`）交互。

### 配置、测试、脚本依赖现状

1. 本参考文件本身无可执行脚本；示例均在 `examples/`，真实测试依赖手工命令验证。
2. 跨目录存在验证规则文档（`plugin-validator`、testing-strategies），但缺专门的 MCP 配置 lint 脚本。

## 风险、边界与改进建议

### 风险 1（高）：`.mcp.json` 结构示例在跨文档存在歧义

- 现状：本目录及 SKILL 多处展示“顶层直接是 server map”；`advanced-plugin` 示例展示 `{ "mcpServers": { ... } }` 包装（`plugins/plugin-dev/skills/plugin-structure/examples/advanced-plugin.md:143-175`）。
- 影响：用户复制示例时可能不确定 canonical 形态。
- 建议：在 `server-types.md` 加“标准结构与兼容结构”对照，明确推荐写法。

### 风险 2（中）：安全建议与条件配置示例存在局部冲突

- 现状：文档强调始终 HTTPS/WSS（`plugins/plugin-dev/skills/mcp-integration/references/server-types.md:184-185,515-518`），但 dev 示例给出 `http://localhost`（`plugins/plugin-dev/skills/mcp-integration/references/server-types.md:500-502`）。
- 影响：新手可能把开发态例外带到生产。
- 建议：为 localhost 场景增加“仅本地开发”强提醒并给生产替代配置。

### 风险 3（中）：HTTP 流程描述较抽象，缺少 MCP 协议细节映射

- 现状：描述为“GET 发现 + POST 调用 + JSON 响应”（`plugins/plugin-dev/skills/mcp-integration/references/server-types.md:237-243`），未说明与 MCP 协议层消息格式的对应关系。
- 影响：实现者容易把普通 REST 端点误当 MCP 端点。
- 建议：补充“最小 MCP over HTTP 交互样例”和失败响应样例。

### 风险 4（中）：WebSocket 章节未给出重连与消息幂等策略

- 现状：提到 heartbeat/reconnect/buffer，但未给重试窗口、消息去重、顺序保障策略（`plugins/plugin-dev/skills/mcp-integration/references/server-types.md:334-369`）。
- 影响：实时场景容易在断连恢复后出现重复执行。
- 建议：增加消息 ID、ack、重放窗口建议模板。

### 风险 5（中）：示例 JSON 含 `_comment` 字段，可能误导直接复制

- 现状：`examples/*.json` 均带顶层 `_comment`（`plugins/plugin-dev/skills/mcp-integration/examples/stdio-server.json:2`; `.../sse-server.json:2`; `.../http-server.json:2`）。
- 影响：若直接作为 `.mcp.json` 使用，可能被当作 server 名解析。
- 建议：将注释迁移到同名 `.md` 说明文件，示例 JSON 保持可直接执行。

### 边界

1. 本文件不实现任何连接管理代码，只定义类型模型与实践建议。
2. 认证细节以 `authentication.md` 为准；工具调用行为以 `tool-usage.md` 为准。
3. 实际连接稳定性仍取决于外部服务 SLA、网络条件和 Claude Code runtime 行为。
