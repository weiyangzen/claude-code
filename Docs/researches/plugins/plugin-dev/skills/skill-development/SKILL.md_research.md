# plugins/plugin-dev/skills/skill-development/SKILL.md 研究

## 场景与职责

`plugins/plugin-dev/skills/skill-development/SKILL.md` 是 `plugin-dev` 里的“技能开发元规范”。它本身不提供业务能力，而是规定“如何设计、编写、验证和迭代其他技能”。

在插件体系中的职责分层：

1. 触发路由职责
- 通过 frontmatter `description` 声明明确触发语句（如 `create a skill`、`improve skill description`），用于让 Claude Code 在技能开发意图出现时命中该技能（`plugins/plugin-dev/skills/skill-development/SKILL.md:1-5`）。

2. 方法论职责
- 定义技能标准结构（`SKILL.md + references/examples/scripts/assets`）及 progressive disclosure 三层加载模型（metadata -> SKILL body -> resources）（`plugins/plugin-dev/skills/skill-development/SKILL.md:25-40,77-85`）。

3. 流程编排职责
- 给出 Step 1~6 的完整流程：样例澄清、复用资产规划、结构创建、正文编写、校验测试、迭代（`plugins/plugin-dev/skills/skill-development/SKILL.md:87-247`）。

4. 质量治理职责
- 通过写作规范、Validation Checklist、Common Mistakes、Quick Reference 固化技能质量门槛（`plugins/plugin-dev/skills/skill-development/SKILL.md:362-603`）。

上下文依赖定位：

- 上游调用方
1. `plugins/plugin-dev/README.md` 将其定义为第 7 个核心技能（`plugins/plugin-dev/README.md:175-193`）。
2. `/plugin-dev:create-plugin` 在 Phase 5 明确要求 “Skills: Load skill-development skill”（`plugins/plugin-dev/commands/create-plugin.md:157-180,371-375`）。

- 下游被调用方
1. `skill-reviewer` agent 承接技能质量评审（`plugins/plugin-dev/skills/skill-development/SKILL.md:225-230`，`plugins/plugin-dev/agents/skill-reviewer.md:40-46,60-79`）。
2. `references/skill-creator-original.md` 作为通用方法来源（`plugins/plugin-dev/skills/skill-development/SKILL.md:618-620`）。
3. 其他 6 个 plugin-dev 技能被作为“可学习样例”反向引用（`plugins/plugin-dev/skills/skill-development/SKILL.md:608-615`）。

## 功能点目的

1. 提高技能触发命中率
- 目的：把 `description` 从“介绍文案”变成“触发契约”，降低误触发与漏触发。
- 关键位点：`plugins/plugin-dev/skills/skill-development/SKILL.md:44,162-182,384-395,426-429`。

2. 控制上下文成本
- 目的：通过 progressive disclosure 把高频必读与低频细节分离，减少每次触发的上下文负担。
- 关键位点：`plugins/plugin-dev/skills/skill-development/SKILL.md:77-85,318-360`。

3. 标准化技能创建顺序
- 目的：避免先写大段 SKILL.md、后补资源的返工路径；强调“先样例、后资源、再正文”。
- 关键位点：`plugins/plugin-dev/skills/skill-development/SKILL.md:91-156,621-635`。

4. 统一写作协议
- 目的：增强“另一实例 Claude”可执行性，减少角色混乱和口语化指令。
- 规则：正文祈使/不定式；frontmatter 第三人称；避免 second person。
- 关键位点：`plugins/plugin-dev/skills/skill-development/SKILL.md:160-170,362-414`。

5. 建立可复核质量门禁
- 目的：把结构、描述、内容分层、引用完整性、测试项变成显式 checklist。
- 关键位点：`plugins/plugin-dev/skills/skill-development/SKILL.md:415-450`。

6. 将通用 skill-creator 适配为插件内模式
- 目的：从“通用 skill ZIP 打包分发”切换到“插件内 skills 目录管理 + 随插件交付”。
- 关键位点：
  - 插件适配声明：`plugins/plugin-dev/skills/skill-development/SKILL.md:146,269-281`
  - 通用原始流程（init/package）：`plugins/plugin-dev/skills/skill-development/references/skill-creator-original.md:132-200`

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 关键数据结构与协议

1. 技能目录结构协议

```text
skill-name/
├── SKILL.md                    # 必需
├── references/                 # 可选
├── examples/                   # 可选
├── scripts/                    # 可选
└── assets/                     # 可选（本文件多处说明）
```

- 结构定义：`plugins/plugin-dev/skills/skill-development/SKILL.md:27-40,261-267,545-578`。
- 约束：引用存在性必须可验证（`plugins/plugin-dev/skills/skill-development/SKILL.md:423,443`）。

2. frontmatter 触发协议

```yaml
---
name: Skill Name
description: This skill should be used when the user asks to "..."
version: 0.1.0
---
```

- 描述字段是触发路由核心，不是仅展示字段（`plugins/plugin-dev/skills/skill-development/SKILL.md:44,162-175`）。
- 明确给出正反例，约束“第三人称 + 具体短语”（`plugins/plugin-dev/skills/skill-development/SKILL.md:172-182,386-395,455-468`）。

3. 渐进加载协议（运行时）

- Level 1: metadata 常驻。
- Level 2: SKILL body 在触发时加载。
- Level 3: references/examples/scripts 按需加载。

对应定义：`plugins/plugin-dev/skills/skill-development/SKILL.md:79-85,271-277`。

### 2) 关键流程

1. 技能开发主流程（Step 1~6）

- Step 1: 以具体用户语句澄清使用场景。
- Step 2: 从场景反推可复用资源。
- Step 3: 在插件 `skills/` 下创建目录。
- Step 4: 先资源后正文，正文保持精简并引用资源。
- Step 5: 做结构、触发、写作、引用、示例、脚本与触发测试。
- Step 6: 基于真实使用迭代。

主流程定义：`plugins/plugin-dev/skills/skill-development/SKILL.md:91-247,621-635`。

2. create-plugin 工作流集成流程

- Phase 5 对 skills 组件固定加载 skill-development。
- 完成后建议调用 `skill-reviewer` 做专项评审。
- Phase 6 继续在插件级做整体验证。

流程锚点：`plugins/plugin-dev/commands/create-plugin.md:157-181,247-250,233-260,371-375`。

3. 插件自动发现流程

- Claude Code 扫描 `skills/`。
- 查找含 `SKILL.md` 的子目录。
- 先加载 metadata，再按触发加载正文，最后按需加载资源。

定义：`plugins/plugin-dev/skills/skill-development/SKILL.md:269-277`。
补充机制：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:339-348`。

### 3) 关键命令与交互约定

1. 插件技能目录初始化命令

```bash
mkdir -p plugin-name/skills/skill-name/{references,examples,scripts}
touch plugin-name/skills/skill-name/SKILL.md
```

来源：`plugins/plugin-dev/skills/skill-development/SKILL.md:141-144`。

2. 本地触发验证命令

```bash
cc --plugin-dir /path/to/plugin
```

来源：`plugins/plugin-dev/skills/skill-development/SKILL.md:286-292`。

3. 质量评审触发语句（Agent 交互）

```text
Review my skill and check if it follows best practices
```

来源：`plugins/plugin-dev/skills/skill-development/SKILL.md:225-230`。

### 4) 与通用 skill-creator 的关键差异实现

1. 初始化方式差异
- 通用版强调 `init_skill.py` 自动初始化（`.../skill-creator-original.md:132-153`）。
- plugin-dev 版本改为插件目录手工创建（`.../SKILL.md:137-147`）。

2. 分发方式差异
- 通用版强调 `package_skill.py` 打包 zip（`.../skill-creator-original.md:175-199`）。
- plugin-dev 版本明确“随插件分发，不单独打包”（`.../SKILL.md:278-281`）。

3. 验收方式差异
- plugin-dev 增加 `skill-reviewer` agent 评审闭环（`.../SKILL.md:225-230`，`plugins/plugin-dev/agents/skill-reviewer.md:47-98`）。

### 5) 配置、测试、脚本、文档现状核查

1. 配置
- 直接配置载体是每个 skill 的 `SKILL.md` frontmatter。
- 插件级结构规范依赖 `.claude-plugin/plugin.json` 与自动发现机制（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:41-48,343-348`）。

2. 测试
- `skill-development` 目录无自动化测试文件。
- 测试方式是本地安装后按触发词手工验证（`plugins/plugin-dev/skills/skill-development/SKILL.md:284-292,446-450`）。

3. 脚本
- 本目录无 `scripts/` 实体脚本。
- 校验动作主要通过 agent 审核和跨技能工具链间接完成。

4. 文档
- 主文档：`plugins/plugin-dev/skills/skill-development/SKILL.md`。
- 补充原文：`plugins/plugin-dev/skills/skill-development/references/skill-creator-original.md`。
- 工作流入口：`plugins/plugin-dev/commands/create-plugin.md`。
- 产品级导航：`plugins/plugin-dev/README.md`。

## 关键代码路径与文件引用

### 目标对象

1. `plugins/plugin-dev/skills/skill-development/SKILL.md`
- 研究主体；定义技能开发规范、流程、验证与最佳实践。

2. `plugins/plugin-dev/skills/skill-development/references/skill-creator-original.md`
- 原始通用方法论来源文件。

### 调用方（上游）

1. `plugins/plugin-dev/README.md:175-193`
- 暴露触发词、职责范围与资源定位。

2. `plugins/plugin-dev/commands/create-plugin.md:157-181,371-375`
- 在 skills 组件实现阶段明确加载本技能。

3. `plugins/plugin-dev/commands/create-plugin.md:247-250`
- 技能实现后调用 `skill-reviewer` 做专项审查。

4. `plugins/README.md:24`
- 在全仓插件索引层面标注 plugin-dev 的能力构成。

### 被调用方（下游）

1. `plugins/plugin-dev/agents/skill-reviewer.md:40-46,60-79,99-106`
- 把本技能中的质量规则转化为可执行审查流程。

2. `plugins/plugin-dev/skills/plugin-structure/SKILL.md:339-348`
- 提供自动发现机制的结构级背景。

3. `plugins/plugin-dev/skills/skill-development/references/skill-creator-original.md:132-200`
- 提供通用 skill 创建/打包原流程供对照。

4. `plugins/plugin-dev/skills/hook-development/SKILL.md`
5. `plugins/plugin-dev/skills/agent-development/SKILL.md`
6. `plugins/plugin-dev/skills/mcp-integration/SKILL.md`
7. `plugins/plugin-dev/skills/plugin-settings/SKILL.md`
8. `plugins/plugin-dev/skills/command-development/SKILL.md`
9. `plugins/plugin-dev/skills/plugin-structure/SKILL.md`
- 以上在本文件中被建议作为高质量模板学习对象（`plugins/plugin-dev/skills/skill-development/SKILL.md:608-615`）。

### 配置/测试/脚本/文档关联路径

1. 配置
- `plugins/plugin-dev/skills/skill-development/SKILL.md`
- `plugins/plugin-dev/skills/plugin-structure/SKILL.md`

2. 测试与验证
- `plugins/plugin-dev/commands/create-plugin.md`（Phase 6 验证编排）
- `plugins/plugin-dev/agents/skill-reviewer.md`（技能质量验证）
- `plugins/plugin-dev/agents/plugin-validator.md`（插件级校验）

3. 脚本
- 本目录无脚本；跨目录可复用脚本主要在 `hook-development`、`agent-development`、`plugin-settings` 下。

4. 文档
- `plugins/plugin-dev/README.md`
- `plugins/plugin-dev/commands/create-plugin.md`
- `plugins/README.md`

## 依赖与外部交互

1. 对 Claude Code 技能路由机制的依赖
- 依赖 frontmatter `description` 质量来决定触发命中。
- 交互模式：运行时自动加载，不需要手工 import。

2. 对 Agent 工具链的依赖
- 通过 `skill-reviewer` 完成质量复核；通过 `plugin-validator` 完成插件级一致性检查。
- 交互模式：在工作流里显式触发 agent。

3. 对命令行与本地文件系统的依赖
- 依赖 `mkdir`、`touch`、`cc --plugin-dir` 等基础命令。
- 依赖插件目录结构与 `skills/*/SKILL.md` 自动发现约定。

4. 对跨文档规范的一致性依赖
- 本文件与 `plugin-structure`、`create-plugin`、`README` 互相约束；任一更新未同步会引入规范漂移。

5. 外部交互边界
- 本技能文件不直接访问网络、不调用外部 API。
- 主要外部交互发生在用户用 Claude Code CLI 安装/启用插件时。

## 风险、边界与改进建议

1. 风险：文档长度超过自身建议上限
- 事实：`wc -w` 显示当前 `SKILL.md` 约 3196 词。
- 对照：文内建议理想 1500-2000，并强调 <3000 更佳（`plugins/plugin-dev/skills/skill-development/SKILL.md:190,329,587-588`）。
- 影响：触发后上下文占用增大，削弱 progressive disclosure 收益。
- 建议：把长反例和 quick reference 进一步下沉到 `references/`。

2. 风险：README 统计信息与实物不一致
- 事实：README 仍标注该技能 `Core SKILL.md (1,232 words)`（`plugins/plugin-dev/README.md:189-190`）。
- 影响：对加载成本与规模预期失真。
- 建议：同步 README 的字数和资源状态说明。

3. 风险：参考方法含当前仓库不可执行命令
- 事实：`skill-creator-original.md` 依赖 `scripts/init_skill.py` 与 `scripts/package_skill.py`（`.../skill-creator-original.md:142-187`），本仓库 `scripts/` 不含这两个文件。
- 影响：执行者可能误以为可直接运行。
- 建议：在 `Additional Resources` 增加“通用版差异提示”，明确该 reference 是方法来源而非本仓库可执行脚本入口。

4. 风险：示例与脚本依赖跨目录，学习路径较跳跃
- 事实：本目录仅有 `SKILL.md + reference` 两个文件，无 `examples/` 与 `scripts/` 实体。
- 影响：新手需要频繁跨目录跳转理解。
- 建议：补一个最小 `examples/minimal-skill-template.md`（或 `references/plugin-adaptation-note.md`）降低迁移成本。

5. 边界：本文件是“规范文档”，不是“可执行模块”
- 不承担代码运行；其主要输出是规则、流程和校验清单。
- 可验证能力主要依赖工作流执行者与 agent 审核是否落实。

6. 改进建议（可落地）
1. 将 `Validation Checklist` 抽取为可机检脚本（例如 frontmatter 第三人称、触发短语数量、引用存在性、字数阈值）。
2. 在 `/plugin-dev:create-plugin` Phase 6 增加“技能 lint”可选步骤，减少纯人工审查波动。
3. 给 `skill-creator-original.md` 顶部补“plugin-dev 适配差异”说明块并链接 `SKILL.md:146,278-281`。
4. 在 `plugin-dev/README.md` 建立“文档统计自动更新”机制，防止字数说明长期漂移。
