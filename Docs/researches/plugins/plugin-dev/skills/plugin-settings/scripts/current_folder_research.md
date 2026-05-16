# DIR 研究：plugins/plugin-dev/skills/plugin-settings/scripts

## 场景与职责

`plugins/plugin-dev/skills/plugin-settings/scripts` 是 `plugin-settings` 技能的“可执行工具层”，目标是把 `.claude/plugin-name.local.md` 这套约定从文档说明落地为可运行脚本，供插件开发者在本地快速验证与解析。

该目录当前仅包含两个脚本：

1. `parse-frontmatter.sh`：提取 frontmatter 全量内容或指定字段。
2. `validate-settings.sh`：检查 settings 文件结构完整性并输出提示。

在仓库中的职责边界如下：

1. 上游调用方主要是文档与人工流程，而不是 CI/运行时代码：
   - `plugins/plugin-dev/skills/plugin-settings/SKILL.md:525-531`
   - `plugins/plugin-dev/README.md:127-133`
2. 下游被作用对象是项目根目录 `.claude/*.local.md` 配置文件（示例路径：`.claude/my-plugin.local.md`）。
3. 与真实插件实现存在“模式复用”关系：`ralph-wiggum` 的 stop hook 和 setup 脚本使用同类 frontmatter/body 解析与状态更新方法（见 `plugins/ralph-wiggum/hooks/stop-hook.sh:20-26,133-156`，`plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:130-150`）。

结论：该目录是 `plugin-settings` 技能的开发辅助基础设施，不是插件运行时自动调用组件。

## 功能点目的

### 1) `parse-frontmatter.sh`

目的：给 hook/command/agent 的 shell 逻辑提供一个轻量 frontmatter 读取器，减少重复写 `sed|grep|awk`。

使用方式（脚本内置）：

- `bash parse-frontmatter.sh <settings-file.md>`
- `bash parse-frontmatter.sh <settings-file.md> <field>`

预期收益：

1. 快速拿到 frontmatter 片段，便于后续逻辑判断（如 `enabled` 开关）。
2. 抽离字段提取模板，降低不同插件脚本中的重复实现。

### 2) `validate-settings.sh`

目的：在 settings 文件进入“被 hook/command 消费”之前，做最基本结构体检，减少明显格式错误导致的运行期问题。

覆盖点：

1. 文件存在与可读。
2. `---` marker 存在性。
3. frontmatter 非空。
4. 检测 key:value 字段并展示。
5. 对常见布尔字段（`enabled`、`strict_mode`）做弱校验提示。
6. markdown body 是否存在。

预期收益：

1. 开发期快速发现“缺 marker/空 frontmatter/文件路径错误”。
2. 输出人类可读摘要，便于手工调试。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 数据结构与协议

两个脚本都依赖同一协议：Markdown 文件中前置 YAML frontmatter + body。

```markdown
---
enabled: true
mode: standard
---

# Body
...
```

实现层面并未引入 YAML 解析器，而是基于文本规则：

1. frontmatter 提取：`sed -n '/^---$/,/^---$/{ /^---$/d; p; }'`
2. 指定字段提取：`grep '^field:' | sed 's/field: *//'`
3. body 提取：`awk '/^---$/{i++; next} i>=2'`

### B. `parse-frontmatter.sh` 关键流程

代码位置：`plugins/plugin-dev/skills/plugin-settings/scripts/parse-frontmatter.sh:1-59`

流程：

1. 参数检查与 help 输出（`7-25`）。
2. 文件存在性检查（`30-34`）。
3. 以 `sed` 提取 marker 区间（`37`）。
4. 未指定字段时直接输出 frontmatter（`44-48`）。
5. 指定字段时 `grep + sed` 提取并去除引号（`51-52`）。
6. 空值时报错（`53-56`），否则输出（`58`）。

实测行为：

1. 对标准单段 frontmatter 文件可正常提取。
2. 当文件中出现多个 `--- ... ---` 区段时，输出会拼接多个区段内容，而不是只取“文件头 frontmatter”。
3. 当字段不存在时，受 `set -euo pipefail` 影响，`grep` 非 0 会提前退出，无法稳定触发脚本后续自定义报错分支。

### C. `validate-settings.sh` 关键流程

代码位置：`plugins/plugin-dev/skills/plugin-settings/scripts/validate-settings.sh:1-101`

流程：

1. 参数检查（`8-19`）。
2. 文件存在/可读检查（`26-38`）。
3. marker 计数检查（`41-53`）。
4. frontmatter 提取并判空（`55-61`）。
5. 检查是否存在 `:`（`63-66`）。
6. 枚举字段（`68-73`）。
7. 布尔字段弱校验（`75-83`）。
8. 读取 body（`86`）并输出总结（`88-101`）。

实测行为（关键）：

1. 对示例文档 `example-settings.md` 会识别到多个模板代码块中的字段，并最终判定“结构有效”。
2. 当文件无 marker 时，`MARKER_COUNT=$(grep -c ... || echo "0")` 可能得到 `0\n0`，触发 `integer expression expected`，随后仍打印“Frontmatter markers present”再失败退出。
3. 布尔字段检查是 warning，不会导致失败；且同名字段多次出现时提示值可读性较差（多行拼接）。
4. 脚本对 marker 的约束是“至少 2 个”，不是“frontmatter 必须位于文件开头且仅一段”。

### D. 命令与协议示例

本次研究执行过的关键命令：

1. `bash plugins/plugin-dev/skills/plugin-settings/scripts/parse-frontmatter.sh <file> [field]`
2. `bash plugins/plugin-dev/skills/plugin-settings/scripts/validate-settings.sh <file>`
3. 基于 `mktemp` 构造正例/反例文件，验证 marker 缺失、多 marker、字段缺失等边界。

与 hook 协议的结合点（来自近邻示例）：

- `plugins/plugin-dev/skills/plugin-settings/examples/read-settings-hook.sh:29-65` 采用 stdin JSON + `jq` + `exit 2` deny/`exit 0` allow 的事件处理模型，脚本产出的 frontmatter 值直接驱动该逻辑分支。

## 关键代码路径与文件引用

### 目标目录文件

1. `plugins/plugin-dev/skills/plugin-settings/scripts/parse-frontmatter.sh`
2. `plugins/plugin-dev/skills/plugin-settings/scripts/validate-settings.sh`

### 直接上游（调用方/说明方）

1. `plugins/plugin-dev/skills/plugin-settings/SKILL.md:514-531`（将脚本列为 utility scripts）
2. `plugins/plugin-dev/README.md:114-133`（在 plugin-dev 总览中定义 plugin-settings 与脚本资源）
3. `plugins/plugin-dev/commands/create-plugin.md:220-225`（创建插件流程要求接入 settings 模式）

### 直接下游（被调用方/被处理对象）

1. `.claude/plugin-name.local.md`（协议目标）
2. `plugins/plugin-dev/skills/plugin-settings/examples/example-settings.md`（文档模板样本）
3. `plugins/plugin-dev/skills/plugin-settings/examples/read-settings-hook.sh:16-23`（字段提取消费方式）

### 上下文近邻与实战映射

1. `plugins/plugin-dev/skills/plugin-settings/references/parsing-techniques.md:24-141,190-299`（解析/更新/校验套路）
2. `plugins/plugin-dev/skills/plugin-settings/references/real-world-examples.md:5-252`（multi-agent-swarm、ralph-wiggum 案例）
3. `plugins/ralph-wiggum/hooks/stop-hook.sh:20-26,133-156`（仓库内真实 frontmatter/body 读写实现）
4. `plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:130-150`（状态文件创建）

补充：仓库内未发现对 `parse-frontmatter.sh` / `validate-settings.sh` 的自动化直接调用（workflow/脚本/运行时代码）。

## 依赖与外部交互

### 运行依赖

1. 必需：`bash`、`sed`、`grep`、`awk`、`wc`、`tr`
2. 间接依赖（上下文示例脚本）：`jq`（hook 输入解析）
3. 可选依赖（reference 建议）：`yq`（复杂 YAML 场景）

### 文件系统交互

1. 只读目标 settings 文件。
2. 不做网络调用。
3. 不直接写回设置文件（更新逻辑只在 reference 中作为建议示例）。

### 测试与验证现状

1. 目标目录无独立自动化测试（未发现 `test/spec/bats`）。
2. 当前质量保障方式主要是手工执行脚本 + 人工查看输出。

## 风险、边界与改进建议

### 风险与边界

1. frontmatter 边界不严格（高）
   - 当前提取策略会处理文件中多个 marker 区段，可能把 body 中内容混入 frontmatter。
   - 影响：文档型文件或包含额外 `---` 的内容会被误判为“有效配置”。

2. `parse-frontmatter.sh` 字段缺失的错误路径不稳定（高）
   - `set -euo pipefail` 下，`grep` 无匹配直接退出，后续“Field not found”分支无法可靠执行。

3. `validate-settings.sh` marker 计数分支存在实现缺陷（高）
   - 无 marker 场景出现 `integer expression expected`，日志与控制流不一致。

4. 校验强度偏弱（中）
   - 布尔字段非法值只 warning，默认仍成功退出；对重复字段、枚举、数值范围无硬约束。

5. 文档案例与仓库现状存在漂移（中）
   - `real-world-examples.md` 里的 `multi-agent-swarm` 在当前仓库无对应插件目录，容易给读者造成“仓库内可直接追踪”的误解。

### 改进建议

1. frontmatter 识别收敛到“文件首段”
   - 要求首行即 `---`，并只接受第一段 frontmatter；多余 marker 明确报错。

2. 修复字段缺失与 marker 计数的错误处理
   - 对 `grep` 使用 `|| true` 后再统一判空并给出稳定错误信息。
   - 将 marker 计数写成不产生多行结果的安全表达式（避免 `0\n0`）。

3. 增加 `--strict` 模式
   - 严格模式下把布尔值非法、重复关键字段、数值/枚举越界升级为失败退出码。

4. 为脚本补最小回归测试
   - 建议新增 `plugins/plugin-dev/skills/plugin-settings/scripts/tests/`，覆盖：
     - 标准 frontmatter
     - 缺少 marker
     - 多 marker/marker 在 body
     - 字段缺失
     - 重复字段
     - 布尔非法值

5. 文档声明“手工工具”定位
   - 在 `SKILL.md` 或脚本 usage 中标注：这是开发期辅助校验，不能替代严格 YAML 解析与集成测试。
