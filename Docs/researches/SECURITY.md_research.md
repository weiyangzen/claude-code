# FILE `SECURITY.md` 研究文档

## 场景与职责

`SECURITY.md` 是仓库级漏洞披露入口说明，核心职责不是提供代码级防护实现，而是定义“发现漏洞后去哪里报、按什么渠道报”。

1. 统一漏洞报告渠道
- 明确要求通过 HackerOne 表单提交已验证漏洞（`SECURITY.md:8`）。

2. 安全项目声明
- 给出安全优先级与对安全研究者的合作态度（`SECURITY.md:2,6`）。

3. 程序规则跳转
- 指向 HackerOne Program 页面查看披露规范（`SECURITY.md:12`）。

## 功能点目的

1. 避免漏洞在公开 issue 泄露
- 使用私有漏洞提交流程（HackerOne）替代公开 GitHub issue 讨论。

2. 提高漏洞受理效率
- 用固定入口减少“发错渠道”的处理成本。

3. 形成与运行时安全机制的治理分层
- `SECURITY.md` 负责“组织流程层”漏洞上报。
- 插件 `security-guidance` 负责“开发时运行层”安全提醒（`plugins/security-guidance/hooks/hooks.json:2-13`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 文档结构与协议

- 采用简洁 Markdown 结构：标题 + 两个章节（Reporting Security Issues / Vulnerability Disclosure Program）。
- 不包含 YAML frontmatter、表格、机器可读字段。
- 关键协议通过 URL 参数体现：
  - 报告链接为 `.../reports/new?type=team&report_type=vulnerability`（`SECURITY.md:8`），已固定到 team + vulnerability 场景。

### 2) 披露流程（文本协议驱动）

1. 研究者确认漏洞有效性。
2. 通过 `SECURITY.md` 的 HackerOne 提交链接进入漏洞提报表单。
3. 依据 HackerOne Program 页面规则完成披露与后续沟通。

### 3) 与仓内其他流程的上下文关系

1. 与一般 bug 报告流程区分
- README 的反馈段默认引导 `/bug` 或 GitHub issue（`README.md:52-55`），并未显式提示安全漏洞应走 `SECURITY.md`。
- `.github/ISSUE_TEMPLATE/config.yml` 仅配置 Discord/文档/入门/排障链接，未配置 security contact link（`config.yml:2-14`）。

2. 与安全运行时 Hook 的关系
- `plugins/security-guidance` 在 PreToolUse 对 Edit/Write/MultiEdit 做安全提醒（`hooks.json:4-13`），属于预防型机制。
- `SECURITY.md` 是事后披露和协调机制，二者互补。

3. 与研究自动化流程关系
- 研究 guard 会把 `SECURITY.md` 作为 FILE 条目驱动研究文档生成、checklist 勾选和 todo 刷新（`.ops/research_guard.sh:320-337`）。

## 关键代码路径与文件引用

1. 主文件
- `SECURITY.md:1-12`

2. 相关入口文档
- `README.md:52-55`（通用 bug 报告）
- `.github/ISSUE_TEMPLATE/config.yml:2-14`（issue 联系入口配置）

3. 运行时安全相关实现（对照）
- `plugins/security-guidance/hooks/hooks.json:2-13`
- `plugins/security-guidance/hooks/security_reminder_hook.py:31-126`（9 类安全模式）

4. 研究流程链路
- `.ops/research_guard.sh:320-345`
- `.ops/generate_daily_research_todo.sh:15-39`
- `Docs/researches/blueprint_checklist.md:146`
- `Docs/researches/todos_20260320.md:16`

## 依赖与外部交互

1. 外部平台依赖
- `hackerone.com/anthropic-vdp/reports/new`（漏洞提报）。
- `hackerone.com/anthropic-vdp`（项目规则页）。

2. 本地仓库依赖
- 未发现任何脚本自动读取 `SECURITY.md` 内容后执行动作。
- 主要依赖 GitHub 平台约定：仓库根 `SECURITY.md` 作为人工可发现的安全政策文件。

3. 配置、测试、脚本交互
- 无针对 `SECURITY.md` 的自动化测试或链接检查。
- 安全相关“可执行部分”在插件 hook 中，不在该文件中。

## 风险、边界与改进建议

1. 风险：缺少支持版本范围与响应 SLA
- 文档未声明“哪些版本受支持”“预计响应时限/修复时限”。
- 建议：增加 Supported Versions 与 Response Targets 章节。

2. 风险：安全上报路径与普通 bug 路径分流不够显式
- README 只强调 `/bug` 与 GitHub issue，可能导致安全问题误报到公开渠道。
- 建议：在 `README.md` 的 Reporting Bugs 段增加安全漏洞专门提示并链接 `SECURITY.md`。

3. 风险：缺少加密通信/替代联系方式说明
- 当前只有 HackerOne 链接，没有 PGP、邮箱或平台故障 fallback。
- 建议：补充备用联系路径（至少一个 mailbox + 使用条件）。

4. 风险：披露规则细节完全外置
- 若外部页面变更或不可达，仓内缺乏最小可用说明。
- 建议：在 `SECURITY.md` 增加最小本地摘要（允许行为、禁止公开披露前置条件、奖励计划入口）。

5. 边界说明
- `SECURITY.md` 不负责代码防护策略实现，也不替代插件安全检查。
- 它是治理流程文档；有效性取决于入口可达、规则清晰、与其他反馈入口的一致分流。
