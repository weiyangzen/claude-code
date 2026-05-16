# plugins/plugin-dev/skills/mcp-integration 目录研究（DIR）

## 场景与职责

`plugins/plugin-dev/skills/mcp-integration` 是 `plugin-dev` 工具包中专门负责 MCP（Model Context Protocol）接入的方法论技能目录，定位是“把外部服务能力（数据库/API/SaaS）安全地接入 Claude Code 插件”。

该目录承担三类职责：

1. 触发与路由职责（何时加载这个 skill）
- `SKILL.md` frontmatter 明确触发语义：当用户提到“add MCP server / integrate MCP / configure .mcp.json / SSE/stdio/HTTP/WebSocket”等语句时应加载该 skill：`plugins/plugin-dev/skills/mcp-integration/SKILL.md:2-4`。
- `plugin-dev` 主 README 将其列为 7 个核心技能之一，并明确“Integrate Services”阶段由该 skill 负责：`plugins/plugin-dev/README.md:7-17`, `244-246`。
- `/plugin-dev:create-plugin` 在组件实现 Phase 5 中显式要求“Load mcp-integration skill”：`plugins/plugin-dev/commands/create-plugin.md:157-163`, `210-218`。

2. 规范职责（如何配置与使用 MCP）
- 规范配置入口：`.mcp.json`（推荐）或 `plugin.json#mcpServers`（内联）：`plugins/plugin-dev/skills/mcp-integration/SKILL.md:19-59`。
- 规范服务类型：`stdio/sse/http/ws`：`plugins/plugin-dev/skills/mcp-integration/SKILL.md:65-166`。
- 规范工具命名与命令使用：`mcp__plugin_<plugin>_<server>__<tool>` 与 `allowed-tools` 预授权：`plugins/plugin-dev/skills/mcp-integration/SKILL.md:190-223`。

3. 运行与调试职责（接入后如何验证）
- 生命周期说明：插件加载 -> 解析 MCP 配置 -> 建连/起进程 -> 工具注册：`plugins/plugin-dev/skills/mcp-integration/SKILL.md:231-237`。
- 观测入口：`/mcp` 查看 server/tool，`claude --debug` 诊断连接与认证：`plugins/plugin-dev/skills/mcp-integration/SKILL.md:238-239`, `445-455`。
- `plugin-validator` 把 MCP 作为插件质量检查项（校验 `.mcp.json` 或 manifest 的 `mcpServers`）：`plugins/plugin-dev/agents/plugin-validator.md:116-123`。

目录内文件职责分工：
- `SKILL.md`：核心实践与工作流。
- `references/server-types.md`：传输层与连接生命周期深度说明。
- `references/authentication.md`：OAuth/Token/headersHelper 等认证模式。
- `references/tool-usage.md`：命令/agent 中消费 MCP 工具的模式。
- `examples/*.json`：stdio/SSE/HTTP 示例配置。

## 功能点目的

### 1) 配置入口选择（`.mcp.json` vs `plugin.json#mcpServers`）
目的：平衡复杂度与可维护性。
- 推荐独立 `.mcp.json` 以支持多 server 与配置解耦：`plugins/plugin-dev/skills/mcp-integration/SKILL.md:23-43`。
- 允许在 `plugin.json` 内联 `mcpServers` 以减少小插件文件数量：`plugins/plugin-dev/skills/mcp-integration/SKILL.md:44-63`。
- `plugin-structure` 侧对 `mcpServers` 字段也给出同样双形态约定（字符串路径或对象）：`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:298-330`。

### 2) 服务类型分层
目的：让插件按场景选择最合适的传输与认证模型。
- `stdio`：本地进程、最低延迟、本地工具与自定义 server：`plugins/plugin-dev/skills/mcp-integration/references/server-types.md:5-44`。
- `sse`：托管服务、流式事件、OAuth 友好：`plugins/plugin-dev/skills/mcp-integration/references/server-types.md:101-157`。
- `http`：无状态 REST 交互，适合标准 API：`plugins/plugin-dev/skills/mcp-integration/references/server-types.md:204-243`。
- `ws`：实时双向、低延迟推送：`plugins/plugin-dev/skills/mcp-integration/references/server-types.md:302-341`。

### 3) 工具暴露与调用约束
目的：把“外部能力”转换为“可治理的 Claude 工具调用”。
- 命名约定统一：`mcp__plugin_<plugin>_<server>__<tool>`：`plugins/plugin-dev/skills/mcp-integration/SKILL.md:194-201`, `plugins/plugin-dev/skills/mcp-integration/references/tool-usage.md:11-27`。
- 在 command frontmatter 中预授权具体工具，避免过宽权限：`plugins/plugin-dev/skills/mcp-integration/SKILL.md:204-223`, `plugins/plugin-dev/skills/mcp-integration/references/tool-usage.md:44-85`。
- agent 场景可做更自主的多步工具编排：`plugins/plugin-dev/skills/mcp-integration/references/tool-usage.md:120-156`。

### 4) 认证与安全策略
目的：在“可用性”与“安全性”之间达成默认安全。
- OAuth 自动流：首次触发时授权、后续刷新由 Claude Code 管理：`plugins/plugin-dev/skills/mcp-integration/references/authentication.md:13-21`, `57-64`。
- Token/API Key/自定义 header 通过环境变量注入，避免硬编码：`plugins/plugin-dev/skills/mcp-integration/references/authentication.md:80-139`, `267-309`。
- 动态 header（`headersHelper`）覆盖短时 token、HMAC 等高级认证：`plugins/plugin-dev/skills/mcp-integration/references/authentication.md:227-258`, `500-518`。

### 5) 集成测试与故障诊断
目的：让接入问题可快速定位。
- 本地测试路径：配置 -> 安装插件 -> `/mcp` 校验 -> 命令调用 -> `claude --debug`：`plugins/plugin-dev/skills/mcp-integration/references/tool-usage.md:410-419`。
- 典型故障排查：连接失败、认证失败、参数不匹配、性能退化：`plugins/plugin-dev/skills/mcp-integration/references/tool-usage.md:502-527`, `plugins/plugin-dev/skills/mcp-integration/references/authentication.md:348-402`。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 端到端流程（从 skill 到运行时）

1. 规划阶段：`/plugin-dev:create-plugin` 进入 Phase 5，装载 mcp-integration skill：`plugins/plugin-dev/commands/create-plugin.md:153-163`, `210-218`。
2. 配置阶段：开发者在插件根写 `.mcp.json` 或在 manifest 写 `mcpServers`：`plugins/plugin-dev/skills/mcp-integration/SKILL.md:19-59`。
3. 发现阶段：Claude Code 根据插件结构自动加载 `.mcp.json` 或 manifest 中指定项：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:343-349`, `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:354-367`。
4. 建连阶段：
- stdio：spawn 子进程并以 stdin/stdout 走 JSON-RPC：`plugins/plugin-dev/skills/mcp-integration/references/server-types.md:38-43`。
- sse/http/ws：建立网络连接并按类型完成握手/请求：`plugins/plugin-dev/skills/mcp-integration/references/server-types.md:133-140`, `237-243`, `334-340`。
5. 暴露阶段：工具按统一前缀注册，命令/agent 开始可调用：`plugins/plugin-dev/skills/mcp-integration/SKILL.md:231-237`, `190-201`。
6. 验证阶段：`/mcp` 查看可见性，`claude --debug` 查日志，必要时用 plugin-validator 进行结构校验：`plugins/plugin-dev/skills/mcp-integration/SKILL.md:238-239`, `445-455`; `plugins/plugin-dev/agents/plugin-validator.md:116-123`。

### B. 关键数据结构

1. server 配置对象（按类型分支）
- stdio 最小结构：`{ "command": string, "args"?: string[], "env"?: object }`（示例见 `SKILL.md` 与 `server-types.md`）：`plugins/plugin-dev/skills/mcp-integration/SKILL.md:71-82`, `plugins/plugin-dev/skills/mcp-integration/references/server-types.md:13-36`。
- 网络型 server 最小结构：`{ "type": "sse|http|ws", "url": string, "headers"?: object }`：`plugins/plugin-dev/skills/mcp-integration/SKILL.md:99-159`。
- 高级认证扩展：`headersHelper` 指向脚本输出 JSON headers：`plugins/plugin-dev/skills/mcp-integration/references/authentication.md:233-240`。

2. 工具命名协议
- 固定格式：`mcp__plugin_<plugin-name>_<server-name>__<tool-name>`：`plugins/plugin-dev/skills/mcp-integration/references/tool-usage.md:13-15`。
- command 里通过 frontmatter `allowed-tools` 精确白名单：`plugins/plugin-dev/skills/mcp-integration/references/tool-usage.md:46-74`。

3. 环境变量展开协议
- 路径锚点：`${CLAUDE_PLUGIN_ROOT}`（可移植性核心）：`plugins/plugin-dev/skills/mcp-integration/SKILL.md:171-176`。
- 凭据注入：`${API_TOKEN}` / `${DB_URL}` 等来自用户 shell：`plugins/plugin-dev/skills/mcp-integration/SKILL.md:178-185`。

### C. 关键命令与协议交互

- `/mcp`：列 server、tool 名、schema，用于“可见性与命名校对”：`plugins/plugin-dev/skills/mcp-integration/references/tool-usage.md:29-40`。
- `claude --debug`：查看连接、认证、工具调用日志：`plugins/plugin-dev/skills/mcp-integration/SKILL.md:447-455`。
- `curl -H "Authorization: Bearer $API_TOKEN" .../health`：认证链路外部自检：`plugins/plugin-dev/skills/mcp-integration/references/authentication.md:395-402`。

### D. 与上下文依赖的耦合点

- 与 `plugin-structure`：共享 `.mcp.json`/`mcpServers` 入口与 auto-discovery 机制：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:233-255`, `343-349`。
- 与 `command-development`：command 的 `allowed-tools` 设计直接消费 MCP 命名协议；其测试策略也单列了“Command + MCP Integration”场景：`plugins/plugin-dev/skills/command-development/references/testing-strategies.md:328-339`。
- 与 `skill-development`：mcp-integration 被当作“高质量参考技能”示例：`plugins/plugin-dev/skills/skill-development/SKILL.md:608-613`。

## 关键代码路径与文件引用

### 目标目录核心文件
- `plugins/plugin-dev/skills/mcp-integration/SKILL.md`
- `plugins/plugin-dev/skills/mcp-integration/references/server-types.md`
- `plugins/plugin-dev/skills/mcp-integration/references/authentication.md`
- `plugins/plugin-dev/skills/mcp-integration/references/tool-usage.md`
- `plugins/plugin-dev/skills/mcp-integration/examples/stdio-server.json`
- `plugins/plugin-dev/skills/mcp-integration/examples/sse-server.json`
- `plugins/plugin-dev/skills/mcp-integration/examples/http-server.json`

### 上游调用方（谁触发它）
- `plugins/plugin-dev/README.md:76-93`（技能定位与触发短语）
- `plugins/plugin-dev/commands/create-plugin.md:157-163`, `210-218`（Phase 5 MCP 实施步骤）

### 下游被调用方（它依赖谁来完成闭环）
- `plugins/plugin-dev/agents/plugin-validator.md:116-123`（MCP 配置校验）
- `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:298-330`, `354-367`（manifest 字段与加载顺序）
- `plugins/plugin-dev/skills/command-development/references/testing-strategies.md:328-339`（集成测试场景）

### 配置与运行时相关关键路径
- `.mcp.json`（推荐配置入口，路径由插件根解析）
- `plugin.json#mcpServers`（内联或外链配置入口）
- `/mcp` 命令（运行态观测）

## 依赖与外部交互

### 本地依赖
- 该目录自身不含可执行脚本；属于“文档 + 示例配置”资产。
- 验证动作主要依赖 Claude Code 运行时能力（`/mcp`、`claude --debug`）与 plugin-validator agent，而非目录内脚本：`plugins/plugin-dev/skills/mcp-integration/SKILL.md:425-431`, `445-455`; `plugins/plugin-dev/agents/plugin-validator.md:116-123`。

### 运行时外部交互
- 进程交互：stdio 通过子进程 stdin/stdout 承载 JSON-RPC：`plugins/plugin-dev/skills/mcp-integration/references/server-types.md:40-43`。
- 网络交互：SSE/HTTP/WS 访问远端 MCP endpoint（HTTPS/WSS）：`plugins/plugin-dev/skills/mcp-integration/references/server-types.md:184-185`, `283-287`, `352-356`。
- 认证交互：OAuth 浏览器授权、token header、动态 headers 脚本输出：`plugins/plugin-dev/skills/mcp-integration/references/authentication.md:13-21`, `86-97`, `227-258`。

### 外部文档依赖
- `SKILL.md` 中直接引用外部规范：MCP 官方站点、Claude Code MCP 文档、MCP SDK：`plugins/plugin-dev/skills/mcp-integration/SKILL.md:535-538`。

## 风险、边界与改进建议

### 风险 1（高）：`.mcp.json` 结构示例存在跨文档不一致

现象：
- mcp-integration 将 `.mcp.json` 示例写成“顶层直接是 server map”：`plugins/plugin-dev/skills/mcp-integration/SKILL.md:25-37`。
- `plugin-structure` 的 advanced 示例把 `.mcp.json` 写成 `{ "mcpServers": { ... } }` 包装结构：`plugins/plugin-dev/skills/plugin-structure/examples/advanced-plugin.md:147-175`。

影响：
- 开发者跨文档拷贝时可能无法确认运行时期望格式，造成加载失败或隐式兼容依赖。

建议：
1. 在 mcp-integration 增加“`.mcp.json` canonical schema”小节，明确支持形态与优先推荐。
2. 在 `plugin-structure/examples/advanced-plugin.md` 标注该包装结构是否为教学简化或真实可用格式。

### 风险 2（中）：示例 JSON 使用 `_comment`，可执行性边界不清

现象：
- 三个示例文件顶层均含 `_comment` 字段：
  - `plugins/plugin-dev/skills/mcp-integration/examples/stdio-server.json:2`
  - `plugins/plugin-dev/skills/mcp-integration/examples/sse-server.json:2`
  - `plugins/plugin-dev/skills/mcp-integration/examples/http-server.json:2`

影响：
- 若用户把示例原样作为 `.mcp.json`，运行时若严格把顶层 key 视作 server，将把 `_comment` 误当 server 定义。

建议：
1. 在示例文件名或文档注明“示例不可直接原样落盘”。
2. 改为 README 注释说明，去掉 JSON 内 `_comment` 字段，保证 copy-paste 可执行。

### 风险 3（中）：环境变量命名在示例与正文之间存在漂移

现象：
- `SKILL.md` 重点强调 `${CLAUDE_PLUGIN_ROOT}` 和用户环境变量：`plugins/plugin-dev/skills/mcp-integration/SKILL.md:167-189`。
- `stdio-server.json` 示例额外使用 `${CLAUDE_PROJECT_DIR}`：`plugins/plugin-dev/skills/mcp-integration/examples/stdio-server.json:5`。

影响：
- 新用户可能不确定该变量是否始终可用、在哪个运行阶段注入。

建议：
1. 在 `SKILL.md` 补充变量来源矩阵（插件根、项目根、用户 shell）。
2. 示例尽量统一到已在正文解释过的变量集合。

### 风险 4（中）：create-plugin 要求与 mcp skill 覆盖面不完全一致

现象：
- `create-plugin` 的 MCP步骤提到 `extensionToLanguage mapping if LSP`：`plugins/plugin-dev/commands/create-plugin.md:215-216`。
- mcp-integration 文档体系未展开该字段/场景。

影响：
- 工作流命令会向用户承诺一个在本技能文档中缺失的实现点。

建议：
1. 在 mcp-integration references 增加 LSP/MCP 映射专题，或
2. 在 create-plugin 中删除/弱化该要求，避免无文档兜底。

### 风险 5（中）：缺少目录内自动化校验脚本

现象：
- 相比 hook/agent/settings 技能，这个目录没有 `scripts/` 下的校验工具。

影响：
- 目前主要依赖手工 `/mcp`、`claude --debug` 与通用 plugin-validator；缺少“配置静态 lint + 示例可执行性验证”的快速反馈。

建议：
1. 增加 `scripts/validate-mcp-config.sh`（校验 type/command/url/headersHelper 等字段组合合法性）。
2. 增加最小 smoke 测试说明（例如对 HTTP health endpoint 的可选探测模板）。

### 边界说明

- 本目录是“技能知识资产”，不直接实现 MCP 客户端代码；其输出是文档规范和示例。
- 真实连接行为、token 存储、OAuth 生命周期由 Claude Code 运行时负责：`plugins/plugin-dev/skills/mcp-integration/references/authentication.md:13-21`, `57-64`。
- 因此该目录的质量上限主要受“文档一致性、示例可执行性、与其他 skill 的协议对齐”影响。
