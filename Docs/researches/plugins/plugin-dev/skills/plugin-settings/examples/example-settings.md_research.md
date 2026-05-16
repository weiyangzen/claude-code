# plugins/plugin-dev/skills/plugin-settings/examples/example-settings.md 研究

## 场景与职责

`plugins/plugin-dev/skills/plugin-settings/examples/example-settings.md` 是 `plugin-settings` 的“模板样例库”，职责是集中展示 `.claude/*.local.md` 的常见结构，不直接作为运行时配置文件被加载。

它的上下文角色：

1. 作为 `plugin-settings/SKILL.md` 的 example 资源之一，服务于“复制模板再改造”的开发流程（`plugins/plugin-dev/skills/plugin-settings/SKILL.md:517-523`）。
2. 被 `plugin-dev/README.md` 计入 Plugin Settings 的 3 个示例资产（`plugins/plugin-dev/README.md:127-133`, `286-293`）。
3. 为 `read-settings-hook.sh` 与命令生成流程提供字段风格参考（`example-settings.md:115-134` 与 `read-settings-hook.sh:16-27`）。

## 功能点目的

1. 提供基础模板  
   最小化字段：`enabled` + `mode`，展示最短可用配置（`example-settings.md:7-16`）。

2. 提供高级模板  
   展示布尔、数值、列表、路径等多类型字段组合（`example-settings.md:22-47`）。

3. 提供 Agent 状态模板  
   映射到多 Agent 协作场景，包含任务号、依赖、协调会话等状态字段（`example-settings.md:53-89`）。

4. 提供 Feature Flag 模板  
   展示“enabled + feature list + experimental_mode”模式（`example-settings.md:95-113`）。

5. 提供 Hook 读取示例  
   内联 Bash 片段示范 quick-exit、frontmatter 提取、enabled 判定（`example-settings.md:119-134`）。

6. 补充生命周期规范  
   明确 `.gitignore` 条目与“编辑后需重启 Claude Code”提示（`example-settings.md:136-159`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 数据结构（文档内嵌模板）

该文件本质是“文档 + fenced code blocks”，并非单一 settings 文件。模板均遵循：

```markdown
---
key: value
---

# Markdown Body
...
```

与 `plugin-settings` 总体协议一致（`plugins/plugin-dev/skills/plugin-settings/SKILL.md:11-19`, `22-40`）。

### 2) 关键示例模式

1. 简化启停模式：`enabled + mode`（`example-settings.md:8-11`）。
2. 策略控制模式：`strict_mode/max_file_size/allowed_extensions/...`（`example-settings.md:24-33`）。
3. 状态机模式：`task_number/pr_number/dependencies`（`example-settings.md:55-62`）。
4. 特性开关模式：YAML 列表 `features`（`example-settings.md:98-103`）。

### 3) Hook 读取命令模式

文档内给出最小读取流程：

```bash
FRONTMATTER=$(sed -n '/^---$/,/^---$/{ /^---$/d; p; }' ".claude/my-plugin.local.md")
ENABLED=$(echo "$FRONTMATTER" | grep '^enabled:' | sed 's/enabled: *//')
```

对应位置：`example-settings.md:126-133`，与 `plugin-settings/SKILL.md:78-87`, `136-155` 一致。

### 4) 与工具脚本的关系（实测）

该文档可作为“阅读模板”，但不适合作为 `scripts/parse-frontmatter.sh` 的输入对象。实测（仓库根目录）：

```bash
bash plugins/plugin-dev/skills/plugin-settings/scripts/parse-frontmatter.sh \
  plugins/plugin-dev/skills/plugin-settings/examples/example-settings.md enabled
```

结果为 4 行 `true`（来自文档中多个代码块），说明该脚本会遍历到所有 `--- ... ---` 段，不区分“文档示例块”与“真实 frontmatter”。

同时执行：

```bash
bash plugins/plugin-dev/skills/plugin-settings/scripts/validate-settings.sh \
  plugins/plugin-dev/skills/plugin-settings/examples/example-settings.md
```

返回 0 且显示“Settings file structure is valid”，体现校验脚本对“多模板文档”存在假阳性空间。

## 关键代码路径与文件引用

1. 目标文件  
   `plugins/plugin-dev/skills/plugin-settings/examples/example-settings.md`

2. 直接上游  
   `plugins/plugin-dev/skills/plugin-settings/SKILL.md:11-19`, `22-40`, `517-523`  
   `plugins/plugin-dev/README.md:114-133`, `286-293`

3. 同目录协同  
   `plugins/plugin-dev/skills/plugin-settings/examples/create-settings-command.md`  
   `plugins/plugin-dev/skills/plugin-settings/examples/read-settings-hook.sh`

4. 下游解析/校验工具  
   `plugins/plugin-dev/skills/plugin-settings/scripts/parse-frontmatter.sh:37-58`  
   `plugins/plugin-dev/skills/plugin-settings/scripts/validate-settings.sh:40-101`

5. 参考实现文档  
   `plugins/plugin-dev/skills/plugin-settings/references/parsing-techniques.md:24-40`, `101-119`, `238-299`  
   `plugins/plugin-dev/skills/plugin-settings/references/real-world-examples.md:5-93`, `266-301`

## 依赖与外部交互

1. 运行时依赖  
   无直接运行时依赖；本文件是模板文档，不是可执行脚本。

2. 外部交互对象  
   交互对象是“插件开发者/维护者”，通过复制模板创建 `.claude/*.local.md`。

3. 工具链耦合  
   与 `sed/grep/awk` 解析模式和 `parse-frontmatter.sh`、`validate-settings.sh` 的输入约定相关，但其本身不调用这些脚本。

4. 生命周期依赖  
   与 `.gitignore`、Claude Code 重启生效提示绑定（`example-settings.md:140-157`）。

## 风险、边界与改进建议

1. 文档与真实配置文件语义混淆  
   该文件包含多个模板块，不应直接被解析脚本当作单个 settings 文件。  
   建议：在文件开头增加醒目说明“此文件仅为示例集合，不可直接用作 parse/validate 输入”。

2. 字段命名存在跨模板差异  
   同一主题出现 `mode`、`strict_mode`、`notification_level` 等多套命名。  
   建议：追加“推荐统一 schema”章节，给出主字段及兼容字段策略。

3. 示例路径含占位绝对路径  
   `custom_path: "/path/to/data"` 可能被原样复制，导致误配置。  
   建议：改为明显占位符（如 `"/ABSOLUTE/PATH/REQUIRED"`）并提示校验。

4. 重启规则需标注适用范围  
   文档写明“Changes require Claude Code restart”，但仓库里也有动态读取型插件（如 hookify 的 `.local.md` 规则可即时生效）。  
   建议：补一句“本结论适用于以会话加载为主的 hook 模式，动态读取实现可例外”。

5. 缺少与验证脚本的闭环示例  
   文档未给“从模板复制到验证通过”的完整命令。  
   建议：新增一段 `cp template -> edit -> validate-settings.sh` 的端到端流程。

