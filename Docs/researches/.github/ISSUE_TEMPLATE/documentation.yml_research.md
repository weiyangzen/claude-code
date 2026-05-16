# FILE `.github/ISSUE_TEMPLATE/documentation.yml` 研究文档

## 场景与职责

`documentation.yml` 用于收集 Claude Code 文档质量问题，覆盖缺失、过时、歧义、错链、示例不足等场景（`.github/ISSUE_TEMPLATE/documentation.yml:14-27`）。该模板将“文档问题”从 bug/feature 通道中分离，形成可直接进入文档维护流程的结构化输入。

关键职责：

- 统一文档类 issue 标识：标题前缀 `[DOCS] ` + 默认标签 `documentation`（`:3-5`）。
- 约束问题表达：要求明确位置、问题描述、改进建议、影响级别（`:30-105`）。
- 为 triage/优先级决策提供稳定字段，减少来回确认。

## 功能点目的

1. 明确问题类型与范围
- `doc_type` 强制选类（缺失/不清晰/过时/typo/示例/坏链/其他）。
- 目的：支持文档 backlog 的快速聚类。

2. 强调可定位与可修复
- `section`、`issue`、`suggested` 必填；`location` 与 `current` 补充上下文（`.github/ISSUE_TEMPLATE/documentation.yml:30-93`）。
- 目的：让维护者可以直接改文档，而不是先做需求澄清。

3. 建立影响评估输入
- `impact` 强制选择高/中/低（`:95-105`）。
- 目的：便于 triage 时按用户阻塞程度排序。

## 具体技术实现（关键流程/数据结构/协议/命令）

1. YAML Form 技术结构
- 顶层元信息：`name/description/title/labels`。
- `body` 内字段：
- 下拉：`doc_type`, `impact`
- 输入：`location`, `section`
- 文本域：`current`, `issue`, `suggested`, `additional`
- 必填校验：`doc_type/section/issue/suggested/impact`

2. 与下游自动化的关键流程
1) 用户通过模板创建文档 issue，自动带 `documentation` 标签。
2) `issues.opened` 触发 triage/dedupe/log/dispatch：
- triage workflow（`.github/workflows/claude-issue-triage.yml`）会基于 issue 内容和标签做二次判断。
- dedupe workflow（`.github/workflows/claude-dedupe-issues.yml`）可能给出重复 issue 评论。
3) 由于 triage 指令明确“生命周期标签主要用于 bug”，文档 issue 通常不应被 `needs-repro/needs-info` 处理（`.claude/commands/triage-issue.md:44-46,68`）。

3. 数据约束与协议细节
- `location` 非必填，适配“用户无法准确定位 URL”的场景。
- `suggested` 必填，推动“问题描述 -> 可执行改法”闭环。
- 整体仍是 markdown issue body 协议，不会以 JSON 结构传递给脚本；脚本通过 `gh issue view` 读取文本（`scripts/gh.sh:29-96`）。

4. 命令链路
- 标签修改由 `scripts/edit-issue-labels.sh` 执行时，会再次校验仓库真实存在的 label（`scripts/edit-issue-labels.sh:47-67`），避免模板标签与仓库标签漂移造成失败。

## 关键代码路径与文件引用

- 当前模板：`.github/ISSUE_TEMPLATE/documentation.yml`
- 入口总控：`.github/ISSUE_TEMPLATE/config.yml`
- triage 工作流：`.github/workflows/claude-issue-triage.yml`
- triage 指令：`.claude/commands/triage-issue.md`
- dedupe 工作流：`.github/workflows/claude-dedupe-issues.yml`
- issue 查询封装：`scripts/gh.sh`
- 标签编辑：`scripts/edit-issue-labels.sh`
- 生命周期定义：`scripts/issue-lifecycle.ts`
- 生命周期执行：`scripts/sweep.ts`, `.github/workflows/sweep.yml`
- 仓库反馈入口说明：`README.md:52-55`

## 依赖与外部交互

1. 平台依赖
- GitHub Issue Forms 对 dropdown/input/textarea 的渲染与必填校验。

2. 仓库运行依赖
- `anthropics/claude-code-action@v1`（triage/dedupe）。
- `gh`（issue 读取）、`bun`（lifecycle 脚本）、GitHub Actions runtime。

3. 外部交互
- 文档问题通常关联 `docs.claude.com` 内容；但当前模板 `location` 是自由输入，不做域名限制。
- Statsig 仅统计 issue 事件，不读取该模板具体字段。

4. 测试现状
- 未发现针对文档模板字段完整性或示例有效性的自动化测试。

## 风险、边界与改进建议

1. 风险：文档站点域名混用历史导致定位歧义
- 现状：仓库中存在 `code.claude.com` 与 `docs.claude.com` 并存引用。
- 影响：`location` 可能提交旧链接，修复定位耗时。
- 建议：在模板 placeholder 中给出统一文档主域，并在 triage 中提示旧域重定向检查。

2. 风险：`location` 非必填可能削弱可执行性
- 现状：用户可不填具体 URL。
- 影响：维护者需要额外追问。
- 建议：保留非必填但增加引导文案，鼓励最小可定位路径（URL/章节标题/截图三选一）。

3. 风险：文档 issue 被误作产品需求
- 现状：`doc_type=Missing documentation` 与 feature request 在边界上可能混淆。
- 影响：可能错误进入功能 backlog。
- 建议：在 triage 规则中加入“文档缺失但功能不存在时转 enhancement”的显式判断标准。

4. 边界：模板不保证建议内容正确
- `suggested` 能提高可操作性，但仍可能包含错误技术方案；最终需维护者技术审核。

5. 建议：新增“来源版本”字段
- 为过时文档问题增加“看到问题时的文档更新时间/页面版本”字段，可减少“文档已更新但 issue 未同步关闭”的噪音。
