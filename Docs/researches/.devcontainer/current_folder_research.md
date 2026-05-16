# DIR `.devcontainer` 研究文档

## 场景与职责

`.devcontainer` 是本仓库的“可复现开发沙箱定义层”，负责把 Claude Code 的运行环境标准化，并在容器启动后立即施加网络出站白名单控制。该目录包含 3 个核心对象：

1. `devcontainer.json`：声明容器构建参数、运行权限、挂载卷、VS Code 定制、启动后命令。  
2. `Dockerfile`：定义镜像内容（Node 20、CLI 工具、zsh 环境、Claude CLI、防火墙脚本部署）。  
3. `init-firewall.sh`：在容器 `postStart` 阶段执行的网络收敛脚本（iptables + ipset）。

从仓库上下文看，它同时被 `Script/run_devcontainer_claude_code.ps1` 作为启动入口依赖，形成“主机调用 devcontainer CLI -> 构建并启动容器 -> 容器内执行防火墙收敛 -> 进入 claude 交互 shell”的完整链路。

## 功能点目的

### 1) 一键复现 Claude Code 开发环境

- `devcontainer.json` 通过 `build.args` 注入时区与版本参数（`CLAUDE_CODE_VERSION`、`GIT_DELTA_VERSION`、`ZSH_IN_DOCKER_VERSION`），目标是让环境参数可显式控制。  
- `Dockerfile` 安装 `@anthropic-ai/claude-code`、`gh`、`jq`、`iptables/ipset/dig` 等工具，目标是让“编码 + GitHub 操作 + 网络控制”在同一镜像内可用。

### 2) 限制容器网络出站面

- `devcontainer.json` 使用 `postStartCommand: sudo /usr/local/bin/init-firewall.sh`，并设置 `waitFor: postStartCommand`，确保工具会话开始前网络策略已生效。  
- `init-firewall.sh` 通过白名单机制仅放行 GitHub 网段和少量域名（npm、Anthropic API、Statsig、VS Code Marketplace/CDN），默认拒绝其他出站流量。

### 3) 保持用户态可用性与会话持久化

- 通过 `mounts` 持久化 `/commandhistory` 与 `/home/node/.claude`，减少容器重建导致的历史丢失。  
- 以 `remoteUser: node` 运行日常命令，同时通过 sudoers 仅放开 `init-firewall.sh` 的免密 root 执行，避免长期 root 会话。

### 4) 提供跨后端启动入口（Windows）

- `Script/run_devcontainer_claude_code.ps1` 对 Docker/Podman 分支初始化后统一执行 `devcontainer up --workspace-folder .`。  
- 启动成功后按 `label=devcontainer.local_folder=<cwd>` 查询容器并执行 `claude; exec zsh`，把环境与 CLI 入口串联。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 启动关键流程

1. 主机侧执行：
- `powershell .\Script\run_devcontainer_claude_code.ps1 -Backend docker|podman`

2. 脚本流程：
- 检查 `$Backend` 与 `devcontainer` 命令存在。  
- Podman 分支：`podman machine init/start/default`。  
- 统一执行：`devcontainer up --workspace-folder .`（Podman 额外 `--docker-path podman`）。

3. 容器侧流程：
- 构建镜像（`Dockerfile`）。  
- 启动后执行 `sudo /usr/local/bin/init-firewall.sh`。  
- 防火墙初始化成功后进入可交互状态。

4. 回连会话：
- 使用 `docker|podman ps --filter "label=devcontainer.local_folder=<path>"` 找容器 ID。  
- 执行 `docker|podman exec -it <id> zsh -c 'claude; exec zsh'`。

### B. 配置数据结构（`devcontainer.json`）

1. 构建参数结构：
- `build.dockerfile = "Dockerfile"`  
- `build.args = { TZ, CLAUDE_CODE_VERSION, GIT_DELTA_VERSION, ZSH_IN_DOCKER_VERSION }`

2. 运行权限与挂载：
- `runArgs`: `NET_ADMIN`, `NET_RAW`（允许网络规则管理）。  
- `mounts`: 两个基于 `${devcontainerId}` 的命名卷（历史与配置）。  
- `workspaceMount`: 绑定本地仓库到 `/workspace`。

3. 会话环境：
- `containerEnv`: `NODE_OPTIONS`, `CLAUDE_CONFIG_DIR`, `POWERLEVEL9K_DISABLE_GITSTATUS`。  
- `remoteUser = node`，默认 shell 为 zsh（由 Dockerfile 环境变量与 zsh-in-docker 配置支撑）。

### C. 防火墙脚本协议与命令序列（`init-firewall.sh`）

1. 重置并最小恢复：
- 先缓存 Docker DNS nat 规则（匹配 `127.0.0.11`），再 flush 各表与旧 ipset。  
- 仅恢复 Docker DNS 相关 nat 规则，保障后续域名解析。

2. 白名单集合构建：
- 创建 `ipset allowed-domains hash:net`。  
- 调用 `curl https://api.github.com/meta` 获取 `.web/.api/.git` 网段。  
- 通过 `jq` + `aggregate -q` 去重聚合后写入 ipset。  
- 对固定域名列表执行 `dig A` 解析并写入 ipset。

3. 规则收敛：
- 先放行 DNS、SSH、loopback、主机所在 `/24` 网段。  
- 设置默认策略 `INPUT/FORWARD/OUTPUT = DROP`。  
- 放行 `ESTABLISHED,RELATED`。  
- 放行 `OUTPUT` 到 `allowed-domains`。  
- 其他出站统一 `REJECT --reject-with icmp-admin-prohibited`。

4. 启动时自检：
- 访问 `https://example.com` 必须失败。  
- 访问 `https://api.github.com/zen` 必须成功。  
- 任一条件不满足即 `exit 1`，导致 postStart 阶段失败。

### D. 镜像实现细节（`Dockerfile`）

1. 基础镜像与工具层：`FROM node:20` + apt 安装开发工具、网络控制工具、GitHub CLI。  
2. 用户与目录：创建 `/workspace`、`/home/node/.claude`、`/commandhistory` 并授权 `node`。  
3. 体验层：安装 `git-delta`、`zsh-in-docker`，设置 `SHELL/EDITOR/VISUAL`。  
4. 业务层：`npm install -g @anthropic-ai/claude-code@${CLAUDE_CODE_VERSION}`。  
5. 权限桥接：拷贝防火墙脚本到 `/usr/local/bin`，在 sudoers 中仅授予该脚本 NOPASSWD 执行。

## 关键代码路径与文件引用

### 目标目录内

- `.devcontainer/devcontainer.json`  
- `.devcontainer/Dockerfile`  
- `.devcontainer/init-firewall.sh`

### 调用方（上游）

- `Script/run_devcontainer_claude_code.ps1`  
  - 参数与前置检查：第 29-56 行。  
  - `devcontainer up` 调用：第 107-119 行。  
  - 用 `devcontainer.local_folder` 标签查容器并 `exec`：第 121-149 行。

### 被调用/协作对象（下游）

- 容器运行时命令：`devcontainer` CLI、`docker|podman`、`zsh`、`claude`。  
- 网络脚本依赖的系统命令：`iptables`, `iptables-save`, `ipset`, `curl`, `jq`, `aggregate`, `dig`, `ip`, `sed`, `awk`。  
- IDE 侧协作：`.vscode/extensions.json` 推荐 `ms-vscode-remote.remote-containers`，与 devcontainer 使用场景匹配。

## 依赖与外部交互

### 外部网络依赖

1. 镜像构建阶段：
- `deb.debian.org`（apt 包）  
- `github.com`（下载 `git-delta` 与 `zsh-in-docker`）  
- `registry.npmjs.org`（安装 Claude CLI）

2. 启动后防火墙初始化阶段：
- `api.github.com/meta`（GitHub IP 元数据）  
- 白名单域名 DNS 解析与连接：`api.anthropic.com`、`statsig.anthropic.com`、`statsig.com`、`marketplace.visualstudio.com`、`vscode.blob.core.windows.net`、`update.code.visualstudio.com` 等。

### 本地/宿主依赖

- 需要 Docker 或 Podman + Dev Container CLI。  
- 需要容器具备 `NET_ADMIN/NET_RAW` 能力以写入 iptables/ipset。  
- Windows 入口脚本当前仅在 PowerShell 场景提供自动化串联。

### 测试与验证现状

- 未发现针对 `.devcontainer` 的独立自动化测试（无专门 test job/脚本）。  
- 当前验证机制主要是：`init-firewall.sh` 启动时内建连通性断言 + 人工进入容器后实测。

## 风险、边界与改进建议

### 风险

1. 启动可用性强依赖外网
- `postStart` 阶段必须访问 GitHub API 与多个域名 DNS；网络抖动会直接导致容器不可用。

2. DNS 白名单策略存在时效风险
- 当前只解析 A 记录并写入即时 IP；域名后端 IP 变化后，长生命周期容器可能出现误拒绝。

3. 规则重置影响面较大
- 脚本开头全量 flush 规则，尽管做了 Docker DNS 规则恢复，但对其他潜在链路较激进。

4. 主机网段推断过于简化
- 通过网关 IP 直接推导 `/24`，在非 `/24` 网络、复杂路由或 IPv6 场景可能不准确。

5. 镜像版本漂移
- `CLAUDE_CODE_VERSION` 默认 `latest`，重复构建可能得到不同行为。

### 边界

- `.devcontainer` 只定义“本仓库的开发容器体验”，不覆盖 CI 运行环境。  
- `Script/run_devcontainer_claude_code.ps1` 是 Windows 便捷入口，不是唯一启动方式。  
- 防火墙策略仅在容器内生效，不直接修改宿主机防火墙策略。

### 改进建议

1. 增加离线/降级模式
- 给 `init-firewall.sh` 增加“GitHub meta 获取失败时使用缓存快照”选项，避免全量失败。

2. 提升白名单策略鲁棒性
- 增加 AAAA 记录支持与定时刷新机制；或使用代理出口集中治理而非直接写 IP。

3. 收敛规则重置范围
- 改为仅管理自建链（例如 `CLAUDE_*` 链），减少对已有规则的干扰。

4. 固化版本可追踪性
- 将 `CLAUDE_CODE_VERSION` 从 `latest` 收敛到明确版本号，并在升级时显式变更。

5. 增加最小自动化校验
- 新增脚本 smoke test：验证 `devcontainer up` 后 `iptables -S` 含期望规则、并校验允许/拒绝域名连通性。
