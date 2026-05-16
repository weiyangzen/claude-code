# DIR 研究：plugins/plugin-dev/skills/mcp-integration/examples

## 场景与职责

`plugins/plugin-dev/skills/mcp-integration/examples` 是 `mcp-integration` skill 的示例配置层，提供 3 个可复制的 MCP server 配置模板（`stdio` / `sse` / `http`），用于把“抽象规范”落到“可填写参数的 JSON 形态”。

该目录在上下文中的职责边界：

1. 上游调用方（谁引导用户来到这些示例）
- `plugins/plugin-dev/skills/mcp-integration/SKILL.md:527-531`：在 Additional Resources 中将本目录列为 working examples。
- `plugins/plugin-dev/README.md:218-221,286-291,344-349`：在 Quick Start、Features、Use Cases 中把 MCP 示例作为实践入口。
- `plugins/plugin-dev/commands/create-plugin.md:210-218`：在 Phase 5 的 MCP 步骤要求开发者创建 `.mcp.json`，本目录示例是该动作的直接参考。

2. 下游被调用方（示例落地后由谁消费）
- Claude Code 插件加载流程对 `.mcp.json` 或 `plugin.json#mcpServers` 的解析与建连（`plugins/plugin-dev/skills/mcp-integration/SKILL.md:23-59,231-237`）。
- 调试与验证链路：`/mcp` 与 `claude --debug`（`plugins/plugin-dev/skills/mcp-integration/SKILL.md:238-239,445-455`）。
- 结构校验链路：`plugin-validator` 对 MCP 配置字段合法性检查（`plugins/plugin-dev/agents/plugin-validator.md:116-123`）。

3. 目录内职责（只做示例，不做执行）
- `stdio-server.json`：演示本地进程型 server（含 `command/args/env`）。
- `sse-server.json`：演示托管服务型 server（含 `type/url/headers`）。
- `http-server.json`：演示 REST MCP 端点（含 token header）。

结论：该目录是“配置样板库”，不是 MCP 客户端实现，也不是自动化测试脚本目录。

## 功能点目的

### 1) 给 `.mcp.json` 提供可直接改造的最小模板

3 个示例都采用“顶层 key = server 名称，value = server 配置对象”的结构，降低新手从零写配置的成本：
- `plugins/plugin-dev/skills/mcp-integration/examples/stdio-server.json:3,10,18`
- `plugins/plugin-dev/skills/mcp-integration/examples/sse-server.json:3,7,11`
- `plugins/plugin-dev/skills/mcp-integration/examples/http-server.json:3,12`

### 2) 把 server type 选择转成可对照的字段差异

示例分别覆盖：
- `stdio`：无 `type` 字段，核心是 `command + args + env`。
- `sse`：`type: "sse" + url (+ headers 可选)`。
- `http`：`type: "http" + url + headers`。

这与 `server-types` 参考文档中的类型定义保持一致（`plugins/plugin-dev/skills/mcp-integration/references/server-types.md:11-36,107-131,210-235`）。

### 3) 明确环境变量与可移植路径的使用方式

示例体现了两类变量来源：
- 插件路径锚点：`${CLAUDE_PLUGIN_ROOT}`（`stdio-server.json:11-12`）。
- 用户环境变量：`${DATABASE_URL}`、`${CUSTOM_API_KEY}`、`${CLIENT_ID}`、`${API_TOKEN}`（`stdio-server.json:14,22`；`sse-server.json:16`；`http-server.json:7,16`）。

与 `SKILL.md` 的环境变量展开原则一致（`plugins/plugin-dev/skills/mcp-integration/SKILL.md:167-189`）。

### 4) 为命令/agent 使用 MCP 工具提供前置条件

本目录不直接定义工具调用，但它是 `tool-usage` 文档中“先有 server，后有工具名”的前置资产。工具命名与 `/mcp` 发现路径在：
- `plugins/plugin-dev/skills/mcp-integration/references/tool-usage.md:7-15,29-40`。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 关键流程（从示例到可用工具）

1. 选型：按场景选 `stdio/sse/http` 示例文件作为起点。
2. 落盘：将示例裁剪后写入插件根 `.mcp.json`，或内联到 `plugin.json#mcpServers`（`SKILL.md:23-59`）。
3. 变量注入：用 `${CLAUDE_PLUGIN_ROOT}` 和用户环境变量替换硬编码路径/密钥（`SKILL.md:171-189`）。
4. 加载建连：Claude Code 解析配置并启动本地进程或建立网络连接（`SKILL.md:231-237`；`server-types.md:38-43,133-140,237-243`）。
5. 工具注册：server 暴露的工具以 `mcp__plugin_<plugin>_<server>__<tool>` 形式可见（`SKILL.md:194-201`）。
6. 观测验证：执行 `/mcp` 检查 server/tool 可见性，必要时 `claude --debug` 查建连与认证日志（`SKILL.md:238-239,445-455`）。

### B. 数据结构（示例 JSON 共同模型）

1. 顶层结构
- `Record<string, ServerConfig>`，即每个顶层 key 是 server 名。
- 当前 3 个示例都附带 `_comment` 顶层字段用于说明。

2. `stdio` 配置对象（示例：`filesystem/database/custom-tools`）
- 必需：`command: string`
- 可选：`args: string[]`、`env: Record<string, string>`
- 见：`plugins/plugin-dev/skills/mcp-integration/examples/stdio-server.json:3-25`

3. `sse/http` 配置对象
- 必需：`type: "sse" | "http"`、`url: string`
- 可选：`headers: Record<string, string>`
- 见：
  - `plugins/plugin-dev/skills/mcp-integration/examples/sse-server.json:3-18`
  - `plugins/plugin-dev/skills/mcp-integration/examples/http-server.json:3-19`

### C. 协议与命令

1. 运行协议
- `stdio`：子进程 + stdin/stdout JSON-RPC 通道（`server-types.md:40-43`）。
- `sse`：HTTP 建连 + SSE 事件流 + 请求调用（`server-types.md:135-140`）。
- `http`：请求/响应式 REST MCP 调用（`server-types.md:239-243`）。

2. 认证协议
- OAuth 自动流（主要在 SSE/HTTP 托管服务场景）（`authentication.md:13-21,57-64`）。
- Token/API Key 通过 headers 注入（`authentication.md:80-119`）。

3. 关键操作命令
- `/mcp`：查看 server 与工具清单（`tool-usage.md:31-40`）。
- `claude --debug`：排查连接、认证、工具调用失败（`SKILL.md:447-455`）。
- `curl -H "Authorization: Bearer $API_TOKEN" ...`：服务侧健康探测（`authentication.md:395-402`）。

### D. 本次对象与“测试/脚本”关系

- 目标目录 `examples/` 不包含 `scripts/` 或自动化测试文件；验证依赖外部流程文档与运行时命令。
- 可复用的测试建议主要在：
  - `plugins/plugin-dev/skills/mcp-integration/SKILL.md:423-441`
  - `plugins/plugin-dev/skills/mcp-integration/references/tool-usage.md:410-446`
  - `plugins/plugin-dev/skills/command-development/references/testing-strategies.md:328-339`

## 关键代码路径与文件引用

### 目标目录（核心研究对象）
- `plugins/plugin-dev/skills/mcp-integration/examples/stdio-server.json`
- `plugins/plugin-dev/skills/mcp-integration/examples/sse-server.json`
- `plugins/plugin-dev/skills/mcp-integration/examples/http-server.json`

### 调用方与流程入口
- `plugins/plugin-dev/skills/mcp-integration/SKILL.md:23-59,527-531,542-553`
- `plugins/plugin-dev/README.md:218-221,286-291,344-349`
- `plugins/plugin-dev/commands/create-plugin.md:210-218`

### 被调用方与约束文档
- `plugins/plugin-dev/skills/mcp-integration/references/server-types.md`
- `plugins/plugin-dev/skills/mcp-integration/references/authentication.md`
- `plugins/plugin-dev/skills/mcp-integration/references/tool-usage.md`
- `plugins/plugin-dev/agents/plugin-validator.md:116-123`
- `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:298-330,354-367`

### 研究流程相关文件
- `Docs/researches/blueprint_checklist.md:86`
- `.ops/generate_daily_research_todo.sh`

## 依赖与外部交互

### 1) 本地依赖

- 目录形态：纯 JSON 示例，无可执行代码。
- 运行时依赖来自 Claude Code MCP 能力与插件清单机制，而非本目录脚本。

### 2) 外部交互

1. 进程交互（stdio）
- 通过本地可执行命令启动 server（如 `npx`、`python`、自定义脚本）。

2. 网络交互（sse/http）
- 访问远端 `https://.../sse` 或 `https://.../mcp`。
- header 中会携带 token 或业务 header（示例：`Authorization`、`X-API-Version`）。

3. 凭据交互
- 依赖用户 shell 环境变量注入，不应在 JSON 中明文写死密钥。

### 3) 与配置/测试/脚本文档的耦合

- 配置：`SKILL.md` 与 `manifest-reference.md` 共同定义 `.mcp.json` / `mcpServers` 入口。
- 测试：`SKILL.md`、`tool-usage.md`、`testing-strategies.md` 提供手工验证路径。
- 脚本：本目录本身无脚本；若需动态认证，脚本能力在 `authentication.md` 的 `headersHelper` 模式中定义（`authentication.md:227-258`）。

## 风险、边界与改进建议

1. 风险：示例含 `_comment` 顶层键，复制到生产配置有歧义
- 现状：三个示例都包含 `_comment`（`stdio-server.json:2`、`sse-server.json:2`、`http-server.json:2`）。
- 风险：运行时若按“顶层键即 server”严格解析，`_comment` 可能被误识别为非法 server。
- 建议：移除 `_comment`，改为在同目录 `README` 或文档注释说明，保证 copy-paste 直接可用。

2. 风险：`.mcp.json` 形态在跨文档存在不一致认知
- 现状：本 skill 常用“顶层直接是 server map”；但部分文档示例出现 `{ "mcpServers": {...} }` 包装形态（`plugin-structure/examples/advanced-plugin.md:147-175`）。
- 风险：用户难以判断 canonical schema，可能导致加载失败或依赖隐式兼容。
- 建议：在 `mcp-integration` 增加“`.mcp.json` canonical schema”小节并给出唯一推荐写法。

3. 风险：示例覆盖了 `stdio/sse/http`，但未覆盖 `ws`
- 现状：`SKILL.md` 与 `server-types.md` 都说明支持 `ws`，examples 目录无 `ws-server.json`。
- 风险：实时场景用户需要跨文档手动拼装，降低上手效率。
- 建议：新增 `ws-server.json`，与现有 3 个示例保持同一注释与变量风格。

4. 风险：变量来源说明不足导致迁移问题
- 现状：`stdio` 示例使用 `${CLAUDE_PROJECT_DIR}`，正文更强调 `${CLAUDE_PLUGIN_ROOT}` 与用户 env。
- 风险：用户不清楚变量作用域和可用时机。
- 建议：在 `SKILL.md` 或 examples 旁增加“变量来源矩阵”（插件根、项目根、用户 shell）。

5. 边界：本目录不是可执行测试资产
- 现状：没有测试脚本、CI、schema lint。
- 影响：示例演进时难以自动发现字段漂移（如 type/url/headers 组合错误）。
- 建议：补充轻量 `validate-mcp-examples.sh`（JSON 语法 + 关键字段组合校验），并接入 `plugin-dev` 文档中的验证流程。

边界结论：该目录价值在“降低配置起步门槛”；其质量关键不是运行效率，而是示例的可复制性、一致性和与上游文档协议的对齐程度。
