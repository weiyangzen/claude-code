# FILE 研究：`plugins/plugin-dev/skills/plugin-settings/scripts/validate-settings.sh`

## 场景与职责

`validate-settings.sh` 是 `plugin-settings` skill 的结构校验工具，用于在 settings 文件被 hook/command 消费前做“快速体检”。它偏向开发期人工校验，不是严格 YAML linter，也不是 CI 强校验器。

- 上游（调用语义来源）：
  - `plugins/plugin-dev/skills/plugin-settings/SKILL.md:525-530`
  - `plugins/plugin-dev/README.md:127-133`
- 下游（被检查对象）：
  - `.claude/plugin-name.local.md` 这类 frontmatter+body 文件。
  - 同协议读取方式由 `plugins/plugin-dev/skills/plugin-settings/examples/read-settings-hook.sh:16-22` 消费。

仓库内未发现自动调用该脚本的 workflow/test harness，当前主要通过手工命令执行。

## 功能点目的

1. 校验目标文件存在且可读，阻断显式 I/O 错误。
2. 校验 `---` marker 存在（至少 2 个），避免非 frontmatter 文件进入后续流程。
3. 校验 frontmatter 非空，提示最小结构完整性。
4. 展示检测到的 key:value 字段，便于人工排查。
5. 对常见布尔字段（`enabled`、`strict_mode`）给出类型提示。
6. 检查 markdown body 是否存在，并输出最终“结构有效”结论。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 关键流程

1. 参数检查与 usage：
   - 无参数直接退出 `1`，见 `plugins/plugin-dev/skills/plugin-settings/scripts/validate-settings.sh:8-19`。
2. 文件存在/可读：
   - `:27-38`。
3. marker 计数：
   - `MARKER_COUNT=$(grep -c '^---$' ... || echo "0")`，`if [ "$MARKER_COUNT" -lt 2 ]`，见 `:41-51`。
4. frontmatter 提取与判空：
   - `sed -n '/^---$/,/^---$/{ /^---$/d; p; }'`，见 `:55-60`。
5. 字段扫描与展示：
   - `grep '^[a-z_][a-z0-9_]*:' | while IFS=':' ...`，见 `:71-73`。
6. 布尔字段软校验：
   - 循环 `enabled strict_mode`，见 `:76-83`。
7. body 检查与汇总：
   - `awk '/^---$/{i++; next} i>=2'`，见 `:86-101`。

### 数据结构与协议

- 输入协议：frontmatter + markdown body。
- 输出协议：
  - 成功：stdout 输出检查过程与摘要，退出码 `0`。
  - 失败：stdout/stderr 输出错误，退出码 `1`。
  - 警告：布尔异常仅 warning，不改变成功退出码。

### 实测命令与行为

1. `bash .../validate-settings.sh <valid-file>`：校验通过，退出 `0`。
2. `bash .../validate-settings.sh <bad-bool-file>`：输出两条布尔 warning，但仍“结构有效”，退出 `0`。
3. `bash .../validate-settings.sh <no-marker-file>`：出现 `integer expression expected`（来自 `MARKER_COUNT` 多行值），随后进入 frontmatter 空错误，退出 `1`。
4. `bash .../validate-settings.sh plugins/plugin-dev/skills/plugin-settings/examples/example-settings.md`：将文档中多个代码块当作 marker 区段解析，字段列表被拼接并出现重复，最终仍退出 `0`。

## 关键代码路径与文件引用

- 目标文件：
  - `plugins/plugin-dev/skills/plugin-settings/scripts/validate-settings.sh:1-101`
- 直接文档入口（调用方语义）：
  - `plugins/plugin-dev/skills/plugin-settings/SKILL.md:525-530`
  - `plugins/plugin-dev/README.md:127-133`
- 协议与实现参考：
  - `plugins/plugin-dev/skills/plugin-settings/references/parsing-techniques.md:24-47`
  - `plugins/plugin-dev/skills/plugin-settings/references/parsing-techniques.md:190-260`
- 消费侧示例（被调用方语义）：
  - `plugins/plugin-dev/skills/plugin-settings/examples/read-settings-hook.sh:16-27`
  - `plugins/plugin-dev/skills/plugin-settings/examples/example-settings.md:90-137`
- 插件流程上下文（配置接入要求）：
  - `plugins/plugin-dev/commands/create-plugin.md:220-225`

## 依赖与外部交互

### 运行依赖

- `bash`、`grep`、`sed`、`awk`、`wc`、`tr`。
- `set -euo pipefail`（`validate-settings.sh:5`）控制失败传播。

### 外部交互

- 文件系统：只读输入文件，不修改项目文件。
- 网络：无。
- 控制台：大量人类可读日志（包含 emoji），适合交互式使用。

### 测试与验证现状

- 无自动化测试脚本；主要依赖手工运行。
- 输出文案面向人工阅读，缺少机器可解析模式（如 `--json`）。

## 风险、边界与改进建议

1. 风险（高）：marker 计数分支存在错误拼接  
   现状：`grep -c ... || echo "0"` 在无匹配时可能形成 `0\\n0`，导致 `-lt` 比较报 `integer expression expected`。  
   建议：改为 `MARKER_COUNT=$(grep -c '^---$' "$SETTINGS_FILE" 2>/dev/null || true); MARKER_COUNT=${MARKER_COUNT:-0}`，并做纯数字校验。

2. 风险（高）：frontmatter 边界校验不严格  
   现状：只要求 marker 至少 2 个，不要求“开头即 frontmatter 且只一段”。  
   建议：验证首行 marker、第二个 marker 的位置，并拒绝附加 marker 或多段 frontmatter。

3. 风险（中）：布尔校验是软警告  
   现状：`enabled: maybe` 仍返回成功。  
   建议：增加 `--strict`，在 strict 下将类型错误升级为失败退出码。

4. 风险（中）：同名字段多值时行为不确定  
   现状：`grep "^enabled:"` 可能拿到多行，warning 信息可读性差。  
   建议：检测重复 key 并报错，或仅接受首个/最后一个并明确规则。

5. 风险（中）：仅“YAML-like”校验，未做真实 YAML 语法验证  
   现状：只检查是否包含冒号。  
   建议：可选接入 `yq` 做结构化验证，或至少增加键名白名单、字段类型映射和范围校验。

6. 改进（工程化）：补充自动化测试与机器输出  
   建议新增 `scripts/tests/validate-settings.bats`，并增加 `--quiet`、`--json`，方便集成到 CI 或其他脚本。
