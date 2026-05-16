# FILE 研究：`plugins/plugin-dev/skills/plugin-settings/scripts/parse-frontmatter.sh`

## 场景与职责

`parse-frontmatter.sh` 是 `plugin-settings` skill 提供的轻量解析工具，面向“手工开发流程”而非运行时自动流程。其职责是从 `.claude/*.local.md` 中提取 frontmatter，供 hook/command 脚本快速读取配置。

- 上游（调用语义来源）：
  - `plugins/plugin-dev/skills/plugin-settings/SKILL.md:525-530` 将其定义为 utility script。
  - `plugins/plugin-dev/README.md:127-133` 将其作为 Plugin Settings 的资源暴露给使用者。
- 下游（被作用对象）：
  - 约定格式的 settings 文件（`.claude/plugin-name.local.md`）。
  - 典型消费者模式见 `plugins/plugin-dev/skills/plugin-settings/examples/read-settings-hook.sh:16-22`（同类 frontmatter 提取链路）。

仓库内未发现 CI/workflow/运行时代码对该脚本的自动调用，当前定位是开发期辅助工具。

## 功能点目的

1. 无字段参数：输出 frontmatter 全量内容，便于调试和二次处理。
2. 指定字段参数：提取单个键值（例如 `enabled`），便于直接在 Bash 条件分支使用。
3. 对无文件/无 frontmatter 做快速失败，避免后续逻辑在空值上继续运行。
4. 提供 `-h/--help` 使用说明，降低脚本接入成本。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 关键流程

1. 参数与帮助：
   - `show_usage()` 输出示例，入口判断在 `plugins/plugin-dev/skills/plugin-settings/scripts/parse-frontmatter.sh:23-25`。
2. 输入校验：
   - 仅校验 `-f`（文件存在），位于 `:31-34`。
3. frontmatter 提取：
   - `sed -n '/^---$/,/^---$/{ /^---$/d; p; }'`，位于 `:37`。
4. 模式分支：
   - 未给字段：直接 `echo "$FRONTMATTER"`，位于 `:45-48`。
   - 给字段：`grep "^${FIELD}:" | sed ...` 提取，位于 `:51`。
5. 结果判空与退出：
   - `FRONTMATTER` 空时报错（`:39-42`），`VALUE` 空时报错（`:53-56`）。

### 数据结构与协议

- 输入协议：Markdown 文件中 YAML 风格 frontmatter（`---` 包围）+ markdown body。
- 输出协议：
  - 成功：stdout 输出 frontmatter 文本或字段值，退出码 `0`。
  - 失败：stderr 输出错误，退出码 `1`。

### 实测命令与行为

1. `bash .../parse-frontmatter.sh <valid-file>`：返回完整 frontmatter，退出 `0`。
2. `bash .../parse-frontmatter.sh <valid-file> enabled`：返回 `true`，退出 `0`。
3. `bash .../parse-frontmatter.sh <valid-file> missing`：在 `set -euo pipefail` 下，`grep` 无匹配直接退出，脚本返回 `1`，且未稳定打印自定义错误文案。
4. `bash .../parse-frontmatter.sh <multi-marker-file>`：会合并多个 `--- ... ---` 区段内容（非仅首段 frontmatter）。

## 关键代码路径与文件引用

- 目标文件：
  - `plugins/plugin-dev/skills/plugin-settings/scripts/parse-frontmatter.sh:1-59`
- 直接文档入口（调用方语义）：
  - `plugins/plugin-dev/skills/plugin-settings/SKILL.md:525-530`
  - `plugins/plugin-dev/README.md:127-133`
- 协议与示例依赖：
  - `plugins/plugin-dev/skills/plugin-settings/references/parsing-techniques.md:24-47`
  - `plugins/plugin-dev/skills/plugin-settings/examples/read-settings-hook.sh:16-22`
  - `plugins/plugin-dev/skills/plugin-settings/examples/example-settings.md:90-137`
- 间接相关流程（插件创建时要求接入 settings 模式）：
  - `plugins/plugin-dev/commands/create-plugin.md:220-225`

## 依赖与外部交互

### 运行依赖

- Shell 与基础命令：`bash`、`sed`、`grep`。
- 采用 `set -euo pipefail`（`parse-frontmatter.sh:5`）强化失败传播。

### 外部交互

- 文件系统：只读目标 settings 文件。
- 网络：无。
- 进程协议：stdout/stderr + 退出码。

### 测试与验证现状

- 目录内未提供该脚本的自动化测试（如 bats/shunit2）。
- 当前质量保障主要依赖手工执行与文档示例回归。

## 风险、边界与改进建议

1. 风险（高）：frontmatter 边界过宽  
   现状：`sed` 范围表达式会覆盖文件中多个 marker 区段。  
   建议：只接受“文件起始第一段 frontmatter”，超出两条 marker 直接报错。

2. 风险（高）：字段缺失错误路径不稳定  
   现状：`VALUE=$(...grep...)` 在无匹配时因 `set -e` 直接退出，`:53-56` 分支不一定执行。  
   建议：将提取改为 `grep ... || true` 后统一判空，保证错误信息一致。

3. 风险（中）：字段名被当作正则  
   现状：`grep "^${FIELD}:"` 对特殊字符无转义。  
   建议：限制字段名字符集（如 `^[a-zA-Z_][a-zA-Z0-9_]*$`），或对 `FIELD` 做正则转义。

4. 风险（中）：YAML 语义仅“行文本”级解析  
   现状：不支持复杂 YAML（嵌套、多行字符串、注释内冒号等）。  
   建议：在 `--strict` 模式下可选接入 `yq`，或至少在文档中声明“仅支持简单 key:value”。

5. 改进（工程化）：补最小回归集  
   建议新增 `scripts/tests/parse-frontmatter.bats`，覆盖：标准文件、缺字段、无 marker、多 marker、字段含引号等场景。
