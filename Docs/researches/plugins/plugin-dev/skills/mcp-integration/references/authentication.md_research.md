# plugins/plugin-dev/skills/mcp-integration/references/authentication.md 研究

## 场景与职责

`authentication.md` 是 `mcp-integration` 技能的认证专题参考文档，负责把“能连上 MCP server”扩展为“可安全上线的认证方案”。它不定义 MCP 传输类型本身，而是覆盖不同 server 类型下的认证策略选择、配置写法、故障排查与迁移路径。

在插件开发流程中的职责位置：

1. 在 `mcp-integration/SKILL.md` 的 `Additional Resources` 中被显式列为深入参考文件，用于主文档触发后按需下钻（`plugins/plugin-dev/skills/mcp-integration/SKILL.md:515-523`）。
2. 在 `/plugin-dev:create-plugin` 的 Phase 5 MCP 实施阶段，间接作为“如何写认证配置和 README”的规范来源（`plugins/plugin-dev/commands/create-plugin.md:210-218`）。
3. 在工具包总览里，MCP 能力强调“OAuth + token + env vars + security-first”，该文件承担这部分可执行细节（`plugins/plugin-dev/README.md:80-93,358-363`）。

它的核心边界是“文档约束层”，不实现 runtime 鉴权逻辑；token 存储、OAuth 回调、刷新等由 Claude Code 运行时负责。

## 功能点目的

### 1) 统一 OAuth 与非 OAuth 两类认证路径

文件先定义 SSE/HTTP 下的 OAuth 自动流程（首次授权、token 存储、自动刷新），再给出 Bearer/API Key/自定义 header 的静态 token 模式，避免插件作者把所有服务都当成一种认证模型（`plugins/plugin-dev/skills/mcp-integration/references/authentication.md:13-21,80-139`）。

### 2) 将“凭据安全”内建到配置范式

文档反复强调 `环境变量注入 > 配置硬编码`，并给出 README 文档模板，目的是让插件仓库可开源、可复用，同时降低密钥泄漏风险（`plugins/plugin-dev/skills/mcp-integration/references/authentication.md:141-171,267-309`）。

### 3) 覆盖短期凭据与企业认证的高级场景

通过 `headersHelper`、JWT、HMAC、mTLS wrapper 场景，解决仅靠静态 header 难以覆盖的短时令牌和签名链路（`plugins/plugin-dev/skills/mcp-integration/references/authentication.md:227-258,458-518`）。

### 4) 给出可执行排障与迁移动作

文档包含 `401/403` 排查、`claude --debug`、`curl` 冒烟验证，以及从硬编码迁移到 env vars、从 Basic Auth 迁移到 OAuth 的步骤，目标是缩短故障闭环时间（`plugins/plugin-dev/skills/mcp-integration/references/authentication.md:348-457`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 关键流程

1. 插件加载 MCP 配置后，首次工具调用触发鉴权判定。
2. 若服务支持 OAuth（文档示例以 SSE/HTTP 为主），Claude Code 打开浏览器授权并保存 token，后续自动刷新（`plugins/plugin-dev/skills/mcp-integration/references/authentication.md:13-21,57-64`）。
3. 若为 token/header 模式，运行时对 `headers` 和 `env` 做变量展开后发起请求（`plugins/plugin-dev/skills/mcp-integration/references/authentication.md:86-120,173-200`）。
4. 若配置 `headersHelper`，先执行脚本并解析 JSON header，再附加到网络请求（`plugins/plugin-dev/skills/mcp-integration/references/authentication.md:233-258`）。
5. 失败时按错误码和日志定位：`401/403` -> 凭据/权限，`claude --debug` -> 运行时链路，`curl` -> 服务侧快速验证（`plugins/plugin-dev/skills/mcp-integration/references/authentication.md:352-402`）。

### B. 关键数据结构

1. OAuth 最小配置（无需显式 auth 字段）：
   - `{"type":"sse","url":"https://..."}`（`plugins/plugin-dev/skills/mcp-integration/references/authentication.md:24-33`）。
2. Token 认证结构：
   - `headers.Authorization = "Bearer ${API_TOKEN}"`（`plugins/plugin-dev/skills/mcp-integration/references/authentication.md:87-97`）。
3. stdio 凭据透传结构：
   - `env` 中注入 `DATABASE_URL/DB_USER/DB_PASSWORD`（`plugins/plugin-dev/skills/mcp-integration/references/authentication.md:179-200`）。
4. 动态 header 结构：
   - `headersHelper` 指向脚本，脚本 stdout 输出 JSON object（`plugins/plugin-dev/skills/mcp-integration/references/authentication.md:233-258`）。
5. 多租户结构：
   - tenant/workspace 可放在 header 或 URL 模板（`plugins/plugin-dev/skills/mcp-integration/references/authentication.md:315-346`）。

### C. 协议与命令

1. 协议层：
   - OAuth 2.0 授权码语义（浏览器跳转 + consent + refresh）由 Claude Code runtime 托管（`plugins/plugin-dev/skills/mcp-integration/references/authentication.md:13-21,57-78`）。
   - 非 OAuth 场景通过 HTTP header 承载鉴权凭据（Bearer/API Key/custom headers）。
2. 调试命令：
   - `claude --debug`：观察认证流、token 刷新和错误。 
   - `curl -H "Authorization: Bearer $API_TOKEN" <health-url>`：验证服务端凭据可用性（`plugins/plugin-dev/skills/mcp-integration/references/authentication.md:384-402`）。

### D. 与插件配置工作流的衔接

1. `plugin.json` 的 `mcpServers` 可指向 `./.mcp.json` 或内联对象，认证配置最终都落在该结构内（`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:298-330`）。
2. `create-plugin` 的 MCP phase 要求补齐 env var 文档与 setup instructions，这与本文件的 README 模板形成一一对应（`plugins/plugin-dev/commands/create-plugin.md:212-218`）。

## 关键代码路径与文件引用

核心研究对象：

1. `plugins/plugin-dev/skills/mcp-integration/references/authentication.md:1-549`

直接调用方/入口：

1. `plugins/plugin-dev/skills/mcp-integration/SKILL.md:241-270,515-523`
2. `plugins/plugin-dev/README.md:80-93,218-221,344-349`
3. `plugins/plugin-dev/commands/create-plugin.md:157-163,210-218`

关键契约与上下文依赖：

1. `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:298-330,354-367`
   - 决定认证配置挂载位置（`mcpServers`）。
2. `plugins/plugin-dev/agents/plugin-validator.md:116-134`
   - 校验 MCP 配置完整性、HTTPS/WSS、安全项（无硬编码凭据）。
3. `plugins/plugin-dev/skills/mcp-integration/references/server-types.md:141-170,244-272,302-332`
   - 提供“按 server type 选择认证载体”的上游上下文。
4. `plugins/plugin-dev/skills/mcp-integration/examples/sse-server.json:11-18`
5. `plugins/plugin-dev/skills/mcp-integration/examples/http-server.json:3-19`
6. `plugins/plugin-dev/skills/mcp-integration/examples/stdio-server.json:10-24`

测试与排障关联：

1. `plugins/plugin-dev/skills/mcp-integration/references/tool-usage.md:410-419,502-519`
2. `plugins/plugin-dev/skills/command-development/references/testing-strategies.md:328-339`

## 依赖与外部交互

### 内部依赖

1. 依赖 `mcp-integration/SKILL.md` 触发器与 progressive disclosure 机制进行加载（`plugins/plugin-dev/skills/mcp-integration/SKILL.md:1-4,515-523`）。
2. 依赖 `manifest-reference.md` 的路径解析和默认值规则，认证配置才会被运行时读取（`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:300-309,356-367`）。
3. 依赖 command frontmatter 的权限约束，认证成功后工具调用仍受 `allowed-tools` 控制（`plugins/plugin-dev/skills/command-development/references/frontmatter-reference.md:60-128`）。

### 外部交互

1. OAuth 授权页面与第三方 IdP（浏览器交互、同意授权）。
2. 目标 MCP endpoint 的鉴权与健康检查接口（Bearer/API Key/custom headers）。
3. 本地 shell 环境变量系统（`export`、`.env` 加载）。
4. 动态 header 生成脚本（bash/openssl 等命令行依赖）。

### 配置与脚本交互

1. 通过 `headersHelper` 调用外部脚本，要求 stdout 输出严格 JSON。
2. 通过 `env` 将凭据传入 stdio 子进程，避免写死在仓库文件。

## 风险、边界与改进建议

### 风险 1（高）：OAuth 适用范围表述存在跨文档差异

- 现状：`authentication.md` 提到 OAuth 自动流覆盖 SSE 和 HTTP（`plugins/plugin-dev/skills/mcp-integration/references/authentication.md:13`）；而 `server-types.md` 主要把 OAuth 示例放在 SSE 语境（`plugins/plugin-dev/skills/mcp-integration/references/server-types.md:141-157`）。
- 影响：插件作者可能误判 HTTP 场景是否必然支持 OAuth 自动化。
- 建议：在两文档增加“runtime 支持能力 vs 服务端是否实现 OAuth”的明确判定表。

### 风险 2（高）：`headersHelper` 执行链路缺少安全约束说明

- 现状：文档说明了如何执行脚本输出 headers，但未定义脚本可信来源、超时、输出大小、失败回退策略。
- 影响：可能引入命令注入、阻塞或泄漏风险。
- 建议：补充 `scripts/` 安全基线（只读权限、超时上限、stderr redaction、非零退出处理）。

### 风险 3（中）：`.env` 加载示例对复杂值不稳健

- 现状：示例给出 `export $(cat .env | xargs)`（`plugins/plugin-dev/skills/mcp-integration/references/authentication.md:223`）。
- 影响：含空格/特殊字符的值可能被错误拆分。
- 建议：改为更稳健方式（例如 `set -a; source .env; set +a`）并明确 `.env` 格式要求。

### 风险 4（中）：高级认证（mTLS/JWT/HMAC）缺少与 MCP 生命周期协同说明

- 现状：给出了 wrapper 或脚本示例，但缺少“token 刷新频率、时钟漂移、重试节流”的操作建议。
- 影响：高并发下可能出现间歇性 401/签名失效。
- 建议：在高级认证章节增加时钟同步、过期窗口、幂等重试建议。

### 风险 5（中）：排障命令偏手工，缺自动化自检脚本

- 现状：主要依赖 `claude --debug` 和 `curl` 手工操作（`plugins/plugin-dev/skills/mcp-integration/references/authentication.md:384-402`）。
- 影响：团队内可重复性不高。
- 建议：新增 `scripts/check-auth.sh` 示例，统一做 env 检查、health 探测和错误码汇总。

### 边界

1. 本文档不控制 token 存储实现与 OAuth 回调实现，只给配置与使用建议。
2. 工具是否可调用仍受 command/agent 权限模型影响，不由认证文档单独决定。
3. 认证成功不等于业务成功，仍需配合 `tool-usage` 的参数校验与错误处理策略。
