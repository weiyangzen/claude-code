# FILE 研究：plugins/plugin-dev/skills/mcp-integration/examples/sse-server.json

## 场景与职责

`sse-server.json` 是 MCP 集成技能中面向“托管 MCP 服务”的示例配置，职责是说明如何用 `type: "sse"` 将云端服务接入插件。

它在上下文里的角色：

1. 上游调用方
- `plugins/plugin-dev/skills/mcp-integration/SKILL.md:95-119,529-531` 将 SSE 定位为 hosted/OAuth 场景，并在示例列表中点名本文件。
- `plugins/plugin-dev/README.md:78-93,286-291,341-349` 将 MCP 三种配置示例作为开发入口。
- `plugins/plugin-dev/commands/create-plugin.md:210-219` 规定 MCP 阶段要创建 `.mcp.json` 并处理认证变量。

2. 下游被调用方
- Claude Code 在插件加载后按 SSE 生命周期建立连接、发现工具并重连（语义来自 `server-types` 文档）。
- `plugin-validator` 在质量检查阶段核对 URL 与类型字段（`plugins/plugin-dev/agents/plugin-validator.md:116-123`）。

结论：该文件定位是“云端 SSE MCP 连接模板”，不承担具体 OAuth 实现或工具业务逻辑。

## 功能点目的

1. 提供标准 SSE 配置骨架
- 每个 server 对象显式给出 `type: "sse"` 和 `url`（`plugins/plugin-dev/skills/mcp-integration/examples/sse-server.json:3-18`）。
- 与类型参考中的 SSE 最小结构一致（`plugins/plugin-dev/skills/mcp-integration/references/server-types.md:109-131`）。

2. 覆盖“官方服务 + 自定义服务”两类入口
- `asana` 与 `github`：只给 `type + url`，强调 hosted service 接入最简形态（`sse-server.json:3-10`）。
- `custom-service`：在最简形态上叠加 headers，展示业务版本、租户或客户端标识注入（`sse-server.json:11-17`）。

3. 演示认证分层
- OAuth 友好服务可以在不显式 token header 的情况下走自动授权流（`SKILL.md:115-119,243-255`；`authentication.md:13-34`）。
- 自定义 header 可补充平台外认证字段（`authentication.md:122-139`）。

4. 支撑工具发现与命令授权
- SSE 服务接入成功后，工具会进入 `mcp__plugin_<plugin>_<server>__<tool>` 命名空间，供命令 `allowed-tools` 或 Agent 调用（`tool-usage.md:7-15,44-74,120-156`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 关键流程

1. 以本文件为模板编写 `.mcp.json`（或内联到 manifest 的 `mcpServers`）。
2. 插件加载后，Claude Code 对每个 SSE server 发起 HTTPS 连接并完成 MCP 握手（`server-types.md:133-139`）。
3. 首次调用需要授权的工具时，可能触发 OAuth 浏览器流程并缓存 token（`authentication.md:13-21,57-64`）。
4. 连接建立后完成工具发现，工具可在 `/mcp` 中查看（`tool-usage.md:31-41`）。
5. 命令层按 `allowed-tools` 精确授权，避免通配越权（`SKILL.md:204-223`）。

### 数据结构

1. 顶层结构
- server map：`asana`、`github`、`custom-service`（`sse-server.json:3,7,11`）。
- 顶层含 `_comment` 说明字段（`sse-server.json:2`）。

2. SSE server 对象结构
- 必需：`type: "sse"`、`url: string`
- 可选：`headers: Record<string,string>`

3. Header 示例含义
- `X-API-Version`：服务版本协商。
- `X-Client-ID`：从 `${CLIENT_ID}` 注入客户端标识（`sse-server.json:14-16`）。

### 协议与命令

1. 协议特征
- SSE 连接生命周期为“初始化 -> 握手 -> 事件流 -> 请求调用 -> 断线重连”（`server-types.md:133-140`）。
- 推荐 HTTPS，不建议 HTTP（`SKILL.md:344-351`；`server-types.md:184-185`）。

2. 调试命令
- `/mcp`：查看 server 与工具注册结果（`tool-usage.md:31-41`）。
- `claude --debug`：定位握手失败、OAuth 失败、重连问题（`SKILL.md:447-455`；`authentication.md:384-394`）。

3. 测试流程
- `SKILL.md` 与 `tool-usage.md` 均给出本地验证顺序：配置 -> 安装 -> `/mcp` -> 调用 -> debug（`SKILL.md:425-431`；`tool-usage.md:410-419`）。

## 关键代码路径与文件引用

1. 目标文件
- `plugins/plugin-dev/skills/mcp-integration/examples/sse-server.json:1-19`

2. 直接规范来源
- `plugins/plugin-dev/skills/mcp-integration/SKILL.md:95-119,243-255,342-364,527-531`
- `plugins/plugin-dev/skills/mcp-integration/references/server-types.md:101-140,141-170,182-203`
- `plugins/plugin-dev/skills/mcp-integration/references/authentication.md:13-34,122-139,348-403`
- `plugins/plugin-dev/skills/mcp-integration/references/tool-usage.md:7-15,29-41,44-85`

3. 调用方与治理链路
- `plugins/plugin-dev/README.md:78-93,286-291,341-351`
- `plugins/plugin-dev/commands/create-plugin.md:210-219,298-299,315-317`
- `plugins/plugin-dev/agents/plugin-validator.md:116-123,130-133`
- `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:298-309,354-367`

## 依赖与外部交互

1. 外部网络依赖
- 强依赖可访问的 `https://.../sse` 端点与合法证书。
- 连接稳定性受网络、代理、防火墙、服务限流影响。

2. 认证依赖
- OAuth 型服务依赖浏览器授权与 token 刷新机制（由 Claude Code runtime 管理）。
- header 型服务依赖用户环境变量 `${CLIENT_ID}` 等注入。

3. 运行时交互
- SSE 负责服务端到客户端事件流；工具调用通常走 HTTP 请求通道（`server-types.md:138-139`）。
- 断开时依赖自动重连语义（同文档定义）。

4. 与脚本/测试关系
- 本文件无脚本执行逻辑。
- 校验依赖 plugin-validator（结构）和 `/mcp` + `claude --debug`（运行态）。

## 风险、边界与改进建议

1. 风险：`_comment` 字段可能干扰严格解析器
- 证据：`sse-server.json:2`。
- 建议：示例 JSON 去除 `_comment`，改写入说明文档。

2. 风险：示例域名是教学占位，不保证真实可用
- 证据：`https://mcp.github.com/sse`、`https://mcp.example.com/sse`（`sse-server.json:9,13`）。
- 影响：用户复制后可能立即遇到 DNS/401/404。
- 建议：在示例旁标注“占位 URL，需替换为真实 endpoint”。

3. 风险：认证方式表达可能引发误解
- 现状：`asana/github` 无 headers，`custom-service` 有 headers。
- 影响：用户可能误以为所有 SSE 都免认证或都必须自带 headers。
- 建议：加“OAuth 自动流 vs 自定义 headers”的显式对照注释。

4. 风险：缺少连接健壮性配置示例
- 影响：真实环境下遇到网络抖动、限流、短期不可用时，用户不知道如何处理。
- 建议：在 `references/server-types.md` 或示例旁新增“重试/退避/超时”建议模板。

5. 风险：Header 版本号静态化
- 证据：`X-API-Version: v1`（`sse-server.json:15`）。
- 影响：服务升级后可能失配。
- 建议：将版本放入环境变量并在 README 记录兼容范围。

边界说明：本文件仅定义“如何连接 SSE MCP 服务”；OAuth scope、工具 schema、权限边界由远端 MCP server 与 runtime 决定，不由本文件本身约束。
