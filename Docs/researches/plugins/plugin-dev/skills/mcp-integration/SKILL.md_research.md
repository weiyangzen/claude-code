# plugins/plugin-dev/skills/mcp-integration/SKILL.md 研究

## 场景与职责

`plugins/plugin-dev/skills/mcp-integration/SKILL.md` 是 `plugin-dev` 工具包中负责“外部服务接入”的核心技能文档，定位为插件开发流程里的 MCP 专项指南。

其职责不是运行时代码执行，而是为 Claude 在插件开发场景中提供可触发的知识与操作模式：

1. 定义触发边界：用户提到 “add MCP server / integrate MCP / .mcp.json / Model Context Protocol / stdio/SSE/HTTP/ws” 时应加载该技能（`plugins/plugin-dev/skills/mcp-integration/SKILL.md:2-5`）。
2. 规范配置方式：给出 `.mcp.json` 与 `plugin.json.mcpServers` 两种配置模型（`.../SKILL.md:19-64`）。
3. 统一接入语义：覆盖 server type、鉴权、工具命名、生命周期、调试与测试（`.../SKILL.md:65-554`）。
4. 为上层工作流提供“实现阶段模板”：`/plugin-dev:create-plugin` 在 Phase 5 明确要求加载此技能并产出 `.mcp.json` 与 README 环境变量说明（`plugins/plugin-dev/commands/create-plugin.md:157-219`）。

在 `plugin-dev` 全局架构中，该技能是“Integrate Services”阶段的知识承载体（`plugins/plugin-dev/README.md:229-246`），与 `plugin-structure`（manifest 与路径）、`command-development`（allowed-tools）、`hook-development`（MCP 工具拦截）、`plugin-validator`（配置校验）形成协作链。

## 功能点目的

### 1) 触发描述与技能加载目的

- 目的：提高技能自动加载准确度，避免在普通命令/插件结构问题中误触发。
- 实现：frontmatter `description` 列举高辨识短语与 MCP 专有词（`.../SKILL.md:2-5`）。
- 上游依赖：`plugin-dev/README.md` 中公开的触发短语与该技能 frontmatter 对齐（`plugins/plugin-dev/README.md:76-93`）。

### 2) 配置模型目的（.mcp.json vs inline）

- 目的：兼顾简单插件（单文件）与复杂插件（多服务拆分）。
- 实现：
  - 推荐 `.mcp.json` 独立文件（`.../SKILL.md:23-43`）。
  - 支持在 `plugin.json` 内联 `mcpServers`（`.../SKILL.md:44-64`）。
- 相关规范：`plugin-structure` 对 `mcpServers` 字段定义为“路径或对象”，默认 `./.mcp.json`（`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:298-331`）。

### 3) 传输类型与认证策略目的

- 目的：让开发者按服务形态选择接入方式，减少“传输层与鉴权错配”。
- 实现：主文档给出 stdio/SSE/HTTP/ws 四类（`.../SKILL.md:65-166`），并在 `references/server-types.md` 做深入分解（`.../references/server-types.md:1-536`）。
- 鉴权扩展：`references/authentication.md` 提供 OAuth、Token、headersHelper、多租户与故障排查（`.../references/authentication.md:1-549`）。

### 4) MCP 工具调用治理目的

- 目的：把“可发现工具”转化为“可控工具权限”，降低越权与误调用风险。
- 实现：
  - 规范工具名格式 `mcp__plugin_<plugin-name>_<server-name>__<tool-name>`（`.../SKILL.md:190-201`）。
  - 在命令 frontmatter 预授权 `allowed-tools`，并提示慎用 wildcard（`.../SKILL.md:202-223`）。
  - `references/tool-usage.md` 继续覆盖参数校验、错误处理、批量/并行调用（`.../references/tool-usage.md:42-538`）。

### 5) 开发调试与测试目的

- 目的：给出最小可执行验证闭环，确保接入不是“只写配置不验证”。
- 实现：
  - `/mcp` 查看 server 与 tool 注册结果（`.../SKILL.md:238-240,427-431`）。
  - `claude --debug` 观察连接/鉴权/工具调用日志（`.../SKILL.md:447-455`）。
  - 主文档内置 validation checklist（`.../SKILL.md:433-441`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 关键流程：从插件加载到工具可用

根据 `mcp-integration` 与 `plugin-structure` 的组合语义，插件启用后流程为：

1. Claude Code 读取 `.claude-plugin/plugin.json`。
2. 解析 `mcpServers`（inline 或 `./.mcp.json` 路径）并定位配置文件（`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:298-367`）。
3. 针对每个 server：
   - stdio：启动本地子进程，通过 stdin/stdout 交换 JSON-RPC（`.../references/server-types.md:38-44`）。
   - SSE/HTTP/ws：按 URL 建连并做协议握手（`.../references/server-types.md:133-140,237-243,334-341`）。
4. 服务端工具被注册为 `mcp__plugin_...__...` 名称空间工具（`.../SKILL.md:190-201,231-237`）。
5. 命令通过 `allowed-tools` 精准授权调用，Agent 自治调用（`.../references/tool-usage.md:44-156`）。

### B. 数据结构与配置模型

1. **Server 字典结构**
   - 顶层 key 为 server-name，value 为 server config。
   - stdio 最小字段：`command`；可选 `args`, `env`（`.../SKILL.md:71-82`）。
   - 网络型最小字段：`type + url`；可选 `headers`（`.../SKILL.md:99-159`）。

2. **工具命名结构**
   - `mcp__plugin_<plugin-name>_<server-name>__<tool-name>`（`.../SKILL.md:194-201`）。
   - 该结构被 hook matcher 直接用于拦截规则（如 `mcp__.*__delete.*`）（`plugins/plugin-dev/skills/hook-development/SKILL.md:404-419`）。

3. **环境变量展开结构**
   - 插件内路径：`${CLAUDE_PLUGIN_ROOT}`（`.../SKILL.md:167-176`）。
   - 用户环境变量：`${API_TOKEN}`、`${DB_URL}` 等（`.../SKILL.md:178-186`）。
   - 扩展文档给出多租户 URL/header 参数化示例（`.../references/authentication.md:311-347`）。

### C. 协议与命令层面

1. **协议面**
   - stdio：进程内 JSON-RPC（stdin/stdout）
   - SSE：HTTP + 事件流 + 重连
   - HTTP：无状态请求/响应
   - ws：双向长连接
   - 见 `server-types.md` 对四类生命周期对比矩阵（`.../references/server-types.md:371-409`）。

2. **命令面**
   - 发现工具：`/mcp`（`.../references/tool-usage.md:31-41`）。
   - 调试连接：`claude --debug`（`.../SKILL.md:447-455`，`.../references/authentication.md:384-394`）。
   - 鉴权独立排查：`curl -H "Authorization: Bearer $API_TOKEN" ...`（`.../references/authentication.md:395-402`）。

3. **性能面**
   - 文档建议批量查询、缓存、并行独立 tool call（`.../SKILL.md:410-421`，`.../references/tool-usage.md:313-357`）。

## 关键代码路径与文件引用

### 核心对象（被研究文件）

1. `plugins/plugin-dev/skills/mcp-integration/SKILL.md`
   - 触发定义：`2-5`
   - 配置方法：`19-64`
   - 类型/鉴权/生命周期：`65-287`
   - 工具命名与授权：`190-223`
   - 测试调试：`423-476`
   - 资源导航与实施清单：`515-554`

### 被调用方（该技能引用的资源）

1. `plugins/plugin-dev/skills/mcp-integration/references/server-types.md`
   - 4 种 transport 深入说明、选型矩阵与迁移模式。
2. `plugins/plugin-dev/skills/mcp-integration/references/authentication.md`
   - OAuth/Token/动态 headers/高级认证方案与故障排查。
3. `plugins/plugin-dev/skills/mcp-integration/references/tool-usage.md`
   - 命令与 Agent 的 MCP 工具使用范式。
4. `plugins/plugin-dev/skills/mcp-integration/examples/*.json`
   - stdio/SSE/HTTP 配置样例。

### 调用方（谁依赖该技能）

1. `plugins/plugin-dev/README.md`
   - 将 MCP Integration 列为七大核心技能之一并提供触发语（`76-93`）。
   - 在工作流图中将其定位为“Integrate Services”阶段（`244-246`）。
2. `plugins/plugin-dev/commands/create-plugin.md`
   - Phase 5 强制按需加载 `mcp-integration`（`157-163`）。
   - For MCP 子流程要求产出 `.mcp.json` 与环境变量文档（`210-219`）。
3. `plugins/plugin-dev/skills/skill-development/SKILL.md`
   - 把 `../mcp-integration/` 作为“高质量技能模板”之一（`608-614`）。

### 横向协作依赖文件（非直接调用但强相关）

1. `plugins/plugin-dev/agents/plugin-validator.md`
   - 校验 `.mcp.json` / `mcpServers` 配置合法性与 HTTPS/WSS 安全要求（`116-133`）。
2. `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md`
   - 定义 `mcpServers` 字段类型、默认值、加载顺序（`298-367`）。
3. `plugins/plugin-dev/skills/hook-development/SKILL.md`
   - 通过 matcher 规则支持 MCP 工具审计/拦截（`404-419`）。
4. `plugins/plugin-dev/skills/command-development/references/testing-strategies.md`
   - 提供“Command + MCP Integration”测试场景（`328-339`）。

## 依赖与外部交互

### 内部依赖

1. **插件系统约束**：依赖 `.claude-plugin/plugin.json` 及自动发现机制（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:339-349`）。
2. **命令权限机制**：依赖 `allowed-tools` 控制 MCP tool 调用边界（`.../SKILL.md:202-223`）。
3. **验证链路**：依赖 `plugin-validator` 的结构与安全检查来兜底（`plugins/plugin-dev/agents/plugin-validator.md:116-133`）。

### 外部交互

1. **MCP 服务端**：本地子进程或远端 URL（SSE/HTTP/ws）。
2. **认证系统**：OAuth 浏览器流、Token/Key、动态 header 生成。
3. **Claude Code CLI**：`/mcp` 与 `claude --debug` 是主要可观测接口。
4. **外部文档/SDK**：
   - `https://modelcontextprotocol.io/`
   - `https://docs.claude.com/en/docs/claude-code/mcp`
   - `@modelcontextprotocol/sdk`
   （来源：`plugins/plugin-dev/skills/mcp-integration/SKILL.md:535-538`）

## 风险、边界与改进建议

### 风险与边界

1. **配置格式存在跨文档歧义**
   - `mcp-integration/SKILL.md` 的 `.mcp.json` 示例采用“顶层即 server map”（`.../SKILL.md:27-37`）。
   - `plugin-structure/examples/advanced-plugin.md` 的 `.mcp.json` 示例采用 `{ "mcpServers": { ... } }` 包裹结构（`.../advanced-plugin.md:147-175`）。
   - 该差异会导致用户复制示例后不确定 loader 期望格式。

2. **示例变量口径不一致**
   - 主文档重点强调 `${CLAUDE_PLUGIN_ROOT}`（`.../SKILL.md:171-176`）。
   - `examples/stdio-server.json` 使用 `${CLAUDE_PROJECT_DIR}`（`.../examples/stdio-server.json:5`）且主文档未解释其语义来源。

3. **工具名硬编码脆弱性**
   - 文档鼓励在 `allowed-tools` 写全名，但不同服务 tool 命名可能变化；若不先用 `/mcp` 实测，容易授权错误（`.../references/tool-usage.md:29-41`）。

4. **认证章节包含“能力暗示”但缺少能力边界声明**
   - `headersHelper`、mTLS workaround、JWT/HMAC 示例较高级（`.../references/authentication.md:227-518`）。
   - 若 Claude Code 版本或 server 实现不支持对应字段，用户可能误判为平台保证能力。

5. **测试偏手工，缺少自动化资产**
   - 文档提供 checklist 与手工步骤（`.../SKILL.md:423-441`），但本技能目录没有实际验证脚本，不利于 CI 一致性。

### 改进建议

1. **统一 `.mcp.json` 规范并在全套文档收敛为单一形态**
   - 建议在 `mcp-integration/SKILL.md` 与 `plugin-structure` 示例中统一声明：
     - `.mcp.json` 文件是“server map”还是“包裹 `mcpServers` 对象”；
     - `plugin.json.mcpServers` 在 path vs inline 两种模式下的确切解析行为。

2. **补充“变量字典”章节**
   - 在主文档新增支持变量表（`CLAUDE_PLUGIN_ROOT`、`CLAUDE_PROJECT_DIR`、用户环境变量），并标注来源与适用范围。

3. **给出最小可执行验证脚本模板**
   - 增加 `scripts/validate-mcp-config.sh`（JSON 语法、URL scheme、command existence、必填 env vars），与 `plugin-validator` 建议联动。

4. **增加版本/兼容性提示**
   - 在 `headersHelper`、ws、高级认证段落加“依赖 Claude Code 版本或服务端实现”的显式提示，避免把示例当作稳定契约。

5. **强化与命令/Hook 安全策略的联动示例**
   - 增加“命令最小授权 + PreToolUse MCP 删除保护 matcher”端到端样例，打通 `mcp-integration` 与 `hook-development` 的协作。

