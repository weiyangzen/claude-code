# FILE `README.md` 研究文档

## 场景与职责

`README.md` 是仓库根入口文档，承担“首屏路由器”职责：

1. 对外定位 Claude Code 与仓库用途
- 在首段给出产品能力摘要（终端代理式编码、自然语言任务执行）并链接官方文档（`README.md:7-9`）。

2. 安装与启动最短路径
- 提供跨平台安装命令矩阵（macOS/Linux、Windows、Homebrew、WinGet、已弃用 npm），并给出运行入口 `claude`（`README.md:13-47`）。

3. 本仓插件生态入口
- 从根 README 导向 `plugins/README.md`，把“仓库即插件样例集合”的信息传递给读者（`README.md:48-50`，`plugins/README.md:1-27`）。

4. 反馈、社区、数据与隐私声明
- 暴露 issue 反馈入口与 Discord 社区链接（`README.md:52-59`）。
- 提供数据收集用途和隐私政策跳转（`README.md:60-72`）。

## 功能点目的

1. 快速认知
- 用 badge 与简短描述降低首次理解成本（`README.md:3-7`）。

2. 安装渠道分流
- 通过“Recommended/Deprecated”标签引导用户走官方安装脚本，避免继续放大 npm 安装路径（`README.md:15,21-44`）。

3. 仓库内容导航
- 告知该仓库不仅是说明页，还包含可复用插件目录（`README.md:50`，`plugins/README.md:13-27`）。

4. 反馈闭环
- 让用户在产品内（`/bug`）或 GitHub issue 两条路径提交问题（`README.md:54`）。

5. 合规与信任沟通
- 明确数据使用与隐私链接，降低用户对遥测和会话数据处理的不确定性（`README.md:62-72`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 文档结构协议（Markdown）

`README.md` 使用标准 Markdown + 少量 HTML 混合：
- 徽章与引用链接定义：`[npm]: ...`（`README.md:3-5`）。
- 演示图通过 `<img src="./demo.gif" />` 引用仓库本地资产（`README.md:11`）。
- 安装命令放入 fenced code block，便于复制执行（`README.md:22-44`）。

### 2) 安装流程协议

文档隐含的执行流程如下：
1. 按平台选择安装命令（`curl|bash` / `brew` / `irm|iex` / `winget` / `npm`）。
2. 安装完成后进入项目目录。
3. 执行 `claude` 启动（`README.md:46`）。

### 3) 依赖跳转链路

`README.md` 主要通过超链接分发到下游文档/页面：
- 官方产品文档：`code.claude.com/docs/...`（`README.md:9,17,66`）。
- 本仓插件总览：`./plugins/README.md`（`README.md:50`）。
- 法务页：Commercial Terms 与 Privacy Policy（`README.md:72`）。

### 4) 与仓内流程的耦合点

虽然没有脚本直接“解析”根 README 作为输入，但存在明确上下文依赖：
- Issue 模板把 `README.md` 作为模型行为示例文件（`model_behavior.yml:52,64,79,94,136`）。
- 变更日志记录过 `@README.md#...` 锚点解析修复，说明 README 属于被高频引用文件（`CHANGELOG.md:716`）。
- 研究流程会将该文件纳入 checklist/todo 并要求输出对应研究文档（`.ops/research_guard.sh:320-337`）。

## 关键代码路径与文件引用

1. 主文件
- `README.md:1-72`

2. 直接下游引用
- `README.md:11` -> `demo.gif`
- `README.md:50` -> `plugins/README.md:1-77`

3. 间接支撑关系
- 插件真实清单与元数据：`.claude-plugin/marketplace.json:10-149`
- 插件目录文档（安装与结构）：`plugins/README.md:29-61`

4. 与问题反馈流程相关
- 根 README 的 bug 入口：`README.md:52-55`
- Issue 模板中 README 例子：`.github/ISSUE_TEMPLATE/model_behavior.yml:52-99`

5. 与研究自动化相关
- 批次任务模板（要求写研究文档、勾选 checklist）：`.ops/research_guard.sh:320-345`
- todo 生成：`.ops/generate_daily_research_todo.sh:15-39`

## 依赖与外部交互

1. 外部网络资源
- `shields.io`（badge 图片，`README.md:3,5`）。
- `npmjs.com`（包页面，`README.md:3`）。
- `claude.ai/install.sh`、`claude.ai/install.ps1`（安装脚本，`README.md:23,33`）。
- `code.claude.com`（官方文档，`README.md:9,17,66`）。
- `anthropic.com/discord`（社区，`README.md:58`）。
- `github.com/anthropics/claude-code/issues`（问题反馈，`README.md:54`）。
- `anthropic.com/legal/*`（法务条款，`README.md:72`）。

2. 本地仓库依赖
- `demo.gif`：README 演示图资源。
- `plugins/README.md`：插件入口索引。

3. 配置、测试、脚本交互
- 未发现 CI/脚本对 `README.md` 的结构化校验（无 README lint/test 链路）。
- 该文件主要由人工维护与人工消费；自动化系统仅把它纳入研究任务管理。

## 风险、边界与改进建议

1. 安装路径信息存在跨文档不一致
- 根 README 明确 npm 安装已弃用（`README.md:15,41-44`），但 `plugins/README.md` 安装段仍直接建议 `npm install -g ...`（`plugins/README.md:33-36`）。
- 建议：统一安装建议，避免用户在不同入口看到相互冲突指引。

2. 文档域名存在双入口，认知成本偏高
- 根 README 混用 `code.claude.com` 与 `docs.claude.com` 生态（后者出现在 `plugins/README.md:9,45,75-77`）。
- 建议：在根 README 增加一句“主文档域名说明/重定向说明”。

3. 安装脚本是远程执行命令，缺少安全提示
- `curl|bash` 与 `irm|iex` 对新用户门槛低但风险高。
- 建议：补充校验/审阅脚本链接说明（例如先下载再审阅）。

4. 反馈入口与安全漏洞入口未在同一处聚合
- 根 README 仅给 bug 入口，未直接链接 `SECURITY.md`。
- 建议：在 Reporting Bugs 旁新增“Security Vulnerability Disclosure”指向 `SECURITY.md`。

5. 边界说明
- 该 README 定位是“入口与路由”，不是完整产品手册；大量细节依赖外部文档页面。
- 因此其质量关键在于：链接有效性、路径一致性、安装策略一致性，而非内容全面性。
