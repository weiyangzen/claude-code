# DIR `Script` 研究文档

## 场景与职责

`Script/` 目录当前是仓库内唯一的 Windows PowerShell 启动入口目录，包含 1 个脚本：`run_devcontainer_claude_code.ps1`。它的职责不是构建业务功能，而是把“本机容器后端初始化 + DevContainer 拉起 + 进入容器执行 Claude”串成一次可重复的操作流，减少 Windows 用户手工执行多条命令的复杂度。

在仓库整体中的定位：

1. 上游调用方（谁触发它）
- 主要是人工从项目根目录执行：`./Script/run_devcontainer_claude_code.ps1 -Backend docker|podman`。
- 仓库内未发现 GitHub Actions、其他 shell 脚本或 npm/bun 脚本对该文件的直接调用；属于“手工入口脚本”。

2. 下游被调用方（它调用谁）
- 主机侧命令：`docker` / `podman` / `devcontainer`。
- 容器侧命令：`zsh -c 'claude; exec zsh'`。
- 间接依赖 `.devcontainer/devcontainer.json`、`.devcontainer/Dockerfile`、`.devcontainer/init-firewall.sh` 来决定容器构建与启动行为。

3. 与同类目录的边界
- `scripts/` 主要是 CI/自动化脚本（Bash/TS）；`Script/` 专门承载 Windows DevContainer 人工启动脚本。

## 功能点目的

### 1. 后端选择与参数约束
- 通过 `param` + `ValidateSet('docker','podman')` 把输入限制为两种后端，避免无效参数导致执行路径不明确。

### 2. 前置依赖快速失败（fail fast）
- 用 `Get-Command` 检查后端命令与 `devcontainer` CLI 是否可用，缺失即退出，防止后续步骤在更深层失败。

### 3. 后端差异收敛
- `podman` 分支负责 `machine init/start` 与默认连接切换。
- `docker` 分支只验证 daemon 可达（`docker info`）。
- 两者最终都汇聚到 `devcontainer up --workspace-folder .`。

### 4. 从“容器已启动”过渡到“可工作交互态”
- 脚本在 `devcontainer up` 后，用 `ps --filter label=devcontainer.local_folder=<cwd>` 定位当前工程容器 ID，再 `exec -it` 进入容器并先执行 `claude`，最后落入交互 `zsh`。

### 5. 提供可诊断错误信息
- 关键步骤都被 `try/catch` 包裹；失败时附带执行命令上下文或缺失条件，方便用户定位问题。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 关键流程（按执行时序）

1. 参数解析
- 输入：`-Backend docker|podman`（必填）。
- 机制：PowerShell `ValidateSet` 在进入业务逻辑前做参数合法性拦截。

2. 依赖检查
- 命令：`Get-Command <backend>`、`Get-Command devcontainer`。
- 处理：若任一命令不可用，抛错并 `exit 1`。

3. 后端初始化
- Podman 路径：
  - `podman machine init claudeVM`
  - `podman machine start claudeVM -q`
  - `podman system connection default claudeVM`
- Docker 路径：
  - `docker info`（验证 Docker Desktop/daemon 可用）

4. 启动 DevContainer
- 统一命令骨架：`devcontainer up --workspace-folder .`
- Podman 增量参数：`--docker-path podman`。

5. 发现容器 ID
- 使用后端 `ps`，筛选 label：
  - `label=devcontainer.local_folder=<当前工作目录绝对路径>`
- 输出格式：`--format '{{.ID}}'`。

6. 进入容器工作会话
- 命令：`<backend> exec -it <containerId> zsh -c 'claude; exec zsh'`
- 语义：先执行 `claude`，退出后保留在 zsh，维持交互环境。

### 2) 关键“数据结构/协议”

1. 参数约束协议
- `Backend` 仅允许字符串枚举 `docker|podman`。

2. DevContainer 发现协议
- 依赖 `devcontainer` 在容器上写入的 label `devcontainer.local_folder`。
- 通过“当前目录绝对路径 == label 值”进行容器归属匹配。

3. 命令构造策略
- 避免 `Invoke-Expression`，改用调用运算符 `&` + 参数数组（历史提交已演进到该方式），降低命令注入与转义错误风险。

### 3) 与配置文件的配合关系

1. `.devcontainer/devcontainer.json`
- `workspaceFolder` 为 `/workspace`，`workspaceMount` 绑定本地工作区。
- `postStartCommand` 执行 `sudo /usr/local/bin/init-firewall.sh`，意味着 `devcontainer up` 成功不仅取决于镜像构建，还取决于启动后防火墙脚本执行成功。

2. `.devcontainer/Dockerfile`
- 安装 `zsh` 与 `@anthropic-ai/claude-code`，为脚本最后一步 `zsh -c 'claude; exec zsh'` 提供前提。

3. `.devcontainer/init-firewall.sh`
- 启动阶段会配置 `iptables/ipset` 并验证可达性，若失败会影响容器可用状态，间接导致脚本后续“查不到容器 ID”或进入失败。

### 4) 历史演进（从提交看设计意图）

- `11cfc05`：初版引入脚本（Windows + Docker/Podman 双后端）。
- `545d78c`：改为注释帮助块 + `ValidateSet`，提升参数与文档可读性。
- `c93c724`：从 `Invoke-Expression` 改为 `&` 调用，参数处理更稳健。
- `9285dfb` / `10a1f7d`：补齐并强化前置依赖检查，错误信息更明确。

## 关键代码路径与文件引用

### 目标目录与主脚本
- `Script/run_devcontainer_claude_code.ps1`

### 关键上下文配置（被该脚本间接消费）
- `.devcontainer/devcontainer.json`
- `.devcontainer/Dockerfile`
- `.devcontainer/init-firewall.sh`

### 文档与流程上下文
- `README.md`（仓库级使用说明，未直接收录该脚本入口）
- `Docs/researches/current_folder_research.md`（根目录研究中标注了该链路）

### 研究流程/脚本上下文
- `.ops/generate_research_blueprint_checklist.sh`
- `.ops/generate_daily_research_todo.sh`
- `Docs/researches/blueprint_checklist.md`（含 `DIR Script` 与 `FILE Script/run_devcontainer_claude_code.ps1` 状态）

### 调用链（调用方 -> 被调用方）
1. 人工 PowerShell 会话 -> `Script/run_devcontainer_claude_code.ps1`
2. `run_devcontainer_claude_code.ps1` -> `docker|podman` / `devcontainer`
3. `devcontainer up` -> 读取 `.devcontainer/devcontainer.json` -> 构建 `.devcontainer/Dockerfile` -> 执行 `postStartCommand` (`init-firewall.sh`)
4. `run_devcontainer_claude_code.ps1` -> `<backend> exec ... zsh -c 'claude; exec zsh'`

## 依赖与外部交互

### 本地依赖
- PowerShell（脚本执行环境）
- `devcontainer` CLI
- `docker` 或 `podman`（二选一）
- 容器内 `zsh` 与 `claude`（由 Dockerfile 提供）

### 外部交互
- 脚本本身不直接调用 HTTP API，但通过 `devcontainer up` 间接触发镜像拉取、npm 安装等网络访问。
- `init-firewall.sh` 会请求 `https://api.github.com/meta` 并解析多个域名 DNS，属于容器启动阶段外部交互。

### 权限与环境约束
- Docker 分支要求本机 Docker daemon 可访问。
- Podman 分支要求可创建/启动 `claudeVM` machine 并允许切换默认连接。
- `exec -it` 需要交互终端；在非交互执行器中可能失败。

### 测试现状
- 仓库未发现该脚本的自动化测试（无 Pester 用例）。
- 当前质量保证主要依赖：人工执行验证 + 提交历史中的渐进改进。

## 风险、边界与改进建议

### 风险

1. 容器 ID 匹配歧义
- 现状：`ps --filter label=devcontainer.local_folder=<cwd>` 可能返回多行（历史残留容器/路径大小写差异）。
- 影响：`$containerId` 可能不是单一 ID，导致 `exec` 失败或命中非预期容器。

2. Podman 初始化幂等性不稳定
- 现状：`podman machine init claudeVM` 在“已存在”场景下的退出码依赖版本/平台实现。
- 影响：脚本可能在“可继续”场景直接退出。

3. 根目录假设未显式校验
- 现状：脚本注释要求在项目根执行，但未检测 `.devcontainer/devcontainer.json` 是否存在。
- 影响：错误路径下会在后续步骤报错，定位成本偏高。

4. 强依赖容器内 zsh/claude
- 现状：最后一步固定 `zsh -c 'claude; exec zsh'`。
- 影响：若镜像变更或用户替换 devcontainer 基础镜像，可能无法进入会话。

5. 自动化回归覆盖缺失
- 现状：无单测/集成测试。
- 影响：PowerShell 兼容性与参数边界回归难以及时发现。

### 边界

- 该目录当前仅面向 Windows/PowerShell DevContainer 启动体验，不负责 Linux/macOS 本地入口统一。
- 脚本不管理 `.devcontainer` 配置正确性，只在执行链路中消费其结果。
- 脚本不提供容器生命周期清理（如 stop/rm/prune），仅负责“启动并进入”。

### 改进建议

1. 增加前置目录校验
- 启动前检查 `.devcontainer/devcontainer.json` 存在性，不满足时给出明确错误与示例命令。

2. 强化容器 ID 选择策略
- 对多 ID 场景给出显式决策（最新容器优先/交互选择），避免直接拼接多行字符串。

3. 提供可配置参数
- 增加可选参数：`-PodmanMachineName`、`-Shell`、`-SkipClaude`，降低脚本与当前镜像实现的耦合。

4. 抽离可测试函数并引入 Pester
- 把“依赖检查、命令参数构造、容器 ID 解析”拆为函数并补最小测试，形成可回归基础。

5. 补充仓库文档入口
- 在 `README.md` 或独立文档增加“Windows DevContainer 一键启动”章节，降低脚本发现成本。
