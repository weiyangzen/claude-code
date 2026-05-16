# FILE `LICENSE.md` 研究文档

## 场景与职责

`LICENSE.md` 是仓库根级法律边界声明文件，当前内容极简但职责明确：

1. 版权归属声明
- 明确版权属于 Anthropic PBC（`LICENSE.md:1`）。

2. 使用条款锚点
- 指向 Anthropic Commercial Terms of Service，说明“使用受商业条款约束”，而非典型开源许可证（`LICENSE.md:1`）。

3. 仓库法务入口角色
- 与 README 中法务链接共同构成合规入口（`README.md:72`）。

## 功能点目的

1. 快速给出许可边界
- 通过单行文本让访问者立即知道此仓库并非默认开源授权模式。

2. 将完整条款外置到权威页面
- 不在仓库内复制长篇法律文本，而是引用官方法律页面，减少仓库内法务文本维护成本。

3. 为生态文档提供一致法务基线
- 插件与技能相关文档涉及 license 字段时，可回指到仓库/组织级法务语义（例如 skill frontmatter 的 license 声明实践）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 文档协议实现

- 文件仅 1 行 Markdown 文本 + 1 个超链接（`LICENSE.md:1`）。
- 没有 SPDX 标识符、没有附带许可证全文。
- 语义是“授权入口指针”，不是“自包含许可证文档”。

### 2) 语义流程（人类消费链路）

1. 用户/贡献者打开仓库看到 `LICENSE.md`。
2. 读取版权与“受商业条款约束”的声明。
3. 跳转外部法务页查看完整 Terms。

### 3) 与仓内其他协议的关系

1. 与 README 法务段形成互证
- `README.md:72` 与 `LICENSE.md:1` 都指向 commercial-terms，避免单点说明丢失。

2. 与插件生态的 license 字段形成对照
- 插件开发参考要求 manifest 使用 SPDX（`manifest-reference.md:164-180`），并建议在插件根包含 `LICENSE` 文件（`manifest-reference.md:552`）。
- 某些 skill frontmatter 使用 `license: Complete terms in LICENSE.txt`（`frontend-design/SKILL.md:4`，`skill-creator-original.md:4`），但对应目录并未提供 `LICENSE.txt`，存在口径不一致风险。

3. 与研究自动化交互
- 研究 guard 将该文件作为 FILE 任务对象，输出 `LICENSE.md_research.md` 并要求勾选 checklist（`.ops/research_guard.sh:320-345`）。

## 关键代码路径与文件引用

1. 核心文件
- `LICENSE.md:1`

2. 法务关联文件
- `README.md:72`（同一 Commercial Terms 链接）

3. 插件/技能 license 语义关联
- `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:164-180`
- `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:552`
- `plugins/frontend-design/skills/frontend-design/SKILL.md:4`
- `plugins/plugin-dev/skills/skill-development/references/skill-creator-original.md:4`

4. 研究流程与状态文件
- `.ops/research_guard.sh:320-345`
- `Docs/researches/blueprint_checklist.md:144`
- `Docs/researches/todos_20260320.md:14`

## 依赖与外部交互

1. 外部依赖
- `https://www.anthropic.com/legal/commercial-terms`（法务条款主页面）。

2. 本地依赖
- 无脚本、无配置读取 `LICENSE.md` 内容并参与运行逻辑。
- 主要消费方式是 GitHub UI 人工阅读与法务审阅。

3. 测试与自动化
- 未发现针对 `LICENSE.md` 的 CI 校验（例如链接可达性校验、许可证格式校验）。
- 当前自动化仅体现在研究流程追踪（checklist/todo），不验证法务文本完整性。

## 风险、边界与改进建议

1. 风险：文件名易引发“开源许可证”误解
- `LICENSE.md` 常被默认理解为 MIT/Apache/GPL 全文；当前是商业条款指针，预期差较大。
- 建议：在首句增加 “This repository is proprietary” 等更显式措辞。

2. 风险：缺少机器可识别许可证标识
- 未使用 SPDX 字段，不利于自动化合规扫描工具直接分类。
- 建议：补充 SPDX 风格声明（例如 `LicenseRef-Anthropic-Commercial` 或清晰的 `UNLICENSED` 口径）。

3. 风险：仓内 license 口径不一致
- 部分 skill frontmatter 指向不存在的 `LICENSE.txt`，与根 `LICENSE.md` 不一致。
- 建议：统一成可解析、可访问的单一路径，并在插件模板中固化。

4. 风险：外链单点依赖
- 若法律页面 URL 结构变化，仓内不会自动发现。
- 建议：增加链接巡检（CI link check）或在 README/LICENSE 双处维护冗余说明。

5. 边界说明
- `LICENSE.md` 不参与运行时代码路径，不影响命令执行。
- 其价值在法务与分发治理层；质量指标是“清晰、稳定、可机器识别”。
