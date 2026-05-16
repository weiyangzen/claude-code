# FILE 研究：plugins/plugin-dev/skills/mcp-integration/examples/http-server.json

## 场景与职责

`http-server.json` 是 MCP 集成技能中面向 REST 风格端点的示例配置，职责是展示 `type: "http"` 服务器在插件中的声明方式、header 认证模式与多服务并列写法。

它在链路里的位置：

1. 上游调用方
- `plugins/plugin-dev/skills/mcp-integration/SKILL.md:120-143,529-531` 将 HTTP 归类为 REST/token 场景，并把本文件列为示例。
- `plugins/plugin-dev/README.md:78-93,286-291` 将 MCP 示例纳入统一学习入口。
- `plugins/plugin-dev/commands/create-plugin.md:210-219` 在插件生成流程中要求产出 MCP 配置与环境变量文档。

2. 下游被调用方
- Claude Code runtime 根据 `type=http` + `url` 建立请求/响应式 MCP 调用路径（`server-types.md` 语义）。
- `plugin-validator` 在质量检查中验证 URL/类型与安全项（`plugins/plugin-dev/agents/plugin-validator.md:116-123,130-133`）。

结论：该文件是“HTTP MCP 接入模板”，关注配置层，不包含 API 实现层代码。

## 功能点目的

1. 给出 HTTP MCP 的最小字段集合
- 每个 server 含 `type: "http"`、`url`、`headers`（`plugins/plugin-dev/skills/mcp-integration/examples/http-server.json:3-19`）。
- 与参考文档的 HTTP 结构定义一致（`plugins/plugin-dev/skills/mcp-integration/references/server-types.md:212-235`）。

2. 演示 token header 注入规范
- `Authorization: Bearer ${API_TOKEN}` 展示环境变量替换，而非硬编码 token（`http-server.json:7,16`）。
- 对应认证文档的 Bearer 模式（`authentication.md:86-103`）。

3. 演示“同协议多 server 并存”
- `rest-api` 与 `internal-service` 两个 server 并列，说明同一插件可同时接入多个 HTTP MCP 端点（`http-server.json:3,12`）。
- 为后续命令层按 server 维度授权工具提供基础（`tool-usage.md:7-15,44-74`）。

4. 演示业务 header 扩展能力
- `Content-Type`、`X-API-Version`、`X-Service-Name` 用于版本协商、服务路由和请求语义传递（`http-server.json:8-9,17`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 关键流程

1. 开发者复制本模板到 `.mcp.json`（或内联到 manifest 的 `mcpServers`）。
2. 插件加载后，runtime 读取 `type=http` server 配置并对 `url` 发起 MCP 协议请求（`SKILL.md:231-237`；`server-types.md:237-243`）。
3. 首次调用工具前后，按 header 注入认证信息（Bearer token / 自定义头）。
4. 服务返回可用工具，Claude 以 `mcp__plugin_<plugin>_<server>__<tool>` 暴露给命令或 Agent（`SKILL.md:194-201`）。
5. 使用 `/mcp` 验证工具可见性，`claude --debug` 诊断 401/403/429/500 等异常（`tool-usage.md:31-41`；`server-types.md:291-300`）。

### 数据结构

1. 顶层结构
- server map：`rest-api`、`internal-service`（`http-server.json:3,12`）。
- 含 `_comment` 说明字段（`http-server.json:2`）。

2. HTTP server 对象结构
- 必需：`type: "http"`、`url: string`
- 常见可选：`headers: Record<string,string>`

3. 两个 server 的差异
- `rest-api`：强调通用 API 请求 header（`Content-Type` + `X-API-Version`）。
- `internal-service`：强调服务标识头（`X-Service-Name`）。

### 协议与命令

1. 协议语义
- HTTP 模式为无状态请求/响应模型，适合 REST 后端（`server-types.md:206-243`）。
- 安全建议为 HTTPS；不建议明文 HTTP（`SKILL.md:344-351`；`server-types.md:283-287`）。

2. 关键命令
- `/mcp`：确认 server/tool 已注册。
- `claude --debug`：观察鉴权失败、限流、响应超时等问题。
- `curl -H "Authorization: Bearer $API_TOKEN" .../health`：独立验证 token/endpoint（`authentication.md:395-402`）。

3. 测试链路
- `SKILL.md` 内 validation checklist 与本地测试步骤（`SKILL.md:425-441`）。
- 命令+MCP 联调场景（`testing-strategies.md:328-339`）。

## 关键代码路径与文件引用

1. 目标文件
- `plugins/plugin-dev/skills/mcp-integration/examples/http-server.json:1-20`

2. 直接规范来源
- `plugins/plugin-dev/skills/mcp-integration/SKILL.md:120-143,256-270,342-364,527-531`
- `plugins/plugin-dev/skills/mcp-integration/references/server-types.md:204-243,244-272,281-301`
- `plugins/plugin-dev/skills/mcp-integration/references/authentication.md:80-120,141-171,348-403`
- `plugins/plugin-dev/skills/mcp-integration/references/tool-usage.md:7-15,29-41,44-85`

3. 调用方与治理链路
- `plugins/plugin-dev/README.md:78-93,286-291,354-372`
- `plugins/plugin-dev/commands/create-plugin.md:210-219,290-299`
- `plugins/plugin-dev/agents/plugin-validator.md:116-123,130-134`
- `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:298-309,354-367`

## 依赖与外部交互

1. 网络依赖
- 依赖可达 `https://.../mcp` 端点、稳定网络与证书链。
- 受限流、网关策略、服务 SLA 影响明显。

2. 认证依赖
- 依赖用户环境变量 `API_TOKEN` 注入。
- 若 token 过期或权限不足，会在运行态体现为 401/403（`server-types.md:291-295`）。

3. 运行时交互
- 每次工具调用本质是一次独立 HTTP 交互（无状态）。
- 若多个 server 并存，工具命名按 server 前缀隔离，避免冲突。

4. 与测试/脚本关系
- 本文件不含执行脚本。
- 验证主要依赖 `/mcp`、`claude --debug` 和 plugin-validator；无目录内自动化测试资产。

## 风险、边界与改进建议

1. 风险：`_comment` 字段复制即用时存在歧义
- 证据：`http-server.json:2`。
- 建议：示例 JSON 改为纯配置，注释放文档。

2. 风险：示例 URL 为占位值，易被误用到生产
- 证据：`https://api.example.com/mcp`（`http-server.json:5,14`）。
- 建议：增加“必须替换 URL”提示，并给出最小 health-check 例子。

3. 风险：两个 server 复用同一 token 与同一 URL
- 影响：权限边界不清，难体现最小权限原则。
- 建议：示例可改为 `API_TOKEN_PUBLIC` / `API_TOKEN_INTERNAL` 分离，URL 也按服务拆分。

4. 风险：Header 版本硬编码导致后续兼容成本
- 证据：`X-API-Version: 2024-01-01`（`http-server.json:9`）。
- 建议：版本提取为环境变量并在 README 声明兼容策略。

5. 风险：缺少限流/超时/重试策略示例
- 影响：遇到 429 或慢响应时，用户没有直接可复制的稳态配置指导。
- 建议：在 `references/server-types.md` 增补 HTTP 稳健性章节（重试退避、超时、失败分级处理）。

边界说明：本文件只定义“客户端如何访问 HTTP MCP endpoint”，不定义服务端 tool schema、配额策略与数据一致性模型，这些由远端服务契约决定。
