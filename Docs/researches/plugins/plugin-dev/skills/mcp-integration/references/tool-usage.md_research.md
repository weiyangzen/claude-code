# plugins/plugin-dev/skills/mcp-integration/references/tool-usage.md 研究

## 场景与职责

tool-usage 文档是 MCP 集成落地到命令与代理执行层的操作规范。它解决的问题不是“如何连接 server”，而是“连接后如何安全、稳定、高效地调用工具”。

在体系中的职责定位：

1. 承接 `server-types.md`（连接）与 `authentication.md`（认证）的后续阶段，定义工具命名、授权、调用、错误处理和测试（`plugins/plugin-dev/skills/mcp-integration/references/tool-usage.md:7-15,44-85,157-208,410-446`）。
2. 与 command frontmatter 的 `allowed-tools` 语义对齐，把最小权限原则落实到命令级访问控制（`plugins/plugin-dev/skills/command-development/references/frontmatter-reference.md:60-128`）。
3. 在 `mcp-integration/SKILL.md` 中作为引用文档，支持 “Pre-allow specific MCP tools” 的策略执行（`plugins/plugin-dev/skills/mcp-integration/SKILL.md:202-223,365-376,521-523`）。

该文档属于“运行手册层”，本身不实现工具调用代码。

## 功能点目的

### 1) 统一工具命名协议，确保调用定位准确

通过 `mcp__plugin_<plugin-name>_<server-name>__<tool-name>` 统一命名，避免多插件多 server 场景的歧义和冲突（`plugins/plugin-dev/skills/mcp-integration/references/tool-usage.md:7-27`）。

### 2) 建立 command 与 agent 的权限边界

文档明确 command 场景要通过 frontmatter 预授权，agent 场景默认更自治，目的在于平衡“可控性”和“自动化效率”（`plugins/plugin-dev/skills/mcp-integration/references/tool-usage.md:44-85,120-156`）。

### 3) 给出调用模式与参数处理套路

覆盖单次调用、串行调用、批处理、错误重试、参数校验、部分成功汇报等常见模式，降低命令作者重复设计成本（`plugins/plugin-dev/skills/mcp-integration/references/tool-usage.md:157-311`）。

### 4) 提供性能与测试闭环

将 batching/caching/parallel 调用、用户反馈、调试与测试场景放在同一文档，目标是提升真实使用场景下的成功率和响应体验（`plugins/plugin-dev/skills/mcp-integration/references/tool-usage.md:313-419,502-527`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 关键流程

1. MCP server 建连并完成 tool discover 后，工具以统一前缀暴露。
2. 命令执行时根据 `allowed-tools` 白名单决策是否允许调用某 MCP 工具（`plugins/plugin-dev/skills/mcp-integration/references/tool-usage.md:44-85`）。
3. 调用前读取 `/mcp` 提供的 `inputSchema` 进行参数构造与校验（`plugins/plugin-dev/skills/mcp-integration/references/tool-usage.md:29-40,210-273`）。
4. 调用后按成功/失败/部分成功分支做用户态反馈与重试建议（`plugins/plugin-dev/skills/mcp-integration/references/tool-usage.md:275-311`）。
5. 在批量或多资源场景执行 batching/caching/parallel 模式，优化总调用成本（`plugins/plugin-dev/skills/mcp-integration/references/tool-usage.md:313-357`）。
6. 开发期使用 `/mcp` 与 `claude --debug` 进行可见性与故障定位（`plugins/plugin-dev/skills/mcp-integration/references/tool-usage.md:414-419,504-519`）。

### B. 关键数据结构

1. 工具名结构：
   - `mcp__plugin_<plugin-name>_<server-name>__<tool-name>`（`plugins/plugin-dev/skills/mcp-integration/references/tool-usage.md:13-15`）。
2. command 授权结构：
   - frontmatter `allowed-tools` 支持数组、精确工具名和 wildcard（`plugins/plugin-dev/skills/mcp-integration/references/tool-usage.md:48-84`）。
3. 工具 schema 结构：
   - `inputSchema.type/properties/required` 决定输入构造（`plugins/plugin-dev/skills/mcp-integration/references/tool-usage.md:216-240`）。
4. 结果汇报结构：
   - 支持 success/error/partial-success 三类输出模式（`plugins/plugin-dev/skills/mcp-integration/references/tool-usage.md:279-311`）。

### C. 协议与命令

1. 协议语义：
   - 文档从调用侧抽象为“toolName + input”的 JSON 结构，不依赖具体 transport（`plugins/plugin-dev/skills/mcp-integration/references/tool-usage.md:244-258`）。
2. 命令接口：
   - `/mcp` 用于工具发现、schema 查看与命名核对（`plugins/plugin-dev/skills/mcp-integration/references/tool-usage.md:31-41`）。
   - `claude --debug` 用于调用失败定位（`plugins/plugin-dev/skills/mcp-integration/references/tool-usage.md:418,518`）。

### D. 与 command/agent 规范的衔接

1. command：
   - 与 frontmatter 文档的“尽量最小工具集合”原则一致（`plugins/plugin-dev/skills/command-development/references/frontmatter-reference.md:107-128`）。
2. agent：
   - 文档给出“可自治调用”的使用方式，但建议在 agent prompt 内声明常用工具集合以提升可预测性（`plugins/plugin-dev/skills/mcp-integration/references/tool-usage.md:124-156`）。

## 关键代码路径与文件引用

核心研究对象：

1. `plugins/plugin-dev/skills/mcp-integration/references/tool-usage.md:1-538`

直接调用方/入口：

1. `plugins/plugin-dev/skills/mcp-integration/SKILL.md:190-223,365-376,499-505,521-523`
2. `plugins/plugin-dev/README.md:347-349`
3. `plugins/plugin-dev/commands/create-plugin.md:157-163`（通过加载 mcp-integration 间接引用）

关键上下文依赖：

1. `plugins/plugin-dev/skills/command-development/references/frontmatter-reference.md:60-128`
   - 定义 `allowed-tools` 类型、格式与最小权限建议。
2. `plugins/plugin-dev/skills/command-development/references/testing-strategies.md:328-339`
   - 提供 command + MCP 集成测试场景。
3. `plugins/plugin-dev/skills/mcp-integration/references/server-types.md:38-44,133-140,237-243,334-341`
   - 决定工具调用前的连接形态。
4. `plugins/plugin-dev/skills/mcp-integration/references/authentication.md:348-402`
   - 决定工具调用前的鉴权有效性与调试手段。
5. `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:298-330`
   - 决定 MCP 配置来源位置。

## 依赖与外部交互

### 内部依赖

1. 依赖 runtime 工具注册机制把远端能力映射为 `mcp__plugin_...` 名称。
2. 依赖 command/agent 执行器处理 `allowed-tools` 和工具调用分发。
3. 依赖 `server-types`、`authentication` 文档定义的连接与认证前提。

### 外部交互

1. 与外部 MCP server 交互执行实际业务工具调用。
2. 与 `/mcp` 命令界面交互，读取 server/tool/schema 目录。
3. 与 CLI debug 输出交互进行故障定位。

### 配置、测试、脚本交互

1. 典型测试路径：`.mcp.json` -> 本地安装 plugin -> `/mcp` -> command 执行 -> `claude --debug`（`plugins/plugin-dev/skills/mcp-integration/references/tool-usage.md:414-419`）。
2. 在 command-development 测试文档中，MCP 集成被列为 L7 集成测试场景（`plugins/plugin-dev/skills/command-development/references/testing-strategies.md:328-339`）。

## 风险、边界与改进建议

### 风险 1（高）：agent“无需预授权”描述可能导致权限边界误解

- 现状：文档写明 agent 可以不预先白名单并由 Claude 自主决定（`plugins/plugin-dev/skills/mcp-integration/references/tool-usage.md:124-155`）。
- 影响：在高敏感写操作场景，团队可能低估权限外溢风险。
- 建议：补充“高风险操作优先 command + 精确 `allowed-tools`”的决策矩阵。

### 风险 2（高）：wildcard 授权虽标注谨慎，但缺审计与回收机制

- 现状：提供 `mcp__plugin_x_y__*` 语法并提示谨慎（`plugins/plugin-dev/skills/mcp-integration/references/tool-usage.md:76-85`）。
- 影响：随着 server 新增工具，历史 command 权限会被动扩张。
- 建议：增加“发布前枚举工具清单 + 定期收敛白名单”检查清单。

### 风险 3（中）：错误处理建议较通用，缺分级重试策略

- 现状：建议最多重试 3 次，但未区分 `429/5xx/网络超时/参数错误`（`plugins/plugin-dev/skills/mcp-integration/references/tool-usage.md:196-208`）。
- 影响：可能对不可重试错误做无效重试，拉高延迟。
- 建议：补充分级重试表（可重试/不可重试、退避策略、熔断阈值）。

### 风险 4（中）：性能优化章节未强调一致性与幂等约束

- 现状：强调并行、缓存、批处理（`plugins/plugin-dev/skills/mcp-integration/references/tool-usage.md:313-357`），但未说明写操作并发的幂等要求。
- 影响：并行写可能造成重复更新或顺序竞争。
- 建议：补充“读可并行、写需幂等键或序列化”的操作规则。

### 风险 5（中）：测试建议以手工为主，自动化不足

- 现状：文档主要提供手工测试步骤（`plugins/plugin-dev/skills/mcp-integration/references/tool-usage.md:410-446`）。
- 影响：回归测试成本高，质量依赖个人执行一致性。
- 建议：补充最小自动化脚本模板（schema contract test、错误码断言、性能基线）。

### 边界

1. 本文档不定义 MCP server 的业务语义，只约束调用方法。
2. 它不能替代认证文档和 server 类型文档，三者需联用。
3. 具体工具参数和返回字段由远端 server schema 决定，文档只提供通用调用框架。
