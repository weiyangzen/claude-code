# FILE `.github/workflows/issue-opened-dispatch.yml` 研究文档

## 场景与职责
该工作流在新 issue 打开时向“目标仓库”发送 `repository_dispatch` 事件，实现跨仓通知/联动。它是本仓库对外事件桥接层。

## 功能点目的
- 将本仓库新 issue 事件同步到外部仓库或中台仓库。
- 把跨仓流程解耦为 dispatch 协议，而非在本仓库内硬编码后续处理。
- 通过 secrets 控制目标仓库与 token，实现环境化配置。

## 具体技术实现（关键流程/数据结构/协议/命令）
- 触发器：`issues: opened`。
- 权限：`issues: read`、`actions: write`。
- job 特点：
  - 不 checkout 代码，直接调用 `gh api`。
  - `timeout-minutes: 1`，快速失败。
- 执行命令：
  - `gh api repos/${TARGET_REPO}/dispatches`
  - `-f event_type=issue_opened`
  - `-f client_payload[issue_url]=<html_url>`
- 环境变量来源：
  - issue 元数据（URL/number/title）来自 event context
  - `TARGET_REPO`、`GH_TOKEN` 来自 secrets
- 失败处理：`|| { exit 0; }`，即 dispatch 失败时吞错并将工作流视为成功。

## 关键代码路径与文件引用
- 工作流入口：`.github/workflows/issue-opened-dispatch.yml`
- 外部消费方：不在本仓库（由 `TARGET_REPO` 指向）
- 上游事件源：GitHub issues webhook context
- 同域观测流：`.github/workflows/log-issue-events.yml`（同为 issue opened 事件）

## 依赖与外部交互
- 依赖：GitHub CLI `gh`（runner 内置）。
- 凭据：`ISSUE_OPENED_DISPATCH_TOKEN`。
- 外部交互：目标仓库 `POST /repos/{owner}/{repo}/dispatches`。
- 协议：`event_type=issue_opened` + `client_payload.issue_url`。

## 风险、边界与改进建议
- 风险 1：失败静默吞掉（`exit 0`），运维侧难以及时发现联动失效。
- 风险 2：只传 `issue_url`，下游需要二次拉取详情，耦合额外 API 调用。
- 风险 3：无重试机制，网络抖动会直接丢事件。
- 边界：该流只负责单向通知，不负责接收确认与补偿。
- 建议：
  - 至少记录 warning 日志并附失败原因。
  - 扩展 payload（issue_number/title/repo）减少下游查询。
  - 加入有限重试与告警通道，降低静默丢失概率。
