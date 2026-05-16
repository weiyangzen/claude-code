# FILE `.devcontainer/devcontainer.json` 研究文档

## 场景与职责

`.devcontainer/devcontainer.json` 是该仓库 DevContainer 的运行编排入口，职责是把“如何构建镜像、如何启动容器、启动后做什么、IDE 如何附着”统一声明成一份结构化配置。它是 `.devcontainer/Dockerfile` 与 `.devcontainer/init-firewall.sh` 的直接调度方：

- 构建期：指定 `Dockerfile` 与构建参数。
- 运行期：声明容器能力（`NET_ADMIN/NET_RAW`）、挂载、环境变量、远程用户。
- 启动后：执行 `sudo /usr/local/bin/init-firewall.sh`，并要求容器等待该步骤完成后再进入可用态。

结合 `Script/run_devcontainer_claude_code.ps1`，该文件是 Windows 入口脚本 `devcontainer up` 之后真正生效的容器行为定义。

## 功能点目的

1. 统一构建参数与版本注入
- `build.dockerfile` 指向本目录 `Dockerfile`。
- `build.args` 传入 `TZ`、`CLAUDE_CODE_VERSION`、`GIT_DELTA_VERSION`、`ZSH_IN_DOCKER_VERSION`，保证镜像构建参数显式可控。

2. 赋予网络控制所需内核能力
- `runArgs` 增加 `--cap-add=NET_ADMIN`、`--cap-add=NET_RAW`，确保容器内可执行 iptables/ipset 相关操作。

3. 固化 IDE 侧开发体验
- 预置 VS Code 扩展（Claude Code、ESLint、Prettier、GitLens）。
- 配置保存格式化、显式 ESLint fix、默认 zsh 终端 profile。

4. 保持状态持久化与资源可用
- 用命名卷持久化 `/commandhistory` 与 `/home/node/.claude`。
- 通过 `containerEnv.NODE_OPTIONS` 调高 Node 内存上限到 4096MB。
- 设置 `CLAUDE_CONFIG_DIR=/home/node/.claude`，与 Changelog 中 “Respect CLAUDE_CONFIG_DIR everywhere” 的行为一致。

5. 启动时强制网络收敛
- `postStartCommand` 调用防火墙脚本。
- `waitFor: postStartCommand` 要求在 postStart 完成前不放行开发会话，防止“先联网再收敛”的窗口期。

## 具体技术实现（关键流程/数据结构/协议/命令）

1. 配置结构与解析入口
- 顶层是标准 devcontainer schema 的 JSON 对象，核心字段顺序为：`build` -> `runArgs` -> `customizations` -> `remoteUser/mounts/containerEnv` -> `workspaceMount/workspaceFolder` -> `postStartCommand/waitFor`。
- `devcontainer` CLI 在 `devcontainer up --workspace-folder .` 时读取并执行这些字段（调用入口见 `Script/run_devcontainer_claude_code.ps1:107-115`）。

2. 构建阶段实现
- `build.dockerfile = "Dockerfile"`，由 CLI 触发容器构建。
- `build.args`:
  - `TZ` 使用 `${localEnv:TZ:America/Los_Angeles}`，语义是“优先宿主 TZ，缺失时默认洛杉矶时区”。
  - 其余版本参数与 Dockerfile 内 `ARG` 一一对应。

3. 运行时权限与用户模型
- `runArgs` 注入 Linux capability 而非 `--privileged`，属于较收敛的授权策略。
- `remoteUser = node`：容器终端默认非 root。

4. IDE 与终端体验层
- `customizations.vscode.extensions` 提供容器内开箱即用插件集合。
- `terminal.integrated.defaultProfile.linux = zsh` 与 Dockerfile `ENV SHELL=/bin/zsh` 形成一致体验。

5. 存储与挂载策略
- `mounts` 使用 `${devcontainerId}` 隔离历史与配置卷，避免不同容器实例互相污染。
- `workspaceMount` 采用 bind mount + `consistency=delegated`（偏向 macOS/Docker Desktop 场景的性能优化选项），`workspaceFolder=/workspace` 与 Dockerfile `WORKDIR` 一致。

6. 启动后命令链路
- `postStartCommand`：`sudo /usr/local/bin/init-firewall.sh`。
- 此命令依赖 Dockerfile 在 `/etc/sudoers.d/node-firewall` 的最小授权设置。
- `waitFor: postStartCommand` 把初始化脚本执行结果变成容器可用性的硬门槛。

## 关键代码路径与文件引用

1. 当前文件关键段
- `.devcontainer/devcontainer.json:3-10`：构建文件与 build args。
- `.devcontainer/devcontainer.json:12-15`：网络能力声明（`NET_ADMIN/NET_RAW`）。
- `.devcontainer/devcontainer.json:16-42`：VS Code 插件与编辑器/终端设置。
- `.devcontainer/devcontainer.json:44-47`：命名卷挂载（历史与 Claude 配置）。
- `.devcontainer/devcontainer.json:48-52`：容器环境变量。
- `.devcontainer/devcontainer.json:53-56`：工作目录绑定与防火墙 postStart 链路。

2. 上下游关联文件
- `.devcontainer/Dockerfile:1-91`：构建被调用方，消费 `build.args` 并准备脚本权限。
- `.devcontainer/init-firewall.sh:1-137`：被 `postStartCommand` 调用，执行网络收敛。
- `Script/run_devcontainer_claude_code.ps1:107-149`：触发 `devcontainer up` 并进入容器执行 `claude`。
- `.vscode/extensions.json:1-8`：推荐安装 `ms-vscode-remote.remote-containers`，与 devcontainer 使用场景对齐。
- `CHANGELOG.md:2181`：`CLAUDE_CONFIG_DIR` 行为升级说明，与本文件 `containerEnv` 配置直接对应。

## 依赖与外部交互

1. 本地依赖
- 主机必须安装 `devcontainer` CLI 与 Docker/Podman。
- 宿主工作区路径通过 `${localWorkspaceFolder}` 绑定进入 `/workspace`。

2. 跨文件依赖
- 依赖 Dockerfile 提供命令：`sudo`、`iptables`、`ipset`、`curl`、`jq`、`dig`、`aggregate`。
- 依赖 init-firewall 脚本成功返回，才能满足 `waitFor`。

3. 外部服务交互（间接）
- 容器构建与 postStart 阶段会访问外网（APT、GitHub、npm、Anthropic、VS Code 相关域名），由下游脚本决定更细粒度交互。

## 风险、边界与改进建议

1. 风险：`postStartCommand` 失败会导致容器不可用
- 当前是 fail-fast 设计，安全性高但可用性敏感。
- 建议：引入“严格模式/开发模式”开关，例如通过环境变量决定失败时是否阻塞会话。

2. 风险：能力授权仍具攻击面
- `NET_ADMIN/NET_RAW` 是为防火墙脚本必需，但能力本身较强。
- 建议：若未来迁移到 eBPF/sidecar 网络代理，可考虑移除这两项 capability。

3. 风险：默认时区可能与团队地区不一致
- 缺失 `TZ` 时会回落到 `America/Los_Angeles`，跨区域协作日志可能产生时间偏差。
- 建议：在团队文档中明确要求设置本地 `TZ`，或改为 `UTC` 默认值。

4. 风险：`consistency=delegated` 的平台差异
- 该选项在非 Docker Desktop 环境（尤其 Linux 原生/Podman）语义可能弱化或被忽略。
- 建议：补充跨后端说明，必要时按后端动态生成 mount 参数。

5. 风险：扩展清单偏 VS Code 专用
- 当前配置对非 VS Code 客户端基本无影响，但也不提供等价体验配置。
- 建议：在仓库文档增加“VS Code 之外如何附着容器”的指引。

6. 边界：该文件只描述容器编排，不负责脚本策略正确性
- 防火墙白名单规则是否合理由 `init-firewall.sh` 决定。
- 镜像供应链安全由 Dockerfile 及其下载来源策略负责。
