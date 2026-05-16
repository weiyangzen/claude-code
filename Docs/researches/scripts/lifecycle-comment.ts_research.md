# FILE `scripts/lifecycle-comment.ts` 研究文档

## 场景与职责

`scripts/lifecycle-comment.ts` 用于在 issue 被加上生命周期标签时自动留言，向报告者说明“为什么被标记”以及“多久后会自动关闭”。

它是生命周期治理的人机沟通层，避免 `sweep.ts` 直接关单时缺少提前告知。

## 功能点目的

1. 监听 `issues.labeled` 事件并读取标签/issue 上下文。
2. 仅对 `issue-lifecycle.ts` 中定义的生命周期标签生效。
3. 生成统一提示评论（`nudge + X days timeout`）。
4. 支持 `--dry-run` 验证文案，不落地写入。
5. 对 API 失败进行显式报错，保障 workflow 可见性。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 输入来源与前置校验

脚本读取环境变量：

- `GITHUB_TOKEN`（非 dry-run 必需）
- `GITHUB_REPOSITORY`（`owner/repo`）
- `LABEL`
- `ISSUE_NUMBER`

任一关键输入缺失会直接抛错终止。

### 2) 生命周期标签匹配

- 从 `issue-lifecycle.ts` 导入 `lifecycle`。
- 使用 `lifecycle.find((l) => l.label === label)` 查配置项。
- 找不到配置项时打印 `No lifecycle entry ...` 并 `process.exit(0)`，避免对非生命周期标签误留言。

### 3) 评论体构建

`body` 格式：

- `entry.nudge`
- 加一句固定后缀：`This issue will be closed automatically ... within <entry.days> days.`

这样“标签策略”和“提示时间”严格由同一配置源驱动。

### 4) API 调用

- `POST https://api.github.com/repos/${repo}/issues/${issueNumber}/comments`
- Header:
  - `Authorization: Bearer <token>`
  - `Accept: application/vnd.github.v3+json`
  - `Content-Type: application/json`
  - `User-Agent: lifecycle-comment`
- 非 2xx 时读取响应文本并抛错，方便 workflow 定位失败原因。

### 5) 触发入口

`.github/workflows/issue-lifecycle-comment.yml` 在 `issues: labeled` 事件触发，执行 `bun run scripts/lifecycle-comment.ts` 并注入：

- `LABEL: ${{ github.event.label.name }}`
- `ISSUE_NUMBER: ${{ github.event.issue.number }}`

`GITHUB_REPOSITORY` 通常由 GitHub Actions 默认环境变量提供。

## 关键代码路径与文件引用

- 主实现：`scripts/lifecycle-comment.ts`
- 生命周期配置：`scripts/issue-lifecycle.ts`
- 调用 workflow：`.github/workflows/issue-lifecycle-comment.yml`
- 打标与关闭执行器：`scripts/sweep.ts`
- 标签移除协同：`.github/workflows/remove-autoclose-label.yml`

关键流程节点：

- 环境变量校验。
- 生命周期 label 匹配。
- dry-run 分支。
- 评论 POST 与失败抛错。

## 依赖与外部交互

1. 运行依赖
- Bun 运行时。

2. 外部 API
- `POST /repos/{owner}/{repo}/issues/{issue_number}/comments`

3. 权限需求
- workflow 需要 `issues: write`。

4. 协作关系
- 与 `issue-lifecycle.ts` 强耦合（文案与天数）。
- 与 `sweep.ts` 时间策略协同，负责“先通知，再可能关闭”。

## 风险、边界与改进建议

1. 幂等性不足
- 同一 lifecycle 标签被重复添加时会重复评论。
- 建议在评论中加入 marker 并检测近似重复评论后跳过。

2. 仓库环境变量隐式依赖
- workflow 未显式传 `GITHUB_REPOSITORY`，依赖平台默认注入。
- 建议在 workflow 显式传入以降低环境差异风险。

3. 失败重试缺失
- 网络抖动或临时限流会直接失败。
- 建议加入轻量重试（指数退避）。

4. 文案统一性边界
- 文案由 `nudge + 固定后缀` 拼接，无法按标签定制更细粒度说明（例如不同复议方式）。
- 建议支持模板占位符或 label-specific footer。

5. 仅基于标签事件
- 对于历史已存在标签的 issue，不会补发提示。
- 建议保留一次性 backfill 能力（按 label 扫描缺失提示评论）。
