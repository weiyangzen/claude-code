# FILE `scripts/issue-lifecycle.ts` 研究文档

## 场景与职责

`scripts/issue-lifecycle.ts` 是 issue 生命周期自动化的配置中枢，集中定义“哪些标签属于生命周期标签、超时多少天、提醒文案与关闭理由”。

它不直接调用 GitHub API，而是被执行脚本读取：

- `scripts/sweep.ts` 用于定时打标/关闭。
- `scripts/lifecycle-comment.ts` 用于加标签时自动留言。

## 功能点目的

1. 统一维护生命周期标签策略，避免多个脚本各自硬编码。
2. 提供 `label / days / reason / nudge` 的完整配置结构。
3. 用 `as const` 保持字面量类型，提升 TypeScript 静态安全。
4. 暴露 `LifecycleLabel` 类型供调用方约束。
5. 提供 `STALE_UPVOTE_THRESHOLD`，保护高关注 issue。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 生命周期策略数组

`lifecycle` 为只读常量数组，每项字段：

- `label`: 生命周期标签名。
- `days`: 从“被打标时间”起的超时天数。
- `reason`: 关闭时写入的理由片段。
- `nudge`: 加标签后提示评论文案。

当前条目：

- `invalid`：3 天。
- `needs-repro`：7 天。
- `needs-info`：7 天。
- `stale`：14 天。
- `autoclose`：14 天。

### 2) 类型导出

- `LifecycleLabel = (typeof lifecycle)[number]["label"]`
- 该类型把 label 限制在上面 5 个字面量中，防止调用方写错字符串。

### 3) 热度阈值导出

- `STALE_UPVOTE_THRESHOLD = 10`
- `sweep.ts` 会用它跳过高赞 issue，避免“活跃但无人更新”问题被误关。

### 4) 配置-执行分离

该文件实现“策略数据与执行逻辑分离”：

- 修改生命周期规则无需改 `sweep.ts`/`lifecycle-comment.ts` 核心代码。
- 一处变更可同时影响“评论预警”和“超时关闭”。

## 关键代码路径与文件引用

- 配置源文件：`scripts/issue-lifecycle.ts`
- 消费方（定时治理）：`scripts/sweep.ts`
- 消费方（标签事件留言）：`scripts/lifecycle-comment.ts`
- 对应 workflow：
  - `.github/workflows/sweep.yml`
  - `.github/workflows/issue-lifecycle-comment.yml`
- 标签策略输入来源（模型层）：`.claude/commands/triage-issue.md`

## 依赖与外部交互

1. 运行依赖
- 纯 TypeScript 常量模块，无外部命令依赖。

2. 外部交互
- 本文件自身不与 GitHub API 交互。
- 外部交互由消费方脚本承担。

3. 协议依赖
- 与仓库真实 labels 集合必须保持一致（否则 sweep/comment 会按配置找不到目标标签）。
- 与 triage 策略文档需要同步，否则可能出现“可打标但无生命周期策略”或反向不一致。

## 风险、边界与改进建议

1. 配置漂移风险
- `triage-issue.md` 中 lifecycle 说明与本文件需要人工同步，存在偏差风险。
- 建议从本文件生成文档片段，减少手工维护。

2. 标签存在性未验证
- 配置中标签是否真的存在于仓库未在 CI 校验。
- 建议增加校验脚本：启动时比对 `gh label list`。

3. i18n 与文案治理边界
- `nudge` 文案硬编码在代码，难以按渠道/语言变体管理。
- 建议抽离到配置文件或模板层。

4. 生命周期粒度固定
- 当前只有全仓统一天数，无法按标签组合或 issue 类型动态调整。
- 建议支持可选的条件策略（例如 bug 与 enhancement 不同超时）。

5. 阈值配置缺少上下文
- `STALE_UPVOTE_THRESHOLD=10` 缺少历史依据与调参说明。
- 建议在文档里记录阈值来源和调整策略。
