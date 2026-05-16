# FILE `demo.gif` 研究文档

## 场景与职责

`demo.gif` 是仓库根目录的演示资产文件，主要承担 README 首屏“能力演示”职责，而不是代码运行时依赖。

1. 文档展示入口
- 在根文档中通过 `<img src="./demo.gif" />` 直接内嵌（`README.md:11`），紧接在产品说明段之后（`README.md:7-11`）。

2. 产品心智建立
- 通过动图直接展示 Claude Code 在终端中的交互流程（从输入任务到执行 Read/Search/Bash 等步骤），降低文字说明理解门槛。

3. 研究流程对象
- 该文件被研究流程脚本自动发现并纳入 checklist/todo（`.ops/generate_research_blueprint_checklist.sh:36-42`，`.ops/generate_daily_research_todo.sh:15-39`）。

## 功能点目的

1. 作为 README 的“视觉主叙事”
- 在“Get started”之前先给出真实操作感，帮助新用户快速判断工具形态和交互模式（`README.md:11-13`）。

2. 强化“代理式编码”定位
- 动图内容呈现了任务分解、工具调用与中间状态（如 `Identifying testing framework...`、`Running tests with coverage...`），与 README 首段定位一致（`README.md:7`）。

3. 降低外链依赖
- 资源随仓库版本管理，README 渲染不依赖第三方媒体托管链接，减少链接失效风险。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 文件与编码参数

`demo.gif` 为 GIF89a 动图，关键参数（实测）：
- 路径：`demo.gif`
- 大小：`11,002,760` bytes（约 11 MB）
- 分辨率：`1552x992`
- 时长：`42.4s`
- 帧数：`414`
- 平均帧率：`10fps`
- 帧间隔分布：`0.1s * 408`、`0.2s * 4`、`0.4s * 2`
- SHA-256：`461cf109f52648929acd9959a87e5673541f8249497512b8e4f1d32d10f0fd57`

### 2) README 嵌入协议

该文件通过 Markdown 中嵌入 HTML `<img>` 标签接入：

1. README 渲染阶段解析 `<img src="./demo.gif" />`（`README.md:11`）。
2. 相对路径 `./demo.gif` 指向仓库根目录同名文件。
3. 页面加载时由前端进行 GIF 解码并按帧播放。

说明：第 3 步是基于 Web 页面加载行为的工程推断，仓库内无自定义渲染器代码。

### 3) 内容流程（基于抽帧观察）

抽取关键帧可看到演示主线：
1. 输入任务：`audit and improve test coverage`。
2. 代理给出计划并进入执行状态。
3. 发生 `Read/Search/Bash` 等工具调用，读取 `package.json` 与测试文件并搜索测试模式。
4. 执行覆盖率相关命令，过程中出现一次依赖检查报错后继续推进。

这说明动图目标不是“完美成功案例”，而是展示真实代理工作流（含中间状态与异常处理）。

### 4) 历史演进

`git log --follow -- demo.gif` 显示该资产经历两次关键变更：
- `715ea8e`：首次引入（随 README 更新）。
- `1abf1a5`：替换为更新录屏版本。

历史对象中可见旧版体积约 `16,351,242` bytes，当前版约 `11,002,760` bytes，说明该文件存在“整文件替换”型演进特征。

## 关键代码路径与文件引用

1. 直接调用方
- `README.md:11`：`<img src="./demo.gif" />`

2. 被调用目标
- `demo.gif`（仓库根目录二进制资产）

3. 与研究流程相关的脚本路径
- `.ops/generate_research_blueprint_checklist.sh:36-42`：通过 `find` 收集文件清单（包含 `demo.gif`）。
- `.ops/generate_daily_research_todo.sh:15-39`：从 checklist 生成当日 pending todo。
- `.ops/research_guard.sh:286-310`：FILE 任务模板（研究文档命名、勾选 checklist、提交要求）。

4. 与当前任务状态相关
- `Docs/researches/blueprint_checklist.md:148`：`demo.gif` 对应 checklist 项。
- `Docs/researches/todos_20260320.md`：每日 todo 快照（由脚本重新生成）。

5. 相关配置
- `.gitattributes:1-2`：仅通用文本规则，未对 GIF 单独配置（例如 LFS 规则）。

## 依赖与外部交互

1. 仓内依赖
- 直接依赖：`README.md` 对 `demo.gif` 的相对路径引用（`README.md:11`）。
- 研究流程依赖：`.ops` 脚本将其作为 FILE 项管理（见上文脚本路径）。

2. 外部交互
- 对终端/运行时无直接外部 API 调用。
- 在代码托管平台展示 README 时，会触发浏览器对该二进制资源的下载与解码。

3. 配置、测试、脚本覆盖情况
- 未发现测试用例或 CI 规则专门验证 `demo.gif` 可访问性、体积阈值或内容时效。
- 未发现 Markdown lint/文档检查链路覆盖该资源引用关系。

## 风险、边界与改进建议

1. 仓库体积与历史膨胀风险
- GIF 属于大体积二进制，且采用整文件替换，历史版本会累积仓库对象体积。
- 建议：若后续高频更新演示素材，评估 `Git LFS` 或迁移为压缩率更优的 `mp4/webm` 并在 README 提供回退图。

2. 首屏加载成本
- `~11MB` 动图在低带宽或移动网络下会明显增加首屏加载时间。
- 建议：提供静态封面图 + 点击播放视频，或至少降低分辨率/帧率后再导出 GIF。

3. 可访问性边界
- 当前 `<img>` 无 `alt` 文本（`README.md:11`），屏幕阅读器语义弱。
- 建议：补充 `alt` 描述，例如“Claude Code terminal demo: audit and improve test coverage”。

4. 内容时效风险
- 动图画面包含具体版本与模型文案（如 `v2.0.0`, `Sonnet 4.5`），随产品迭代容易过时。
- 建议：建立“版本发布时检查 demo 录屏是否需更新”的文档资产清单。

5. 自动化缺口
- 目前无脚本校验 README 中相对媒体文件存在性。
- 建议：增加轻量检查（例如在 CI 中扫描 `README.md` 的本地媒体路径并验证文件存在/大小阈值）。
