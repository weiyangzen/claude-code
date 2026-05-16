# FILE `.devcontainer/Dockerfile` 研究文档

## 场景与职责

`.devcontainer/Dockerfile` 是 DevContainer 镜像的唯一构建定义文件，职责是把“Claude Code 可用开发环境 + 网络收敛前置依赖”固化为可重复构建的镜像层。它不直接运行业务逻辑，但决定了：

- 容器内是否具备 Claude CLI、GitHub CLI、网络控制工具。
- `devcontainer.json` 的 `postStartCommand`（`init-firewall.sh`）能否在最小权限模型下执行。
- `Script/run_devcontainer_claude_code.ps1` 进入容器后执行 `claude; exec zsh` 时，命令是否存在、交互体验是否可用。

在本仓库链路中，该文件是 `.devcontainer/devcontainer.json` 的 `build.dockerfile` 被调用方，也是 `.devcontainer/init-firewall.sh` 的部署上游（将脚本拷入 `/usr/local/bin` 并做 sudoers 授权）。

## 功能点目的

1. 提供统一基础运行时
- 以 `node:20` 为基础镜像，满足 `npm install -g @anthropic-ai/claude-code` 的运行前提。
- 通过 `ARG TZ` + `ENV TZ` 对齐容器时区到宿主或默认值。

2. 安装开发与网络控制工具
- 一次性安装 `git/gh/jq/fzf/zsh/vim/nano` 等开发工具。
- 安装 `iptables/ipset/iproute2/dnsutils/aggregate`，为 `init-firewall.sh` 提供命令依赖。

3. 维持非 root 默认开发体验
- 默认 `USER node`，并设置 `NPM_CONFIG_PREFIX=/usr/local/share/npm-global`，避免全局 npm 依赖写入系统受限目录。
- 创建 `/workspace`、`/home/node/.claude`、`/commandhistory` 并授权 `node`，配合 devcontainer 挂载卷实现配置/历史持久化。

4. 提升终端可用性与可读性
- 安装 `git-delta`（增强 `git diff` 可读性）。
- 通过 `zsh-in-docker` 配置 zsh + fzf 键绑定，统一默认 shell 与交互体验。

5. 构建最小提权桥接
- 把 `init-firewall.sh` 放到 `/usr/local/bin` 并仅授予 `node` 对该脚本的 `sudo NOPASSWD` 权限，而不是全面 root 权限。

## 具体技术实现（关键流程/数据结构/协议/命令）

1. 构建参数注入与环境变量初始化
- `ARG TZ` -> `ENV TZ="$TZ"`：允许上层 `devcontainer.json` 用 `${localEnv:TZ:...}` 注入时区。
- `ARG CLAUDE_CODE_VERSION=latest`：控制 CLI 安装版本。
- `ARG GIT_DELTA_VERSION`、`ARG ZSH_IN_DOCKER_VERSION`：控制附加工具版本。

2. 基础工具层安装
- 使用 `apt-get install --no-install-recommends` 安装最小必要包，最后 `apt-get clean && rm -rf /var/lib/apt/lists/*` 减少镜像层冗余。
- 关键包分组：
  - 开发：`git`, `gh`, `fzf`, `zsh`, `man-db`, `vim`, `nano`
  - 网络/防火墙：`iptables`, `ipset`, `iproute2`, `dnsutils`, `aggregate`, `jq`

3. 用户目录与历史持久化准备
- 创建 `/commandhistory/.bash_history`，并通过 `PROMPT_COMMAND='history -a'` 使历史即时落盘。
- 创建 `/workspace` 与 `/home/node/.claude`，供 `workspaceMount` 与 `CLAUDE_CONFIG_DIR` 对接。

4. 二进制工具安装路径
- `git-delta` 通过 `dpkg --print-architecture` 解析架构，再从 GitHub release 下载对应 `.deb` 安装。
- Claude CLI 通过 `npm install -g @anthropic-ai/claude-code@${CLAUDE_CODE_VERSION}` 安装至 `/usr/local/share/npm-global/bin`。

5. 交互环境设置
- `ENV SHELL=/bin/zsh`，并把 `EDITOR/VISUAL` 固定为 `nano`。
- 执行远程脚本 `zsh-in-docker.sh`，附加 fzf completion/key-bindings。

6. 权限桥接实现
- `COPY init-firewall.sh /usr/local/bin/`。
- 临时切换 `USER root` 执行：
  - `chmod +x /usr/local/bin/init-firewall.sh`
  - 写入 `/etc/sudoers.d/node-firewall`：`node ALL=(root) NOPASSWD: /usr/local/bin/init-firewall.sh`
- 再切回 `USER node`，保证默认开发会话保持非 root。

## 关键代码路径与文件引用

1. 当前文件关键段
- `.devcontainer/Dockerfile:1-6`：基础镜像与核心构建参数。
- `.devcontainer/Dockerfile:9-28`：依赖包安装（含防火墙脚本必需命令）。
- `.devcontainer/Dockerfile:36-47`：历史、工作目录、Claude 配置目录准备。
- `.devcontainer/Dockerfile:51-55`：`git-delta` 按架构下载安装。
- `.devcontainer/Dockerfile:81-82`：Claude CLI 全局安装。
- `.devcontainer/Dockerfile:85-91`：防火墙脚本部署与最小 sudo 授权。

2. 调用方与协作文件
- `.devcontainer/devcontainer.json:3-10`：通过 `build.args` 注入本文件 ARG。
- `.devcontainer/devcontainer.json:55`：启动后调用 `/usr/local/bin/init-firewall.sh`。
- `.devcontainer/init-firewall.sh:1-137`：运行时实际消费 `iptables/ipset/jq/dig/aggregate`。
- `Script/run_devcontainer_claude_code.ps1:107-115`：触发 `devcontainer up`，间接驱动该 Dockerfile 构建。
- `Script/run_devcontainer_claude_code.ps1:143`：容器内执行 `zsh -c 'claude; exec zsh'`，依赖本文件安装结果。

## 依赖与外部交互

1. 构建期外部网络交互
- Debian APT 源：安装系统包。
- GitHub Releases：下载 `git-delta` 与 `zsh-in-docker` 安装脚本。
- npm registry：下载 `@anthropic-ai/claude-code`。

2. 运行期跨文件依赖
- 由 `devcontainer.json` 提供运行能力（`NET_ADMIN/NET_RAW`）与 postStart 触发。
- 由 `init-firewall.sh` 消费镜像中预装命令。
- 由 devcontainer mounts 把 `/commandhistory` 与 `/home/node/.claude` 绑定到命名卷。

3. 权限与安全边界
- 默认会话用户是 `node`。
- 仅允许 `sudo /usr/local/bin/init-firewall.sh`，未开放其他 root 命令免密执行。

## 风险、边界与改进建议

1. 风险：版本漂移导致可复现性下降
- `FROM node:20` 与 `CLAUDE_CODE_VERSION=latest` 都是浮动版本，重建镜像行为可能变化。
- 建议：固定镜像 digest 与 Claude 版本号，升级时显式 PR 管理。

2. 风险：供应链下载缺少完整性校验
- `wget` 下载 `.deb` 与远程安装脚本后直接执行，未校验 checksum/signature。
- 建议：引入校验哈希或 GPG 签名验证，并在失败时终止构建。

3. 风险：远程脚本执行面较大
- `zsh-in-docker` 通过 `sh -c "$(wget -O- ...)"` 直接执行远程内容。
- 建议：改为下载固定版本文件后本地审计/校验再执行。

4. 风险：防火墙提权依赖 sudoers 文件完整性
- 当前做法是最小授权，但若 `/usr/local/bin/init-firewall.sh` 被意外覆盖，可能扩大发生面。
- 建议：将脚本路径改为只读层并附加完整性校验（如启动时校验 sha256）。

5. 边界：Dockerfile 只负责能力供给，不负责策略正确性
- 网络白名单策略是否合理由 `init-firewall.sh` 决定；本文件仅保证命令存在与权限可达。
- Windows/Powershell 启动稳定性由 `Script/run_devcontainer_claude_code.ps1` 负责，不在本文件边界内。
