# FILE `.devcontainer/init-firewall.sh` 研究文档

## 场景与职责

`.devcontainer/init-firewall.sh` 是 DevContainer 启动后的安全收敛脚本，职责是在容器进入开发会话前建立“默认拒绝、白名单放行”的出站网络策略。该脚本由 `.devcontainer/devcontainer.json` 的 `postStartCommand` 调用，并被 `waitFor: postStartCommand` 设为硬前置，因此它的成功与否直接决定容器可用性。

它的设计目标不是通用防火墙管理，而是为 Claude Code 开发环境提供最小网络面：

- 保持 DNS、loopback、宿主网段和已建立连接可用。
- 放行 GitHub 相关网段与少量业务依赖域名。
- 默认拒绝其他所有出站连接，并在启动时做连通性正反向自检。

## 功能点目的

1. 启动时清理旧规则并恢复 Docker DNS 基础能力
- 清空 `filter/nat/mangle` 规则，销毁旧 `allowed-domains` ipset。
- 在清理前提取 Docker 内部 DNS 规则（匹配 `127.0.0.11`），清理后仅恢复该部分，避免容器 DNS 被彻底破坏。

2. 构建允许访问目标集合
- 创建 `ipset allowed-domains hash:net`。
- 通过 `https://api.github.com/meta` 动态获取 GitHub `web/api/git` 网段并聚合后写入。
- 对固定域名列表执行 DNS A 记录解析，把 IP 写入 ipset。

3. 建立默认拒绝策略
- 先放行必要基础流量（DNS、SSH、loopback、主机网段）。
- 设置默认策略 `INPUT/FORWARD/OUTPUT = DROP`。
- 放行已建立连接、放行到 `allowed-domains` 的出站连接，其余显式 `REJECT`。

4. 启动阶段自验证
- 验证 `https://example.com` 不可达（证明默认拒绝生效）。
- 验证 `https://api.github.com/zen` 可达（证明白名单链路可用）。

## 具体技术实现（关键流程/数据结构/协议/命令）

1. 严格执行选项
- `set -euo pipefail` 与 `IFS=$'\n\t'`：
  - 任意关键命令失败立即退出。
  - 未定义变量与 pipeline 失败都会中断。
  - 降低字符串分词/空格导致的脚本歧义。

2. 规则重置与 Docker DNS 恢复
- `iptables-save -t nat | grep "127\.0\.0\.11"` 提取 Docker DNS 相关 NAT 规则。
- 依次 flush `filter/nat/mangle` 表，并删除旧链。
- 若存在提取结果，创建 `DOCKER_OUTPUT` / `DOCKER_POSTROUTING` 链并通过 `xargs -L 1 iptables -t nat` 逐条恢复。

3. 基础放行规则
- DNS：`OUTPUT udp dport 53 ACCEPT` + `INPUT udp sport 53 ACCEPT`。
- SSH：`OUTPUT tcp dport 22 ACCEPT` + `INPUT tcp sport 22 ESTABLISHED ACCEPT`。
- 本地回环：`INPUT/OUTPUT lo ACCEPT`。

4. 白名单数据源构建
- 数据结构：`ipset allowed-domains hash:net`（同时支持 IP 与 CIDR）。
- GitHub 网段流程：
  1) `curl -s https://api.github.com/meta` 获取 JSON。
  2) `jq -e '.web and .api and .git'` 校验关键字段存在。
  3) `jq -r '(.web + .api + .git)[]' | aggregate -q` 合并网段。
  4) 正则校验 CIDR 形态后 `ipset add allowed-domains <cidr>`。
- 固定域名流程：
  1) `dig +noall +answer A <domain> | awk '$4 == "A" {print $5}'` 解析 IPv4。
  2) 正则校验 IP 形态。
  3) `ipset add allowed-domains <ip>`。

5. 宿主网络推导与规则收口
- 通过 `ip route | grep default | cut -d" " -f3` 获取默认网关 IP。
- 用 `sed 's/\.[0-9]*$/.0\/24/'` 推导宿主网段（固定 /24 假设）。
- 放行该网段 `INPUT/OUTPUT`。
- 设置默认策略 DROP 后，再放行：
  - `ESTABLISHED,RELATED`
  - `-m set --match-set allowed-domains dst`
- 最后一条 `OUTPUT REJECT --reject-with icmp-admin-prohibited` 使被拒连接立即失败，避免长时间超时。

6. 验证逻辑
- `curl --connect-timeout 5 https://example.com` 成功即报错退出。
- `curl --connect-timeout 5 https://api.github.com/zen` 失败即报错退出。

## 关键代码路径与文件引用

1. 当前文件关键段
- `.devcontainer/init-firewall.sh:1-3`：严格执行模式。
- `.devcontainer/init-firewall.sh:5-25`：规则清理与 Docker DNS 恢复。
- `.devcontainer/init-firewall.sh:27-41`：基础放行与 ipset 初始化。
- `.devcontainer/init-firewall.sh:43-64`：GitHub meta 拉取、校验、聚合写入。
- `.devcontainer/init-firewall.sh:66-91`：固定域名解析并写入白名单。
- `.devcontainer/init-firewall.sh:93-120`：主机网段推导、默认策略和最终拒绝规则。
- `.devcontainer/init-firewall.sh:122-137`：启动自检。

2. 调用链与依赖文件
- `.devcontainer/devcontainer.json:55-56`：`postStartCommand` 调用并等待本脚本执行结果。
- `.devcontainer/Dockerfile:9-28`：提供脚本依赖命令（iptables/ipset/dig/jq/aggregate/curl）。
- `.devcontainer/Dockerfile:85-91`：把本脚本安装到 `/usr/local/bin` 并授予 `node` 免密 sudo 调用。
- `Script/run_devcontainer_claude_code.ps1:107-115`：`devcontainer up` 触发 postStart，间接触发本脚本。

## 依赖与外部交互

1. 系统命令依赖
- `iptables`, `iptables-save`, `ipset`, `ip`, `sed`, `grep`, `cut`, `xargs`, `curl`, `jq`, `aggregate`, `dig`, `awk`。

2. 外部网络依赖
- GitHub 元数据接口：`https://api.github.com/meta`。
- DNS 查询目标域名：
  - `registry.npmjs.org`
  - `api.anthropic.com`
  - `sentry.io`
  - `statsig.anthropic.com`
  - `statsig.com`
  - `marketplace.visualstudio.com`
  - `vscode.blob.core.windows.net`
  - `update.code.visualstudio.com`

3. 运行能力依赖
- 容器需要 `NET_ADMIN/NET_RAW` capability（由 `devcontainer.json` 注入）。
- 调用用户需要 sudo 执行权限（由 Dockerfile 的 sudoers 文件授权）。

## 风险、边界与改进建议

1. 风险：全量 flush 规则可能影响非预期链路
- 当前会清空多个表的全部规则，再尝试恢复部分 Docker DNS 规则。
- 改进：只管理自定义链（如 `CLAUDE_*`），避免覆盖已有规则。

2. 风险：`ipset add` 重复项可能触发脚本中断
- 在 `set -e` 下，若不同域名解析出同一 IP，重复 `ipset add` 可能返回错误并退出。
- 改进：使用 `ipset add -exist` 或写入前先查询是否存在。

3. 风险：仅支持 IPv4 A 记录
- 未解析 AAAA，且规则未覆盖 ip6tables，IPv6 场景会行为不一致。
- 改进：补充 IPv6 路径（`dig AAAA` + `ipset family inet6` + `ip6tables`）。

4. 风险：DNS 仅放行 UDP 53
- 某些响应可能触发 TCP fallback（大响应、DNSSEC 场景），当前会被阻断。
- 改进：补充 TCP 53 的 INPUT/OUTPUT 放行规则。

5. 风险：宿主网络固定推导 `/24`
- 对非 `/24` 网段或复杂路由环境不准确，可能误放行或误阻断。
- 改进：从路由表读取真实前缀长度，或允许通过环境变量显式传入。

6. 风险：启动成功依赖外部 API 与 DNS 实时可用
- `api.github.com/meta` 或 DNS 临时失败会直接让 postStart 失败，容器不可用。
- 改进：引入缓存快照与重试机制（指数退避），提供受控降级模式。

7. 风险：`xargs` 恢复 NAT 规则的稳健性一般
- 若规则文本格式变化或包含边界字符，恢复可能失败。
- 改进：使用更稳健的 `iptables-restore` 片段恢复方式。

8. 边界：脚本目标是“开发容器网络收敛”，不是完整零信任网关
- 它不提供细粒度用户/进程级审计，也不处理跨容器服务网格策略。
- 其设计重点是“最小可用 + 明确失败”，而非最大兼容性。
