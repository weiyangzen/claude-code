# plugins/plugin-dev/skills/plugin-settings/examples/create-settings-command.md 研究

## 场景与职责

`plugins/plugin-dev/skills/plugin-settings/examples/create-settings-command.md` 是 `plugin-settings` 技能中的“交互式创建配置文件”示例命令，职责是给插件作者一份可复制的命令模板，用于通过问答收集偏好并写入 `.claude/*.local.md`。

它在目录中的定位是：

1. 上游由 `plugin-settings/SKILL.md` 在“Creating Settings Files”与“Example Files”章节引用，作为命令侧落地范式（`plugins/plugin-dev/skills/plugin-settings/SKILL.md:272-287`, `517-523`）。
2. 在 `create-plugin` 流程中，它对齐 Phase 5 的 Settings 子任务：创建模板、实现读取、加入 `.gitignore`（`plugins/plugin-dev/commands/create-plugin.md:220-226`）。
3. 在 `plugin-dev/README.md` 中，它属于 Plugin Settings 的 3 个 example 之一（`plugins/plugin-dev/README.md:127-133`, `286-293`）。

## 功能点目的

1. 用 frontmatter 声明命令能力边界  
   通过 `description` 与 `allowed-tools: ["Write", "AskUserQuestion"]` 将命令限制在“交互采集 + 写文件”两类操作（`create-settings-command.md:1-4`）。

2. 交互采集核心配置  
   第一步定义了两道单选问题：是否启用、校验模式，目标是把用户偏好显式化（`create-settings-command.md:12-55`）。

3. 定义答案到配置字段的映射  
   第二步给出 `answers["0"]`、`answers["1"]` 的解释，把问答结果映射到 `enabled` 与 mode 字段（`create-settings-command.md:57-63`）。

4. 生成标准 `.local.md` 文件  
   第三步要求用 `Write` 产出 YAML frontmatter + markdown body 的双区结构（`create-settings-command.md:64-81`）。

5. 补齐运行期注意事项  
   第四步要求回告路径、当前配置、手工编辑方式、重启生效、gitignore 策略（`create-settings-command.md:83-90`）。

6. 约束输入安全  
   “Implementation Notes”要求校验模式值、数值类型、路径安全、自由文本清洗（`create-settings-command.md:92-98`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 数据结构

本命令输出目标是一个 markdown 文件，采用 `plugin-settings` 约定的数据协议：

```markdown
---
enabled: true|false
validation_mode: strict|standard|lenient
max_file_size: 1000000
notify_on_errors: true
---

# Plugin Configuration
...
```

对应 `plugin-settings` 总体协议（frontmatter + body，文件位于 `.claude/`）见 `plugins/plugin-dev/skills/plugin-settings/SKILL.md:11-19`, `22-40`。

### 2) 关键流程

1. 触发命令后调用 `AskUserQuestion`，一次提交多问题（`create-settings-command.md:14-55`）。
2. 读取回答并映射为布尔/枚举值（`create-settings-command.md:59-63`）。
3. 使用 `Write` 落盘 `.claude/my-plugin.local.md`（`create-settings-command.md:66-81`）。
4. 向用户输出“已创建 + 生效方式 + 生命周期”说明（`create-settings-command.md:85-90`）。

### 3) 协议与约束

1. 命令前置协议  
   该文件本身是命令 markdown，使用 YAML frontmatter 声明元数据与工具权限（`create-settings-command.md:1-4`）。

2. 交互问答协议  
   AskUserQuestion 问题对象包含 `question`、`header`、`multiSelect`、`options`，与 command-development 交互模式文档一致（`plugins/plugin-dev/skills/command-development/references/interactive-commands.md:68-120`）。

3. 运行期协同协议  
   生成的 `.local.md` 需被 Hook/Command/Agent 消费，且变更通常要求重启会话后生效（`plugins/plugin-dev/skills/plugin-settings/SKILL.md:60-134`, `367-383`）。

### 4) 可执行命令关联

虽然该文件是模板命令，不含 shell 执行体，但它的输出文件可直接被以下工具脚本消费：

```bash
bash plugins/plugin-dev/skills/plugin-settings/scripts/parse-frontmatter.sh .claude/my-plugin.local.md enabled
bash plugins/plugin-dev/skills/plugin-settings/scripts/validate-settings.sh .claude/my-plugin.local.md
```

对应实现：`parse-frontmatter.sh:7-58`、`validate-settings.sh:7-101`。

## 关键代码路径与文件引用

1. 目标文件  
   `plugins/plugin-dev/skills/plugin-settings/examples/create-settings-command.md`

2. 直接上游引用  
   `plugins/plugin-dev/skills/plugin-settings/SKILL.md:272-287`, `517-523`  
   `plugins/plugin-dev/README.md:127-133`, `286-293`  
   `plugins/plugin-dev/commands/create-plugin.md:157-163`, `220-226`

3. 同目录协同对象  
   `plugins/plugin-dev/skills/plugin-settings/examples/example-settings.md`  
   `plugins/plugin-dev/skills/plugin-settings/examples/read-settings-hook.sh`

4. 下游脚本/验证链  
   `plugins/plugin-dev/skills/plugin-settings/scripts/parse-frontmatter.sh`  
   `plugins/plugin-dev/skills/plugin-settings/scripts/validate-settings.sh`

5. 相关规范文档  
   `plugins/plugin-dev/skills/command-development/SKILL.md:98-146`  
   `plugins/plugin-dev/skills/command-development/references/interactive-commands.md:68-120`, `462-489`

## 依赖与外部交互

1. Claude Code 工具依赖  
   `AskUserQuestion`（收集偏好）、`Write`（落盘配置）。

2. 文件系统交互  
   读写项目根下 `.claude/my-plugin.local.md`；预期还会与 `.gitignore` 协作（`plugin-settings/SKILL.md:327-336`）。

3. 跨组件交互  
   输出文件由 Hook/Command/Agent 在后续会话中读取，形成“配置驱动行为”闭环（`plugin-settings/SKILL.md:60-134`）。

4. 测试/验证交互  
   本目录无自动化测试；主要依赖 `validate-settings.sh` 与手动命令回归。

## 风险、边界与改进建议

1. 字段命名不完全统一  
   本模板写 `validation_mode`，而其它示例常用 `mode` 或 `strict_mode`；复制后若读取脚本字段不匹配会失效。  
   建议：在技能内统一主字段（例如 `mode`）并在脚本中兼容别名。

2. 回答索引耦合顺序  
   `answers["0"]`/`answers["1"]` 对问题顺序敏感，后续维护易引入错配。  
   建议：在命令实现中增加显式映射校验，或用稳定键名（若工具协议支持）。

3. 缺少失败分支说明  
   文档未覆盖“用户取消/空回答/非法回答”的显式处理流程。  
   建议：补充失败分支与重试逻辑，参照 interactive-commands 的错误处理建议（`interactive-commands.md:472-489`）。

4. 创建前置条件未写全  
   文档默认 `.claude/` 已存在，未显式要求 `mkdir -p .claude`。  
   建议：在 Step 3 前加入目录存在性检查。

5. 安全说明与实现脱节  
   末尾提出校验路径与清洗输入，但正文模板未给出实际实现步骤。  
   建议：补充最小实现片段，至少覆盖模式白名单、数字范围、路径穿越检查。

