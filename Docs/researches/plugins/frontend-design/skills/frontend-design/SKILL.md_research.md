# FILE `plugins/frontend-design/skills/frontend-design/SKILL.md` 研究文档

## 场景与职责

`plugins/frontend-design/skills/frontend-design/SKILL.md` 是 `frontend-design` 插件的核心行为规范文件，属于“Skill 协议层”，不直接执行脚本，而是定义 Claude 在前端任务中的设计决策与实现约束。

它在仓库中的职责链路如下：

1. 上游调用方（让该 Skill 被发现/触发）
- marketplace 注册插件源目录：`.claude-plugin/marketplace.json:73-81`。
- 插件总览声明该插件包含并自动用于前端工作的 Skill：`plugins/README.md:21`。
- 插件 README 提供自然语言触发样例：`plugins/frontend-design/README.md:16-22`。
- 自动发现机制扫描 `skills/*/SKILL.md`：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:343-347`。

2. 下游被调用方（Skill 触发后影响谁）
- Claude 的前端生成流程（先做设计方向决策，再写 HTML/CSS/JS/React/Vue 代码）：`plugins/frontend-design/skills/frontend-design/SKILL.md:13-25`。
- 输出代码的视觉与工程风格（字体、色彩、动画、布局、背景细节、复杂度控制）：`plugins/frontend-design/skills/frontend-design/SKILL.md:29-40`。

3. 文件边界
- 该插件是“纯 Skill 插件”，目录仅有 3 个文件：
  - `plugins/frontend-design/.claude-plugin/plugin.json`
  - `plugins/frontend-design/README.md`
  - `plugins/frontend-design/skills/frontend-design/SKILL.md`
- 不含 `commands/`、`agents/`、`hooks/`、`scripts/`、测试文件。

## 功能点目的

### 1) Frontmatter 触发语义
- `name: frontend-design` 与 `description` 给出任务触发范围（组件/页面/应用构建）：`SKILL.md:2-3`。
- 目的：将 Skill 自动关联到“前端实现请求”，减少手工指定。

### 2) 设计决策前置
- 明确“先理解上下文，再确定大胆审美方向”，并拆成 `Purpose/Tone/Constraints/Differentiation` 四维：`SKILL.md:13-17`。
- 目的：防止直接生成模板化页面，提升输出可辨识度。

### 3) 生产可用代码要求
- 要求输出是“真实可运行代码”，且具备生产可用质量：`SKILL.md:21-25`。
- 目的：避免只给视觉描述或概念稿，强制落到可执行实现。

### 4) 美学执行与反模式约束
- 正向约束：字体、色彩主题、动效、空间构图、背景细节：`SKILL.md:30-35`。
- 反向约束：禁止常见“AI 套路审美”（泛化字体、套路渐变、模板布局）：`SKILL.md:36-38`。
- 目的：抑制同质化输出，提高“场景定制感”。

### 5) 实现复杂度匹配
- 明确“视觉方向决定代码复杂度”：`SKILL.md:40`。
- 目的：避免“极简设计配复杂实现”或“极繁设计配简陋实现”的错配。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 关键流程

1. 插件注册
- marketplace 条目把 `frontend-design` 映射到 `./plugins/frontend-design`：`.claude-plugin/marketplace.json:73-81`。

2. 插件启用与发现
- 读取 `.claude-plugin/plugin.json`：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:343`。
- 扫描 `skills/` 下含 `SKILL.md` 的子目录：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:346`。

3. Skill 触发
- 加载元数据（name/description）并进行任务匹配；命中后加载正文：`plugins/plugin-dev/skills/skill-development/SKILL.md:79-83,271-276`。

4. 运行期执行
- 执行“设计先行”流程与“美学规则”约束，最终输出前端代码：`plugins/frontend-design/skills/frontend-design/SKILL.md:13-40`。

### B. 关键数据结构

1. 插件 manifest（JSON）
- 文件：`plugins/frontend-design/.claude-plugin/plugin.json:1-9`
- 关键字段：`name/version/description/author`。
- 用途：插件身份、版本与作者元信息。

2. Skill 元数据（YAML frontmatter）
- 文件：`plugins/frontend-design/skills/frontend-design/SKILL.md:1-5`
- 字段：`name`、`description`、`license`。
- 用途：Skill 触发与治理元信息。

3. Skill 指令正文（Markdown 协议）
- 章节：`Design Thinking` + `Frontend Aesthetics Guidelines`。
- 用途：定义可执行的生成约束，而非可执行脚本。

### C. 协议与命令

1. 自动发现协议
- Skill 目录规范：`skills/<skill-name>/SKILL.md`：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:166-168`。
- 自动发现顺序：manifest -> commands -> agents -> skills -> hooks -> mcp：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:343-348`。

2. 渐进加载协议（Progressive Disclosure）
- metadata 常驻，正文按触发加载，资源按需加载：`plugins/plugin-dev/skills/skill-development/SKILL.md:79-85`。
- 当前 Skill 没有 `references/`、`scripts/`、`assets/`，仅使用前两层。

3. 研究/验证可用命令（本仓库内）
```bash
# 文件与调用关系扫描
rg -n "frontend-design" .claude-plugin plugins

# 验证插件目录构成
find plugins/frontend-design -type f | sort

# 检查是否存在测试或脚本（当前为空）
find plugins/frontend-design -type f | rg '(test|spec|__tests__|\.sh$|\.py$|\.ts$|\.js$)'
```

4. 插件级手工验证命令（规范建议）
- `cc --plugin-dir /path/to/plugin`：`plugins/plugin-dev/skills/skill-development/SKILL.md:284-292`。

## 关键代码路径与文件引用

核心对象：
- `plugins/frontend-design/skills/frontend-design/SKILL.md:1-42`

调用方（上游）：
- `.claude-plugin/marketplace.json:73-81`（插件注册与 source 路由）
- `plugins/README.md:21`（插件总览与能力声明）
- `plugins/frontend-design/README.md:7-22`（触发样例与行为说明）
- `plugins/plugin-dev/skills/plugin-structure/SKILL.md:343-347`（自动发现机制）

被调用方（下游）：
- Claude 的前端代码生成过程（由 SKILL 文本规则约束）

配置文件：
- `plugins/frontend-design/.claude-plugin/plugin.json:1-9`
- `plugins/frontend-design/skills/frontend-design/SKILL.md:1-5`（frontmatter）

测试与脚本：
- `plugins/frontend-design` 目录无测试、无脚本、无命令实现文件（实测文件清单仅 3 个）。

文档：
- `plugins/frontend-design/README.md:1-31`
- `plugins/README.md:21`

## 依赖与外部交互

### 1) 内部依赖

- 依赖 marketplace 条目与 `source` 路径正确，否则插件不可发现：`.claude-plugin/marketplace.json:73-81`。
- 依赖 plugin manifest 放在 `.claude-plugin/plugin.json` 并可解析：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:343`。
- 依赖 Skill frontmatter 描述质量来决定命中率：`plugins/plugin-dev/skills/skill-development/SKILL.md:44`。

### 2) 外部交互

- 运行时无 API 调用、无 shell 脚本执行、无 MCP 配置。
- 外部交互主要是文档链接：Frontend Aesthetics Cookbook（README 外链）：`plugins/frontend-design/README.md:26`。

### 3) 配置/测试/脚本依赖现状

- 配置：manifest + frontmatter。
- 测试：无内建自动化测试。
- 脚本：无技能辅助脚本。
- 结果：质量控制主要依赖文档约束与人工评审。

## 风险、边界与改进建议

### 风险

1. 许可声明悬挂风险
- `SKILL.md` 写明 `license: Complete terms in LICENSE.txt`（`SKILL.md:4`），但插件目录中没有 `LICENSE.txt`。

2. 元数据口径漂移风险
- 作者信息在 `plugin.json`、`marketplace.json`、README 三处格式不一致（单邮箱/双邮箱、姓名连接方式不同）。

3. 规则可验证性不足
- 规则集中在自然语言约束，没有脚本化质量门禁（可访问性、性能、动效降级、字体策略等）。

4. 触发描述精度风险
- 现有 frontmatter 描述可触发前端任务，但未按“第三人称 + 具体触发短语”最佳实践组织（建议见 `skill-development/SKILL.md:44`），可能影响触发稳定性。

5. 规范扩展拥塞风险
- 当前无 `references/` 分层，后续新增规则将持续堆积到单文件，维护成本会升高。

### 边界

1. 该文件是策略规范，不是执行器。
2. 不负责构建、测试、发布，不直接改动用户工程文件。
3. 不包含 command/agent/hook/mcp 组件逻辑。

### 改进建议

1. 修复许可引用
- 增加 `plugins/frontend-design/LICENSE.txt`，或把 frontmatter 许可说明改为仓库现有许可路径。

2. 强化触发描述
- 将 frontmatter `description` 重写为第三人称且包含更具体触发短语（组件、落地页、仪表盘、设计系统等关键词）。

3. 增加 `references/`
- 将可访问性、响应式、性能预算、动效降级、语义 HTML 清单拆到引用文档，保持 `SKILL.md` 核心简洁。

4. 增加 `examples/` 与轻量校验脚本
- 示例覆盖至少 3 类前端任务（dashboard、landing、settings）。
- 脚本可做静态检查（如 `prefers-reduced-motion`、基本语义标签、CSS 变量使用率）。

5. 做元数据一致性治理
- 统一 `plugin.json`、marketplace、README 的作者/描述字段来源，减少长期漂移。
