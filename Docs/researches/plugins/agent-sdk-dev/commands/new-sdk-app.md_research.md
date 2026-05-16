# FILE `plugins/agent-sdk-dev/commands/new-sdk-app.md` 研究文档

## 场景与职责

`plugins/agent-sdk-dev/commands/new-sdk-app.md` 是 `agent-sdk-dev` 插件的主入口命令协议，目标是把“新建 Claude Agent SDK 项目”的完整流程（需求澄清、项目初始化、依赖安装、可运行验证、交付引导）编排成可执行对话指令，而不是可执行脚本。

在插件体系中的职责分工：

- 作为用户入口：由 `/new-sdk-app [project-name]` 暴露给用户（`plugins/agent-sdk-dev/commands/new-sdk-app.md:1-4`，`plugins/agent-sdk-dev/README.md:11-33`）。
- 作为流程编排层：定义固定提问顺序、计划生成、安装与验证阶段（`plugins/agent-sdk-dev/commands/new-sdk-app.md:27-136`）。
- 作为路由层：在验证阶段按语言分发到 `agent-sdk-verifier-ts` 或 `agent-sdk-verifier-py`（`plugins/agent-sdk-dev/commands/new-sdk-app.md:128-135`）。
- 作为质量闸门：明确“验证通过前不算完成”，尤其 TypeScript 强制 `npx tsc --noEmit`（`plugins/agent-sdk-dev/commands/new-sdk-app.md:117-127,163-166`）。

其上游发现机制来自插件系统约定：`commands/` 目录内 Markdown 命令会自动发现并注册（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:112-115`），插件本体由 marketplace 清单注册到 `./plugins/agent-sdk-dev`（`.claude-plugin/marketplace.json:12-16`）。

## 功能点目的

该命令覆盖 6 类关键能力，每一类都对应明确目的与交付结果：

1. 文档对齐与版本新鲜度控制  
   目的：避免基于过期 SDK 认知生成脚手架。  
   机制：先读 overview，再按语言读 TS/Python 文档，并要求安装前检查 npm/PyPI 最新版本（`plugins/agent-sdk-dev/commands/new-sdk-app.md:8-25,74-79,110,162-169`）。

2. 渐进式需求采集  
   目的：降低一次性多问题导致的信息遗漏与误配。  
   机制：强制“一次只问一个问题”，顺序为语言 -> 项目名 -> agent 类型 -> 起点 -> 工具偏好；支持 `$ARGUMENTS` 直接注入项目名（`plugins/agent-sdk-dev/commands/new-sdk-app.md:27-58,174-176`）。

3. 脚手架计划化执行  
   目的：把“怎么建项目”结构化，减少临场遗漏。  
   机制：计划项固定覆盖初始化、版本检查、安装、starter file、环境配置、可选 `.claude/` 结构（`plugins/agent-sdk-dev/commands/new-sdk-app.md:60-105`）。

4. 语言分支安装与校验  
   目的：保证 TS/Python 路径都可交付且可核验。  
   机制：TS 分支执行 `npm install @anthropic-ai/claude-agent-sdk@latest` + `npm list`；Python 分支执行 `pip install claude-agent-sdk` + `pip show`（`plugins/agent-sdk-dev/commands/new-sdk-app.md:81-87`）。

5. 运行前质量门禁  
   目的：交付前尽量阻断显性错误。  
   机制：TS 强制 typecheck 清零后才可结束；Python 要求导入与基础语法正确（`plugins/agent-sdk-dev/commands/new-sdk-app.md:117-127`）。

6. 交付后可启动引导  
   目的：让用户立即运行、扩展并持续迭代。  
   机制：输出 API key 配置、运行命令、文档链接和常见下一步（`plugins/agent-sdk-dev/commands/new-sdk-app.md:137-158`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 协议结构与元数据

命令采用 Markdown + YAML frontmatter 结构，当前定义：

- `description: Create and setup a new Claude Agent SDK application`
- `argument-hint: [project-name]`

证据：`plugins/agent-sdk-dev/commands/new-sdk-app.md:1-4`。  
`argument-hint` 的语义是提示参数槽位与补全（`plugins/plugin-dev/skills/command-development/references/frontmatter-reference.md:196-236`）。

### 2) 关键状态机（对话流程）

该命令本质是“分阶段状态机”，可以抽象为：

1. `DocSync`：读取官方文档并确认最新版本来源。  
2. `ReqCollect`：按固定顺序单问采集 5 个槽位。  
3. `PlanConfirm`：输出计划并获用户确认。  
4. `ScaffoldExec`：创建目录/配置/依赖/示例文件。  
5. `SelfVerify`：执行 TS 或 Py 的本地基础验证。  
6. `VerifierDispatch`：按语言调 verifier agent。  
7. `Handoff`：输出运行与扩展指南。

对应证据链：`plugins/agent-sdk-dev/commands/new-sdk-app.md:8-176`。

### 3) 输入“数据结构”（隐式）

虽然没有 JSON schema，但命令通过提问顺序定义了稳定输入槽位：

- `language`: TypeScript | Python
- `project_name`: 来自 `$ARGUMENTS` 或交互输入
- `agent_type`: coding | business | custom 描述
- `starting_point`: minimal | basic | specific
- `tooling_choice`: npm/yarn/pnpm 或 pip/poetry 等偏好

证据：`plugins/agent-sdk-dev/commands/new-sdk-app.md:31-57`。

### 4) 关键命令协议（执行面）

TypeScript 路径命令要求：

- 初始化：`npm init -y`
- 安装：`npm install @anthropic-ai/claude-agent-sdk@latest`
- 版本确认：`npm list @anthropic-ai/claude-agent-sdk`
- 强制验证：`npx tsc --noEmit`

Python 路径命令要求：

- 安装：`pip install claude-agent-sdk`
- 版本确认：`pip show claude-agent-sdk`
- 验证：导入与基础语法检查

证据：`plugins/agent-sdk-dev/commands/new-sdk-app.md:68,83-87,117-126,163-166`。

### 5) 与 verifier agent 的协议衔接

`new-sdk-app` 不直接定义 verifier 细则，而是在 Verification 阶段路由到子协议：

- TypeScript -> `agent-sdk-verifier-ts`
- Python -> `agent-sdk-verifier-py`

证据：`plugins/agent-sdk-dev/commands/new-sdk-app.md:128-135`。  
下游协议要求：

- TS verifier 明确再执行一次 `npx tsc --noEmit` 并输出 PASS/WARN/FAIL 报告（`plugins/agent-sdk-dev/agents/agent-sdk-verifier-ts.md:39,101-116`）。
- Py verifier 聚焦导入/语法/SDK 模式与文档对齐并输出同结构报告（`plugins/agent-sdk-dev/agents/agent-sdk-verifier-py.md:95-110`）。

### 6) 配置与产物约束

命令明确需要生成或检查这些配置/文件：

- TypeScript：`package.json`（含 `type: "module"`、scripts、typecheck）+ `tsconfig.json`
- Python：`requirements.txt` 或 `pyproject`/poetry 路线
- 环境安全：`.env.example` 含 `ANTHROPIC_API_KEY`，并将 `.env` 纳入 `.gitignore`
- 可选插件化结构：`.claude/`（agents/commands/settings）

证据：`plugins/agent-sdk-dev/commands/new-sdk-app.md:64-105`。

## 关键代码路径与文件引用

以下路径构成 `new-sdk-app.md` 的完整上下文依赖图：

1. 主体命令协议  
   `plugins/agent-sdk-dev/commands/new-sdk-app.md:1-176`

2. 直接调用方（用户入口文档与插件目录索引）  
   `plugins/agent-sdk-dev/README.md:11-48,117-148`  
   `plugins/README.md:13-16`

3. 被调用方（验证子代理）  
   `plugins/agent-sdk-dev/agents/agent-sdk-verifier-ts.md:1-145`  
   `plugins/agent-sdk-dev/agents/agent-sdk-verifier-py.md:1-140`

4. 插件注册与装载入口  
   `plugins/agent-sdk-dev/.claude-plugin/plugin.json:1-9`  
   `.claude-plugin/marketplace.json:12-16`

5. 命令协议规范参考（frontmatter/命令自动发现）  
   `plugins/plugin-dev/skills/plugin-structure/SKILL.md:110-115`  
   `plugins/plugin-dev/skills/command-development/references/frontmatter-reference.md:5-20,196-236`

6. 测试与脚本现状（缺失项）  
   `plugins/agent-sdk-dev` 目录仅 5 个文档/元数据文件，无本地测试与自动化脚本（`find plugins/agent-sdk-dev -maxdepth 4 -type f` 结果）。

## 依赖与外部交互

### 内部依赖

- 插件发现依赖：marketplace 将 `agent-sdk-dev` 指向 `./plugins/agent-sdk-dev`（`.claude-plugin/marketplace.json:12-16`）。
- 命令发现依赖：`commands/*.md` 自动加载约定（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:112-115`）。
- 验证依赖：`new-sdk-app` 将验收职责分发到 TS/Py verifier（`plugins/agent-sdk-dev/commands/new-sdk-app.md:128-135`）。
- 文档承诺依赖：README 对外承诺“自动验证”，与命令协议一致性必须保持（`plugins/agent-sdk-dev/README.md:22,69,103,134`）。

### 外部交互

- 文档服务：`docs.claude.com`（overview/typescript/python 及相关 guides）  
  证据：`plugins/agent-sdk-dev/commands/new-sdk-app.md:12-23`。
- 包管理生态：npm 与 PyPI 最新版查询及安装  
  证据：`plugins/agent-sdk-dev/commands/new-sdk-app.md:74-87`。
- 开发者平台：API key 引导到 `https://console.anthropic.com/`  
  证据：`plugins/agent-sdk-dev/commands/new-sdk-app.md:98-101`。
- 本地 shell 执行：`npm/pip/npx` 等命令依赖用户机器环境与网络可达性。

## 风险、边界与改进建议

### 主要风险

1. 在线依赖风险  
   文档读取与“最新版本”校验都依赖外网；离线或受限网络会导致流程降级。  
   证据：`plugins/agent-sdk-dev/commands/new-sdk-app.md:10-25,74-79`。

2. Python 验收可重复性弱  
   命令只写“导入与语法检查”，但未给固定执行模板（如 `python -m py_compile`），不同执行者可能标准不一。  
   证据：`plugins/agent-sdk-dev/commands/new-sdk-app.md:123-125`。

3. 命令与 README 双维护漂移风险  
   README 写“自动验证/语法验证”，命令与 verifier 写的是可执行细节；若更新不同步，用户预期会偏移。  
   证据：`plugins/agent-sdk-dev/README.md:21-23,134`，`plugins/agent-sdk-dev/commands/new-sdk-app.md:128-135`。

4. 缺少自动化回归  
   当前目录没有 smoke test 或 lint 规则去验证命令协议的关键条款（如“单问约束”“版本检查步骤”）是否被改坏。

### 边界说明

- 该文件是“提示词流程协议”，不是可执行脚本；执行质量依赖 Claude 在会话中的遵循程度。
- 不负责深度业务逻辑生成，只负责项目初始化与基本可运行保证。
- 不覆盖部署流水线、测试框架搭建、CI 配置等工程化外围能力。

### 改进建议

1. 为 Python 分支补充最小可复现验证命令模板（例如 `python -m py_compile` + import smoke check），与 TS 分支同级别可验证。
2. 在 `agent-sdk-dev` 增加轻量 smoke-test 文档/脚本，对命令协议关键段落做静态断言（例如必须包含 verifier 路由与版本检查条款）。
3. 在命令中增加“网络失败回退策略”段落（无法访问文档或 npm/PyPI 时如何告知用户并安全继续）。
4. 将 README 的“特性承诺”改为引用命令中的锚点或生成式片段，降低双写漂移。
