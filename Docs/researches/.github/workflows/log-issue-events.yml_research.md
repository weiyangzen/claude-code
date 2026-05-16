# FILE `.github/workflows/log-issue-events.yml` 研究文档

## 场景与职责
该工作流负责把 issue 事件写入 Statsig，用于运营观测与数据分析。它监听 issue 打开/关闭事件，是 issue 域埋点出口。

## 功能点目的
- 记录 issue 发生情况，支持外部看板统计。
- 把 issue 元数据（编号、标题、作者、时间）发送到 Statsig。
- 保持逻辑极简，降低业务流耦合。

## 具体技术实现（关键流程/数据结构/协议/命令）
- 触发器：`issues: [opened, closed]`。
- 权限：`issues: read`。
- 实现：单步 shell `curl` 调 Statsig。
- 输入字段来自环境变量：
  - `ISSUE_NUMBER`, `REPO`, `ISSUE_TITLE`, `AUTHOR`, `CREATED_AT`
- 请求协议：
  - `POST https://events.statsigapi.net/v1/log_event`
  - Header：`Content-Type: application/json`、`statsig-api-key`
  - Body：`events[0].eventName = github_issue_created`
- 安全细节：标题经 `sed` 转义双引号，避免 JSON 破坏。

## 关键代码路径与文件引用
- 工作流入口：`.github/workflows/log-issue-events.yml`
- 同类观测流：`.github/workflows/claude-dedupe-issues.yml`（也写 Statsig）
- 事件来源：GitHub `issues` webhook context

## 依赖与外部交互
- 依赖：`curl`、`sed`、`date`（runner 内置）。
- 凭据：`STATSIG_API_KEY`。
- 外部交互：Statsig Events API。
- 数据边界：当前 payload 未携带 issue body，仅标题与基础元数据。

## 风险、边界与改进建议
- 风险 1：监听了 `closed`，但事件名仍固定 `github_issue_created`，语义不一致。
- 风险 2：时间字段使用 `CREATED_AT`，closed 事件未反映 `closed_at`。
- 风险 3：`curl` 未显式校验 HTTP 状态，失败可能不易发现。
- 边界：只有单事件上报，不做重试与去重。
- 建议：
  - 根据 `github.event.action` 区分 `github_issue_opened`/`github_issue_closed`。
  - 为 closed 事件追加 `closed_at`。
  - 增加 `--fail`、状态码检查与失败日志。
