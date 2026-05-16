# FILE 研究：plugins/plugin-dev/skills/mcp-integration/examples/stdio-server.json

## 场景与职责

`stdio-server.json` 是 `plugin-dev` 中 MCP 集成技能的本地进程型示例配置，职责是给开发者一个可直接改写为 `.mcp.json` 的 `stdio` 模板，而不是可执行脚本本身。

它在链路中的定位：

1. 上游调用方
- `plugins/plugin-dev/skills/mcp-integration/SKILL.md:67-94,527-531` 将 `stdio` 作为本地服务首选类型，并把本文件列为示例入口。
- `plugins/plugin-dev/README.md:78-93,286-291` 把 MCP 三类样例（stdio/SSE/HTTP）作为 toolkit 的“working examples”。
- `plugins/plugin-dev/commands/create-plugin.md:210-218` 在创建插件流程中要求产出 `.mcp.json`，本文件是可直接套用的输入模板。

2. 下游被调用方
- Claude Code 插件加载流程会解析 `.mcp.json`/`mcpServers` 并按 stdio 方式拉起子进程（`plugins/plugin-dev/skills/mcp-integration/SKILL.md:231-237`）。
- 配置质量检查由 `plugin-validator` 的 MCP 校验步骤兜底（`plugins/plugin-dev/agents/plugin-validator.md:116-123`）。

结论：本文件是“本地 MCP 服务配置样板”，服务的是配置编写与教学，不直接承载运行时逻辑。

## 功能点目的

1. 演示 `stdio` 最小可用结构
- 每个 server 对象都以 `command` 为核心，辅以 `args` 与 `env`（`plugins/plugin-dev/skills/mcp-integration/examples/stdio-server.json:3-25`）。
- 与类型参考一致：stdio 关键字段是 `command/args/env`（`plugins/plugin-dev/skills/mcp-integration/references/server-types.md:13-36`）。

2. 一次覆盖 3 种常见本地 server 形态
- `filesystem`：NPM 包即拉即用（`npx -y @modelcontextprotocol/server-filesystem ...`）。
- `database`：插件内自带可执行文件（`${CLAUDE_PLUGIN_ROOT}/servers/db-server.js`）。
- `custom-tools`：Python 模块形式（`python -m my_mcp_server`）。

3. 演示路径与凭据解耦
- 路径通过 `${CLAUDE_PLUGIN_ROOT}` / `${CLAUDE_PROJECT_DIR}` 占位，避免硬编码安装目录（`stdio-server.json:5,11-12`）。
- 机密通过环境变量注入，如 `${DATABASE_URL}`、`${CUSTOM_API_KEY}`（`stdio-server.json:14,22`）。

4. 为命令/Agent层 MCP 工具调用提供前置条件
- 只有 server 成功启动并注册工具后，命令中的 `allowed-tools` 或 Agent 自主调用才成立（`plugins/plugin-dev/skills/mcp-integration/references/tool-usage.md:7-15,44-85,120-156`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 关键流程

1. 开发者以本文件为蓝本编辑 `.mcp.json` 或 `plugin.json#mcpServers`（`plugins/plugin-dev/skills/mcp-integration/SKILL.md:23-59`）。
2. 插件加载时解析 server map，识别为 stdio server（无 `type` 字段时按进程型配置处理）。
3. Claude Code 为每个 server 执行 `command + args` 启动子进程，并注入 `env`。
4. 客户端与 server 通过 stdin/stdout 承载 JSON-RPC 交互（`plugins/plugin-dev/skills/mcp-integration/references/server-types.md:40-43`）。
5. 工具注册为 `mcp__plugin_<plugin-name>_<server-name>__<tool-name>` 供命令/Agent调用（`plugins/plugin-dev/skills/mcp-integration/SKILL.md:194-201`）。
6. 通过 `/mcp`、`claude --debug` 做可见性和故障诊断（`plugins/plugin-dev/skills/mcp-integration/SKILL.md:238-239,447-455`）。

### 数据结构

1. 顶层结构
- 顶层 key 为 server 名：`filesystem`、`database`、`custom-tools`（`stdio-server.json:3,10,18`）。
- 另含 `_comment` 说明字段（`stdio-server.json:2`）。

2. 单个 server 对象结构（stdio）
- 必需：`command: string`
- 常见可选：`args: string[]`、`env: Record<string,string>`

3. 三个示例对象细节
- `filesystem`：`command="npx"`，参数指向 `@modelcontextprotocol/server-filesystem` 与允许目录 `${CLAUDE_PROJECT_DIR}`（`stdio-server.json:4-8`）。
- `database`：`command` 指向插件内脚本，`--config` 指向插件内配置文件，`env` 注入数据库连接串与连接池大小（`stdio-server.json:10-16`）。
- `custom-tools`：Python module 方式启动，端口参数放在 `args`，密钥放在 `env`（`stdio-server.json:18-24`）。

### 协议与命令

1. 协议层
- `stdio` 依赖本地进程与标准 IO 通道，非 HTTP URL 连接模型（`server-types.md:5-10,40-43`）。

2. 关键命令
- `/mcp`：确认 server 是否加载与工具是否注册（`tool-usage.md:31-41`）。
- `claude --debug`：观察启动失败、协议报错、工具调用异常（`SKILL.md:447-455`）。

3. 测试与验证建议来源
- MCP 本身的验证清单：`plugins/plugin-dev/skills/mcp-integration/SKILL.md:423-441`。
- 命令与 MCP 联合验证场景：`plugins/plugin-dev/skills/command-development/references/testing-strategies.md:328-339`。

## 关键代码路径与文件引用

1. 目标文件
- `plugins/plugin-dev/skills/mcp-integration/examples/stdio-server.json:1-26`

2. 直接规范来源
- `plugins/plugin-dev/skills/mcp-integration/SKILL.md:67-94,167-189,224-237,527-531`
- `plugins/plugin-dev/skills/mcp-integration/references/server-types.md:13-36,38-44,80-99`
- `plugins/plugin-dev/skills/mcp-integration/references/authentication.md:173-200`
- `plugins/plugin-dev/skills/mcp-integration/references/tool-usage.md:7-15,29-41`

3. 调用方/流程整合
- `plugins/plugin-dev/README.md:78-93,286-291,341-351`
- `plugins/plugin-dev/commands/create-plugin.md:210-219,290-299`
- `plugins/plugin-dev/agents/plugin-validator.md:116-123,130-134`
- `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:298-309,354-367`

## 依赖与外部交互

1. 本地依赖
- 可执行命令依赖：`npx`、`python`、插件内 `servers/db-server.js`。
- 路径依赖：`${CLAUDE_PLUGIN_ROOT}` 与 `${CLAUDE_PROJECT_DIR}` 的运行时展开。

2. 环境变量依赖
- `DATABASE_URL`、`CUSTOM_API_KEY` 等必须由用户 shell 提前提供。
- 常量型参数（如 `DB_POOL_SIZE`, `DEBUG`）体现“配置即代码”风格，但值更新需改配置文件。

3. 外部交互
- `filesystem` 可能触发 NPM 下载（`npx -y`）与本地文件系统访问。
- `database/custom-tools` 与外部 DB/API 交互由被启动 server 自身逻辑决定，不在本文件内定义。

4. 与测试/脚本文档关系
- 本目录无执行脚本；测试链路依赖 `/mcp`、`claude --debug` 与 plugin-validator 的检查策略。

## 风险、边界与改进建议

1. 风险：`_comment` 可能被当成 server 键解析
- 证据：`stdio-server.json:2`。
- 建议：将说明移到文档，示例 JSON 保持纯 server map。

2. 风险：`${CLAUDE_PROJECT_DIR}` 在主文档解释不足
- 证据：示例使用该变量（`stdio-server.json:5`），而主文档主要强调 `${CLAUDE_PLUGIN_ROOT}`（`SKILL.md:171-176`）。
- 建议：补“变量来源矩阵”，明确项目根与插件根变量语义差异。

3. 风险：`npx -y` 的供应链与版本漂移
- 影响：无版本 pin 时，未来发布版本变更可能导致行为漂移。
- 建议：补充固定版本示例（如 `@modelcontextprotocol/server-filesystem@x.y.z`）。

4. 风险：可执行路径与解释器存在性未前置校验
- 影响：`python` 不存在、脚本不可执行、模块未安装都会在运行时报错。
- 建议：在 plugin-validator 中增加“命令存在性”提示，或提供 `validate-mcp-config` 脚本做静态预检。

5. 风险：`env` 参数的安全治理未在示例体现
- 影响：开发者可能将密钥打印到 stdout，污染 MCP 通道或泄露凭据。
- 建议：补充“日志只写 stderr、不输出敏感 env”示例注释，并在认证文档链接到安全章节（`authentication.md:267-310`）。

边界说明：本文件只定义“如何启动 stdio server”，不定义 server 的业务协议细节、工具 schema 与权限模型，这些在运行时 `/mcp` 发现结果中才能确定。
