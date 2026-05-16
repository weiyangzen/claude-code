# DIR `.vscode` 研究文档

## 场景与职责

`.vscode` 目录在本仓库中只包含一个文件：`.vscode/extensions.json`。它的职责不是运行时逻辑，而是“工作区开发体验约束”。具体职责如下：

1. 为以 VS Code 打开仓库的开发者提供统一扩展推荐，降低初次进入项目时的环境差异。
2. 与 `.devcontainer/devcontainer.json` 中的 `customizations.vscode` 形成互补：
- `.vscode/extensions.json` 作用于“本地工作区”。
- `.devcontainer/devcontainer.json` 作用于“容器内工作区”。
3. 被研究自动化流程纳入对象清单（`Docs/researches/blueprint_checklist.md`），用于持续化仓库知识沉淀。

边界上，`.vscode` 目录当前不承载 `settings.json`、`tasks.json`、`launch.json`，因此不直接控制编译/调试/任务，只负责扩展推荐。

## 功能点目的

`.vscode/extensions.json` 当前推荐 4 个扩展（`recommendations[]`）：

1. `dbaeumer.vscode-eslint`
- 目的：在编辑器内对 JS/TS 等文件提供 ESLint 诊断与修复入口。

2. `esbenp.prettier-vscode`
- 目的：统一格式化行为；与 DevContainer 的 `editor.defaultFormatter` 配置形成一致预期。

3. `ms-vscode-remote.remote-containers`
- 目的：支持通过 Dev Containers 打开项目容器环境，配合 `.devcontainer/devcontainer.json` 生效。

4. `eamodio.gitlens`
- 目的：增强 Git 历史/责任归属可见性，提升代码追踪效率。

设计意图是“最小但高频”的编辑体验增强集合，而非完整的团队 IDE 策略。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 数据结构

`.vscode/extensions.json` 使用简单 JSON 对象：

```json
{
  "recommendations": ["publisher.extension", "..."]
}
```

关键特征：
- 仅声明扩展标识，不包含版本锁定。
- 标识格式为 `publisher.extensionName`。
- 本仓库未定义 `unwantedRecommendations`（即未显式排除扩展）。

### 2) 关键流程 A：本地 VS Code 打开仓库

1. 用户在 VS Code 中打开仓库目录。
2. VS Code 读取 `.vscode/extensions.json`。
3. VS Code 比对本地已安装扩展和 `recommendations[]`。
4. 对缺失扩展在 UI 中提示安装（“推荐”语义，不是强制安装）。

这个链路的调用方是 VS Code 客户端，仓库内没有 shell/TS/Python 脚本直接读取该文件。

### 3) 关键流程 B：DevContainer 启动与扩展联动

1. `Script/run_devcontainer_claude_code.ps1` 执行 `devcontainer up --workspace-folder .`。
2. DevContainer 读取 `.devcontainer/devcontainer.json`。
3. `customizations.vscode.extensions` 在容器工作区安装/建议扩展。
4. `customizations.vscode.settings` 设置默认格式化器与 ESLint on-save 行为。

相关命令证据（脚本内实际执行）：
- `devcontainer up --workspace-folder .`
- `docker|podman exec -it <containerId> zsh -c 'claude; exec zsh'`

### 4) 配置协同细节

`.vscode/extensions.json` 与 `.devcontainer/devcontainer.json` 的扩展集合存在交集与差异：

- 共同项：`vscode-eslint`、`prettier-vscode`、`gitlens`
- 仅 `.vscode/extensions.json`：`ms-vscode-remote.remote-containers`
- 仅 `.devcontainer/devcontainer.json`：`anthropic.claude-code`

这说明当前策略是“本地工作区强调容器入口扩展，容器内强调 Claude 扩展与编辑器行为设置”。

### 5) 网络/协议侧实现约束

`.devcontainer/init-firewall.sh` 将以下域名加入白名单：
- `marketplace.visualstudio.com`
- `vscode.blob.core.windows.net`
- `update.code.visualstudio.com`

这为容器内拉取 VS Code 扩展提供网络通路；若该脚本失败或域名解析失败，扩展安装链路会受阻。

## 关键代码路径与文件引用

1. 目标对象
- `.vscode/extensions.json:1-8`（目录核心实现，推荐扩展列表）

2. 直接协同配置
- `.devcontainer/devcontainer.json:16-41`
  - `customizations.vscode.extensions`
  - `customizations.vscode.settings.editor.defaultFormatter`
  - `customizations.vscode.settings.editor.codeActionsOnSave`

3. DevContainer 启动调用链
- `Script/run_devcontainer_claude_code.ps1:107-115`（`devcontainer up`）
- `Script/run_devcontainer_claude_code.ps1:141-144`（容器内执行 `claude`）

4. 扩展网络依赖路径
- `.devcontainer/init-firewall.sh:67-75`（允许 VS Code Marketplace/更新域名）

5. 研究自动化上下文（间接依赖）
- `.ops/generate_research_blueprint_checklist.sh:27-42`（扫描目录/文件并生成 checklist）
- `.ops/research_guard.sh:174-199`（从 checklist 取 pending 项并为 DIR 生成 `current_folder_research.md`）

## 依赖与外部交互

1. 本地依赖
- VS Code（读取 `.vscode/extensions.json`）
- Dev Containers CLI / 扩展（读取 `.devcontainer/devcontainer.json`）

2. 外部服务
- VS Code Marketplace 与扩展下载域名（见 `init-firewall.sh` 白名单）

3. 与仓内其它模块的交互关系
- 与 `.devcontainer` 强相关（扩展集合与编辑器行为设置联动）
- 与 `.ops` 研究流水线弱耦合（被纳入 checklist/todo 管理）
- 与业务脚本、插件运行逻辑无直接调用关系

4. 测试现状
- 仓库内无针对 `.vscode/extensions.json` 的自动化测试或 schema 校验脚本。
- 正确性主要依赖 VS Code 打开工作区时的运行时反馈。

## 风险、边界与改进建议

1. 风险：双源扩展清单漂移
- 现状：`.vscode/extensions.json` 与 `.devcontainer/devcontainer.json` 都维护扩展集合。
- 影响：长期可能出现本地与容器体验不一致。
- 建议：引入“单一来源 + 生成”策略（例如脚本生成两处配置），并在 CI 做一致性检查。

2. 风险：推荐语义非强制
- 现状：`recommendations` 只提示，不保证用户安装。
- 影响：代码风格、lint 反馈、Git 可视化能力可能不一致。
- 建议：在 README 或贡献文档补充“最低开发环境要求”与安装校验命令。

3. 风险：扩展集合与项目定位存在轻微割裂
- 现状：DevContainer 推荐含 `anthropic.claude-code`，但 `.vscode/extensions.json` 未包含该项。
- 影响：不使用容器的 VS Code 用户可能错过核心扩展。
- 建议：评估是否将 `anthropic.claude-code` 同步到 `.vscode/extensions.json`，或在文档中说明差异为“有意设计”。

4. 风险：网络白名单失败导致扩展链路不可用
- 现状：`init-firewall.sh` 依赖 DNS 解析和外部服务可达性。
- 影响：容器启动阶段可能因域名解析失败而中断，从而影响扩展安装与开发体验。
- 建议：增加 DNS/域名失败的重试与降级策略，并在日志中输出更明确的故障指引。

5. 边界说明
- `.vscode` 当前是“编辑器建议层”，不是运行时功能层。
- 对项目功能正确性的直接影响较小，但对开发效率、代码风格一致性、上手成本有显著影响。
