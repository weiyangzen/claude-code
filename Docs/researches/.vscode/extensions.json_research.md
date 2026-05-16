# FILE `.vscode/extensions.json` 研究文档

## 场景与职责

`.vscode/extensions.json` 是仓库面向 VS Code 本地工作区的扩展推荐清单。它不参与业务运行时逻辑，但直接影响开发者第一次打开仓库时的工具链完整度与一致性。

从调用关系看：
- 调用方：VS Code 工作区加载流程（打开仓库时读取 `.vscode/extensions.json`）。
- 被调用方：无仓内可执行脚本直接读取该文件；其“被消费者”是 VS Code 的扩展推荐机制。

从上下文角色看，它与 `.devcontainer/devcontainer.json` 中 `customizations.vscode.extensions` 共同定义 IDE 体验，但作用域不同：
- `.vscode/extensions.json`：宿主机本地工作区推荐。
- `.devcontainer/devcontainer.json`：容器工作区推荐与编辑器行为约束。

## 功能点目的

当前文件仅包含一个 `recommendations` 数组（4 个扩展），目的如下：

1. `dbaeumer.vscode-eslint`
- 提供 ESLint 诊断与修复入口，降低风格漂移和低级语法问题遗漏。

2. `esbenp.prettier-vscode`
- 提供统一格式化能力；与 DevContainer 的默认格式化器设置保持一致预期。

3. `ms-vscode-remote.remote-containers`
- 为“在容器中打开仓库”提供入口能力，与仓库的 `.devcontainer` 方案形成闭环。

4. `eamodio.gitlens`
- 强化 blame/历史追踪能力，提升调试与代码审阅效率。

整体定位是“最小核心开发体验集合”，不是完整 IDE 治理文件（例如未包含 `unwantedRecommendations`、未做版本锁定）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 数据结构

文件结构是标准 JSON 对象：

```json
{
  "recommendations": [
    "publisher.extension"
  ]
}
```

实现特征：
- 扩展 ID 采用 `publisher.extensionName` 协议。
- 只声明建议安装，不声明版本。
- 未使用 `unwantedRecommendations`，因此没有显式排除列表。

### 2) 关键流程 A：本地 VS Code 工作区加载

1. 用户在 VS Code 打开仓库根目录。
2. VS Code 读取 `.vscode/extensions.json`。
3. VS Code 对比本机已装扩展与 `recommendations` 列表。
4. 对缺失项显示推荐安装提示（可拒绝，非强制）。

结论：该链路是“编辑器侧被动消费”，仓库内没有 CLI/脚本命令直接触发该文件解析。

### 3) 关键流程 B：与 DevContainer 流程协同

1. `Script/run_devcontainer_claude_code.ps1` 执行 `devcontainer up --workspace-folder .`。
2. DevContainer 读取 `.devcontainer/devcontainer.json`。
3. 容器内 VS Code 使用 `customizations.vscode.extensions/settings` 建立扩展与格式化行为。
4. 若开发者先在宿主机用 `ms-vscode-remote.remote-containers` 打开容器，该流程更顺滑。

这意味着 `.vscode/extensions.json` 是容器开发体验的“入口增强层”，`.devcontainer/devcontainer.json` 是容器体验的“行为定义层”。

### 4) 扩展集合一致性现状

`.vscode/extensions.json` 与 `.devcontainer/devcontainer.json` 的集合对比：
- 交集：`vscode-eslint`、`prettier-vscode`、`gitlens`
- 仅本地推荐：`ms-vscode-remote.remote-containers`
- 仅容器推荐：`anthropic.claude-code`

该差异反映了当前策略：本地强调容器入口，容器内强调 Claude 扩展与保存时格式修复。

## 关键代码路径与文件引用

1. 目标文件
- `.vscode/extensions.json:1-8`

2. 直接协同配置
- `.devcontainer/devcontainer.json:16-23`（容器内推荐扩展）
- `.devcontainer/devcontainer.json:24-29`（保存时格式化与 ESLint 修复）

3. 启动脚本与命令链
- `Script/run_devcontainer_claude_code.ps1:107-115`（`devcontainer up --workspace-folder .`）
- `Script/run_devcontainer_claude_code.ps1:126-127`（通过 `label=devcontainer.local_folder` 定位容器）
- `Script/run_devcontainer_claude_code.ps1:141-144`（进入容器后执行 `claude`）

4. 扩展下载相关网络白名单
- `.devcontainer/init-firewall.sh:67-75`（`marketplace.visualstudio.com`、`vscode.blob.core.windows.net`、`update.code.visualstudio.com`）

5. 研究流程相关文档/脚本
- `Docs/researches/.vscode/current_folder_research.md:3-153`（目录级研究背景）
- `.ops/generate_research_blueprint_checklist.sh:27-42`（扫描对象生成 checklist）
- `.ops/generate_daily_research_todo.sh:15-18,33-38`（按 checklist 输出当日 pending）

## 依赖与外部交互

1. 运行依赖（本地）
- VS Code（工作区推荐机制）。
- 用户是否安装推荐扩展由人工选择决定。

2. 仓内配置依赖
- `.devcontainer/devcontainer.json`：容器内扩展与编辑器设置。
- `Script/run_devcontainer_claude_code.ps1`：容器拉起命令链与开发入口。

3. 外部交互
- 扩展安装依赖 VS Code Marketplace/CDN（由防火墙白名单放行）。
- 网络失败会导致推荐扩展无法安装或更新。

4. 测试与校验现状
- 未发现针对 `.vscode/extensions.json` 的自动化测试、schema 校验或 CI 一致性检查。
- 当前正确性依赖手工打开 VS Code 时的运行反馈。

## 风险、边界与改进建议

1. 风险：双清单漂移
- 现状：`.vscode/extensions.json` 与 `.devcontainer/devcontainer.json` 各自维护扩展列表。
- 影响：本地与容器环境的能力可能长期偏离。
- 建议：引入单一来源配置（例如 `extensions.base.json`）并脚本生成两处目标文件，CI 校验差异。

2. 风险：推荐非强制
- 现状：`recommendations` 仅提示安装。
- 影响：团队成员可能缺失 ESLint/Prettier，导致提交风格不一致。
- 建议：在 `README` 或贡献指南增加“最低 IDE 依赖清单”和自检步骤。

3. 风险：核心扩展在本地场景缺失
- 现状：容器内推荐 `anthropic.claude-code`，本地清单未推荐。
- 影响：不使用 DevContainer 的 VS Code 用户可能错过核心扩展。
- 建议：评估是否把 `anthropic.claude-code` 加入 `.vscode/extensions.json`，或在文档里明确这是有意分层策略。

4. 风险：扩展安装链路受网络策略影响
- 现状：容器网络初始化脚本对 DNS 和特定域名可达性敏感。
- 影响：扩展安装失败可能造成“配置存在但体验缺失”。
- 建议：对扩展域名解析失败增加重试/降级日志，提供显式排障指引。

5. 边界说明
- 该文件只影响编辑器体验，不直接改变 CLI 行为、构建产物或运行时安全策略。
- 它的价值主要体现在研发效率和协作一致性，而非业务功能正确性。
