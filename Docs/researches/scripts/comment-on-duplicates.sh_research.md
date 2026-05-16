# FILE `scripts/comment-on-duplicates.sh` 研究文档

## 场景与职责

`scripts/comment-on-duplicates.sh` 是 `/dedupe` 工作流的“落地写入器”。

前置步骤（模型搜索/筛选）完成后，最终由该脚本把候选重复 issue 以固定模板评论到目标 issue 上，为后续自动关闭流程提供标准输入。

## 功能点目的

1. 解析命令参数：`--base-issue` 与 `--potential-duplicates`。
2. 约束候选重复 issue 数量在 `1..3`。
3. 校验 base issue 与候选 issue 是否都真实存在。
4. 生成规范化评论文本（含候选链接、3 天自动关闭说明、反对方式）。
5. 调用 `gh issue comment` 落地评论。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 参数协议

- 命令格式：
  - `./scripts/comment-on-duplicates.sh --base-issue <N> --potential-duplicates <D1> <D2> [D3]`
- 解析策略：
  - `--base-issue` 读取单值。
  - `--potential-duplicates` 后读取连续非 `--` 参数作为数组。
- 非法参数、缺失参数、非数字参数会直接 `exit 1`。

### 2) 业务约束

- `BASE_ISSUE` 必填且必须为纯数字。
- `DUPLICATES` 至少 1 个、最多 3 个。
- 每个 duplicate 编号必须为纯数字。

### 3) 存在性校验

在真正评论前，脚本用 `gh issue view <num> --repo anthropics/claude-code` 逐个验证：

- base issue 必须存在。
- 候选 issue 必须全部存在。

任一不存在会报错并终止，防止发布无效链接评论。

### 4) 评论模板协议

评论正文结构：

1. 头部：`Found N possible duplicate issue(s):`
2. 编号列表：`https://github.com/anthropics/claude-code/issues/<dup>`
3. 自动化说明：3 天后自动关闭。
4. 用户引导：
   - 若确认重复可手动关闭并去原 issue 点赞。
   - 若不认同可留言或对该评论点 `👎`。
5. 机器人签名：`Generated with Claude Code`。

该模板被 `scripts/auto-close-duplicates.ts` 以关键词方式识别，因此属于跨脚本隐式协议。

## 关键代码路径与文件引用

- 主实现：`scripts/comment-on-duplicates.sh`
- 直接调用方：`.claude/commands/dedupe.md`
- workflow 入口：`.github/workflows/claude-dedupe-issues.yml`
- 下游消费方：`scripts/auto-close-duplicates.ts`

关键实现片段：

- 参数解析循环（`while [[ $# -gt 0 ]]` + `case`）。
- 数量与类型校验（数字正则 + 上限 3）。
- 预检 API 调用（`gh issue view`）。
- 文本构建 + `gh issue comment`。

## 依赖与外部交互

1. 运行依赖
- Bash（`set -euo pipefail`）。
- GitHub CLI `gh`（需已登录/注入 token）。

2. 外部交互
- `gh issue view`：校验 issue 是否存在。
- `gh issue comment`：写入重复提示评论。

3. 认证与权限
- 通常通过 GitHub Actions 中的 `GH_TOKEN` / `GITHUB_TOKEN` 提供权限。
- 需要 `issues:write` 才能发评论。

4. 与其他自动化的协同
- 上游依赖 `/dedupe` 搜索筛选结果。
- 下游被自动关闭脚本读取其评论文本与 reaction 状态。

## 风险、边界与改进建议

1. 仓库硬编码
- `REPO="anthropics/claude-code"` 无法复用到 fork 或其他仓库。
- 建议改为优先读取 `GH_REPO`/`GITHUB_REPOSITORY`。

2. 协议耦合强
- 自动关闭逻辑依赖该脚本的英文文案关键词。
- 建议在评论中加入不可见 marker，减少因文案改动导致的兼容问题。

3. 候选去重缺失
- 未检查 `DUPLICATES` 内重复编号、与 `BASE_ISSUE` 相同编号等无效输入。
- 建议增加去重和自引用拦截。

4. 状态语义未校验
- 仅校验“存在”，不校验候选是否 open、是否已锁定、是否为 PR。
- 建议根据策略增加状态过滤。

5. 幂等控制不足
- 重复执行会反复发同类评论。
- 建议在发评论前扫描近似模板评论并跳过。

6. 可观测性一般
- 成功日志只输出一行，缺少 comment URL 或请求 ID。
- 建议输出结构化日志方便后续审计。
