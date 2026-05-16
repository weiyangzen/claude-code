# parsing-techniques.md 研究

源文件：`plugins/plugin-dev/skills/plugin-settings/references/parsing-techniques.md`
目标文档：`Docs/researches/plugins/plugin-dev/skills/plugin-settings/references/parsing-techniques.md_research.md`

## 场景与职责

`parsing-techniques.md` 是 `plugin-settings` 技能下的“实现细节参考层”，用于把 `SKILL.md` 中的概念性描述下钻为可直接复制的 Bash 解析/更新套路。

在仓库中的职责边界：

1. 面向插件作者提供 `.claude/plugin-name.local.md` 的标准读写模型（frontmatter + body）。
2. 为同目录可执行脚本提供语义依据：
   - `plugins/plugin-dev/skills/plugin-settings/scripts/parse-frontmatter.sh`
   - `plugins/plugin-dev/skills/plugin-settings/scripts/validate-settings.sh`
3. 为示例与真实插件用法提供方法学共识：
   - `plugins/plugin-dev/skills/plugin-settings/examples/read-settings-hook.sh`
   - `plugins/ralph-wiggum/hooks/stop-hook.sh`
4. 被上游技能索引显式引用（`plugins/plugin-dev/skills/plugin-settings/SKILL.md:510-530`），并在 plugin-dev 总览中作为资源项公布（`plugins/plugin-dev/README.md:114-133`）。

## 功能点目的

### 1) 统一 settings 文件协议

文档先定义统一载体 `.claude/<plugin>.local.md`，并明确结构是“YAML frontmatter + markdown body”（`parsing-techniques.md:5-22`）。这样 hooks/commands/agents 在读取时可以有同一套语义假设。

### 2) 给出低依赖的可执行解析方案

核心方案以 `sed + grep + awk` 为主，避免额外依赖：

1. `sed` 提取 frontmatter（`parsing-techniques.md:28-40`）。
2. `grep|sed` 读取字符串/布尔/数字字段（`parsing-techniques.md:43-85`）。
3. `awk` 提取 body（`parsing-techniques.md:103-141`）。

### 3) 提供可持续状态更新策略

文档强调“临时文件 + 原子替换”（`parsing-techniques.md:190-236`），目的不是语法炫技，而是防止 hook 中断导致 `.local.md` 损坏。

### 4) 补齐工程化细节

后半段覆盖默认值回退、字段校验、调试、性能和 `yq` 替代路线（`parsing-techniques.md:238-549`），将“能跑”扩展到“可维护”。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 数据结构

统一数据结构为：

1. frontmatter：结构化键值（布尔、数字、字符串、列表）。
2. body：自由文本（常用作提示词、任务说明、补充上下文）。

文档示例位置：`parsing-techniques.md:9-22`。

### 关键流程 A：frontmatter 读取

主命令：

```bash
FRONTMATTER=$(sed -n '/^---$/,/^---$/{ /^---$/d; p; }' "$FILE")
```

配套字段读取模式：

```bash
VALUE=$(echo "$FRONTMATTER" | grep '^field:' | sed 's/field: *//' | sed 's/^"\(.*\)"$/\1/')
```

对应章节：`parsing-techniques.md:26-99`。

与脚本实现的一致性：

1. `scripts/parse-frontmatter.sh:37-52` 直接使用相同套路。
2. `examples/read-settings-hook.sh:16-23` 也是同样提取链。
3. `ralph-wiggum/hooks/stop-hook.sh:20-25` 在真实 Stop hook 中复用同模式。

### 关键流程 B：body 读取与协议输出

主命令：

```bash
BODY=$(awk '/^---$/{i++; next} i>=2' "$FILE")
```

关键点：`i>=2` 允许正文出现额外 `---`（`parsing-techniques.md:119,336`）。

文档还给出安全 JSON 输出：

```bash
jq -n --arg prompt "$PROMPT" '{"decision":"block","reason":$prompt}'
```

对应位置：`parsing-techniques.md:123-141`。

与真实实现的一致性：`plugins/ralph-wiggum/hooks/stop-hook.sh:165-174` 使用 `jq -n --arg` 构造 Hook 决策 JSON。

### 关键流程 C：状态更新

推荐模式：

```bash
TEMP_FILE="${FILE}.tmp.$$"
sed "s/^field: .*/field: $NEW_VALUE/" "$FILE" > "$TEMP_FILE"
mv "$TEMP_FILE" "$FILE"
```

覆盖单字段和多字段更新（`parsing-techniques.md:211-236`），并在真实 ralph hook 中落地（`plugins/ralph-wiggum/hooks/stop-hook.sh:152-156`）。

### 关键流程 D：校验与回退

文档给出：

1. 文件存在/可读检查（`parsing-techniques.md:240-254`）。
2. marker 计数检查（`parsing-techniques.md:256-266`）。
3. 枚举值与数值范围校验（`parsing-techniques.md:268-299`）。
4. 默认值兜底与错误恢复（`parsing-techniques.md:486-549`）。

对应的可执行工具是：

- `plugins/plugin-dev/skills/plugin-settings/scripts/validate-settings.sh:26-101`

### 实测行为（本次研究）

针对脚本进行临时文件复现：

1. `parse-frontmatter.sh` 在正文存在第三个 `---` 时，会把“第二段 marker 之后再次落入范围”的文本混入 frontmatter。
2. `validate-settings.sh` 对 0 marker 文件会出现 `integer expression expected`（`MARKER_COUNT` 变成 `0\n0`）。

对应实现位置：

- `scripts/parse-frontmatter.sh:37`
- `scripts/validate-settings.sh:41-44`

## 关键代码路径与文件引用

### 直接研究对象

1. `plugins/plugin-dev/skills/plugin-settings/references/parsing-techniques.md`

### 调用方（文档消费者）

1. `plugins/plugin-dev/skills/plugin-settings/SKILL.md:510-516`
2. `plugins/plugin-dev/README.md:127-133`
3. `plugins/plugin-dev/commands/create-plugin.md:157-163,220-226,374-375`
4. `plugins/plugin-dev/skills/skill-development/SKILL.md:311-314`

### 被调用方/实现映射

1. `plugins/plugin-dev/skills/plugin-settings/scripts/parse-frontmatter.sh`
2. `plugins/plugin-dev/skills/plugin-settings/scripts/validate-settings.sh`
3. `plugins/plugin-dev/skills/plugin-settings/examples/read-settings-hook.sh`
4. `plugins/ralph-wiggum/hooks/stop-hook.sh`

### 配置、测试、脚本、文档链路

1. 配置载体：项目根 `.claude/*.local.md`
2. 脚本：`scripts/parse-frontmatter.sh`、`scripts/validate-settings.sh`
3. 测试：当前目录无自动化测试文件，主要依赖手工验证
4. 文档链路：`plugin-dev/README.md` -> `plugin-settings/SKILL.md` -> `references/parsing-techniques.md`

## 依赖与外部交互

### 运行时依赖

1. 强依赖：`bash`、`sed`、`grep`、`awk`
2. 常见增强：`jq`（JSON 安全构造）
3. 可选增强：`yq`（复杂 YAML）

### 与 Claude Code 的交互协议

1. Hook 从 stdin 读取事件 JSON（示例：`examples/read-settings-hook.sh:29-32`，真实：`ralph stop-hook.sh:9-10`）。
2. Hook 通过 stdout/stderr 输出决策 JSON 或错误消息（示例与 ralph 实现均有）。
3. settings 文件的修改通常在下一次运行或重启后稳定生效（`plugin-settings/SKILL.md` 与示例文档均强调 restart 边界）。

### 文件系统交互

1. 读取：`.claude/<plugin>.local.md`
2. 更新：临时文件 + `mv` 原子替换
3. 生命周期：建议 gitignore（`.claude/*.local.md`）

## 风险、边界与改进建议

### 风险 1（高）：frontmatter 提取边界过宽

问题：`sed` 区间写法默认匹配“每一对 marker”，在正文带额外 `---` 时可能混入无关行。实测已复现。

改进：

1. 要求首行必须是 `---`，并仅消费前两个 marker。
2. 采用 `awk` 状态机只抓取第一段 frontmatter。
3. 增加 `--strict` 模式，发现额外 marker 时显式报错。

### 风险 2（高）：marker 计数表达式在 0 匹配时不稳

问题：`MARKER_COUNT=$(grep -c ... || echo "0")` 可能产生 `0\n0`，导致整数比较异常（实测触发）。

改进：

1. 改为：
   - `MARKER_COUNT=$(grep -c '^---$' "$SETTINGS_FILE" 2>/dev/null || true)`
   - `MARKER_COUNT=${MARKER_COUNT:-0}`
2. 或使用 `awk` 单次扫描计数，避免 `grep` 的退出码分支。

### 风险 3（中）：正则式字段读取对复杂 YAML 不可靠

问题：`grep '^key:'` 不支持嵌套对象、多行字符串、缩进列表、带冒号值。

改进：

1. 在文档显式标注“适用范围：扁平键值”。
2. 增加 `yq` 主路径 + shell fallback 的双方案模板。

### 风险 4（中）：字段名直接拼接正则，存在误匹配边界

问题：`grep "^${FIELD}:"` 若字段名含正则元字符，匹配行为不可控。

改进：

1. 对 `FIELD` 做白名单约束（`^[a-zA-Z_][a-zA-Z0-9_]*$`）。
2. 不满足约束立即失败。

### 风险 5（中）：缺少自动化回归

现状：无针对 parser/validator 的 fixture 测试。

改进：

1. 新增 `scripts/tests/`，覆盖：
   - 正常 frontmatter
   - 0 marker
   - body 含 `---`
   - 字段缺失/非法类型
2. 将 smoke test 纳入 plugin-dev 的验证流程。

### 风险 6（低）：原子替换未处理权限与恢复细节

现状：`mv` 后未恢复文件权限，也无异常时临时文件清理。

改进：

1. 使用 `trap 'rm -f "$TEMP_FILE"' EXIT`。
2. 写入后按需 `chmod` 保持权限一致。
