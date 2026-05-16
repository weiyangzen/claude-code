# plugins/plugin-dev/skills/plugin-settings/references 研究

## 场景与职责

`plugins/plugin-dev/skills/plugin-settings/references` 是 `plugin-settings` 技能的“深度知识层”，承接 `SKILL.md` 的核心规范，把细节从主文档剥离到按需加载的专题文档。

该目录当前由两份文档组成：

1. `parsing-techniques.md`：偏“方法论 + 命令级实现”，覆盖 frontmatter/body 解析、字段提取、原子更新、验证、调试与 `yq` 替代路线（`plugins/plugin-dev/skills/plugin-settings/references/parsing-techniques.md:24-549`）。
2. `real-world-examples.md`：偏“模式归纳 + 案例映射”，把 `.claude/*.local.md` 模式映射到多 Agent 协调与循环控制两个场景（`plugins/plugin-dev/skills/plugin-settings/references/real-world-examples.md:5-395`）。

在插件体系中的职责边界：

1. 不直接参与运行时加载注册（不被 hook 引擎直接执行）。
2. 作为 `plugin-settings/SKILL.md` 的下钻资源，被技能触发后按需引用（`plugins/plugin-dev/skills/plugin-settings/SKILL.md:510-516`）。
3. 为 `scripts/parse-frontmatter.sh`、`scripts/validate-settings.sh`、`examples/*` 提供设计依据与解释性语义（`plugins/plugin-dev/skills/plugin-settings/SKILL.md:517-531`）。

## 功能点目的

### 1) 将 `.local.md` 协议从“概念”落到“可执行套路”

`parsing-techniques.md` 直接给出 shell 可执行片段，把 `.claude/plugin-name.local.md` 的双区结构（YAML frontmatter + markdown body）拆分为可复用步骤（`parsing-techniques.md:5-23,26-141`）。

### 2) 统一 settings 驱动行为的最小工程模式

两份 reference 共同强调四个核心模式：

1. quick-exit（文件不存在或 disabled 立即退出）
2. enabled 开关（运行时软启停）
3. 原子更新（临时文件 + `mv`）
4. 默认值与容错回退

证据：`parsing-techniques.md:145-236,486-549`、`real-world-examples.md:268-329`。

### 3) 给插件作者“从创建到消费”的闭环视角

`real-world-examples.md` 覆盖创建、读取、更新全过程：

1. 创建：`cat <<EOF` 写入 `.local.md`（`real-world-examples.md:96-116,231-252`）。
2. 读取：`sed/grep/awk` 从 hook 中读取 frontmatter 与 body（`real-world-examples.md:50-86,156-219`）。
3. 更新：`sed` 改字段并原子替换（`real-world-examples.md:120-127,197-207,292-299`）。

### 4) 作为 plugin-dev 技能体系的“教学型 reference”

上游文档把该目录定位成 plugin-settings 的核心资源之一：

1. `plugin-dev/README.md` 显式登记“2 reference docs”（`plugins/plugin-dev/README.md:127-133`）。
2. `create-plugin` Phase 5 要求实现 settings 能力时加载 plugin-settings skill（`plugins/plugin-dev/commands/create-plugin.md:157-163,220-226,374-375`）。
3. `skill-development` 将 plugin-settings 作为“progressive disclosure + real-world examples”范例（`plugins/plugin-dev/skills/skill-development/SKILL.md:311-316`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 数据结构

统一数据结构是 `.claude/<plugin>.local.md`：

1. frontmatter 区：布尔、数字、字符串、列表（`parsing-techniques.md:7-16`）。
2. body 区：自由 markdown 文本，可作为 prompt/说明文（`parsing-techniques.md:18-22,101-141`）。

### B. 关键流程 1：frontmatter 提取与字段读取

主流程：

1. frontmatter 提取：`sed -n '/^---$/,/^---$/{ /^---$/d; p; }'`（`parsing-techniques.md:28-40`）。
2. 字段读取：`grep '^field:' | sed ...`（`parsing-techniques.md:43-85`）。
3. 类型校验：布尔、数值、枚举范围校验（`parsing-techniques.md:52-73,268-299`）。

### C. 关键流程 2：body 提取与协议输出

body 使用 `awk '/^---$/{i++; next} i>=2'`，只把第二个 marker 后内容作为正文（`parsing-techniques.md:103-141`）。

该正文可被拼装为 Hook 决策 JSON；reference 推荐使用 `jq -n --arg` 避免字符串拼接转义问题（`parsing-techniques.md:131-141`）。

### D. 关键流程 3：状态更新

更新策略是“临时文件 + 原子替换”：

1. 单字段更新：`sed ... > temp && mv`（`parsing-techniques.md:211-223`）。
2. 多字段批量更新：`sed -e ... -e ... > temp && mv`（`parsing-techniques.md:224-236`）。

### E. 关键流程 4：验证与调试

1. 文件存在/可读性检查（`parsing-techniques.md:240-254`）。
2. marker 数量检查（`parsing-techniques.md:256-266`）。
3. 调试模式 `set -x`、打印解析值（`parsing-techniques.md:416-451`）。
4. 完整默认值回退与恢复示例（`parsing-techniques.md:486-549`）。

### F. 案例流程映射（real-world-examples）

1. `multi-agent-swarm`：配置用于协调通知，核心字段 `coordinator_session/agent_name/task_number/pr_number/enabled`（`real-world-examples.md:11-86`）。
2. `ralph-wiggum`：配置用于循环控制，核心字段 `iteration/max_iterations/completion_promise`，并把 body 作为下一轮 prompt（`real-world-examples.md:135-219`）。
3. 两者共性被抽象为 quick-exit、enabled、原子更新、容错（`real-world-examples.md:266-329`）。

### G. 与仓库实物的对应关系（实测核验）

1. `ralph-wiggum` 真实实现存在并与 reference 主流程一致：
   - hook 读取与状态推进：`plugins/ralph-wiggum/hooks/stop-hook.sh:13-177`
   - state 文件创建：`plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:130-150`
2. `multi-agent-swarm` 在当前仓库无对应插件目录，仅在文档中出现（`rg` 检索结果仅命中文档）。

### H. 命令级实测结果

在仓库中执行了 reference 对应脚本做行为验证：

1. 对包含第三个 `---`（位于 body）的测试文件执行 `parse-frontmatter.sh`，输出会把第三段 marker 后文本混入 frontmatter。
2. `validate-settings.sh` 在“无 marker 文件”场景会触发 `integer expression expected`，随后仍打印“Frontmatter markers present”，再因 frontmatter 为空失败退出。

关联代码位置：

1. frontmatter 提取范围过宽：`plugins/plugin-dev/skills/plugin-settings/scripts/parse-frontmatter.sh:37`
2. marker 计数表达式脆弱：`plugins/plugin-dev/skills/plugin-settings/scripts/validate-settings.sh:41-45`

## 关键代码路径与文件引用

### 目标目录（本次研究对象）

1. `plugins/plugin-dev/skills/plugin-settings/references/parsing-techniques.md`
2. `plugins/plugin-dev/skills/plugin-settings/references/real-world-examples.md`

### 调用方（谁消费该目录）

1. `plugins/plugin-dev/skills/plugin-settings/SKILL.md:510-516`
2. `plugins/plugin-dev/README.md:127-133`
3. `plugins/plugin-dev/commands/create-plugin.md:157-163,220-226,374-375`
4. `plugins/plugin-dev/skills/skill-development/SKILL.md:311-316`

### 被调用方/近邻依赖（该目录落地到哪里）

1. `plugins/plugin-dev/skills/plugin-settings/scripts/parse-frontmatter.sh`
2. `plugins/plugin-dev/skills/plugin-settings/scripts/validate-settings.sh`
3. `plugins/plugin-dev/skills/plugin-settings/examples/read-settings-hook.sh`
4. `plugins/plugin-dev/skills/plugin-settings/examples/create-settings-command.md`
5. `plugins/ralph-wiggum/hooks/stop-hook.sh`
6. `plugins/ralph-wiggum/scripts/setup-ralph-loop.sh`
7. `plugins/ralph-wiggum/hooks/hooks.json`

### 配置、测试、脚本、文档链路

1. 配置载体：`.claude/*.local.md`（reference 主协议）。
2. 脚本：`scripts/parse-frontmatter.sh`、`scripts/validate-settings.sh`。
3. 测试现状：本目录无自动化测试文件，主要依赖示例与手工验证。
4. 文档链路：`README` -> `SKILL.md` -> `references/*.md` -> `examples/*` -> `scripts/*`。

## 依赖与外部交互

### Shell 与 CLI 依赖

1. 强依赖：`bash`、`sed`、`grep`、`awk`。
2. 高频可选：`jq`（JSON 组装/解析，`parsing-techniques.md:96-99,128-141,469-471`）。
3. 可选强解析：`yq`（复杂 YAML，`parsing-techniques.md:87-99,453-485`）。
4. 场景依赖：`tmux`（swarm 通知示例，`real-world-examples.md:79-83`）、`perl`（ralph promise 提取示例，`real-world-examples.md:188`）。

### 与 Claude Code 运行时交互

1. Hook 通过 stdin 读事件 JSON、输出决策 JSON（在 examples 和 ralph 实现中可见）。
2. settings 修改存在“需重启生效”的运行时边界（`plugins/plugin-dev/skills/plugin-settings/SKILL.md:367-383`，`plugins/plugin-dev/skills/plugin-settings/examples/example-settings.md:154-159`）。

### 文件系统交互

1. 读取路径是项目根 `.claude/*.local.md`。
2. 更新采用原子替换避免损坏（reference 推荐模式）。
3. 生命周期建议为用户本地态并加入 `.gitignore`（`plugins/plugin-dev/skills/plugin-settings/SKILL.md:327-336`）。

## 风险、边界与改进建议

### 风险 1（高）：frontmatter 提取边界不严格

现状：`sed` 范围模式在文件含多个 `---` 时会把中间段内容混入结果。该行为在实测中可复现。

影响：当正文包含 markdown 分隔线时，字段提取可能污染，进而误判 `enabled` 或其他关键字段。

建议：

1. parser/validator 默认要求首行即 `---`，且只消费前两段 marker。
2. 增加 `--strict`，检测到额外 marker 时直接失败并提示。

### 风险 2（高）：`validate-settings.sh` 的 marker 计数存在分支缺陷

现状：`MARKER_COUNT=$(grep -c '^---$' "$SETTINGS_FILE" 2>/dev/null || echo "0")` 在“无匹配”时可能形成 `0\n0`，触发 `integer expression expected`（`plugins/plugin-dev/skills/plugin-settings/scripts/validate-settings.sh:41-45`）。

影响：校验提示与真实状态不一致，降低脚本可信度。

建议：

1. 改为 `MARKER_COUNT=$(grep -c '^---$' "$SETTINGS_FILE" 2>/dev/null || true); MARKER_COUNT=${MARKER_COUNT:-0}`。
2. 增加单测覆盖“0 marker”场景并断言错误文案。

### 风险 3（中）：reference 的 YAML 解析模型能力边界较窄

现状：主路径依赖正则/行文本匹配，不覆盖嵌套对象、多行字符串、复杂数组。

影响：用户按文档复制后，遇到复杂 YAML 可能出现“解析成功但语义错误”。

建议：

1. 文档中明确“简单字段/复杂字段”分界。
2. 提供 `yq` 优先、shell fallback 的双通道示例模板。

### 风险 4（中）：real-world 案例的可追溯性不一致

现状：`ralph-wiggum` 可在仓库核验；`multi-agent-swarm` 仅文档存在，无本仓库实现。

影响：读者可能误判其为当前仓库可直接复用资产。

建议：

1. 在 `real-world-examples.md` 标注 `multi-agent-swarm` 为 external example。
2. 若保留“real-world”定位，补充可访问的来源链接或替换为仓库内案例。

### 风险 5（中）：缺少 references 层面的自动化回归

现状：无针对 `references` 建议命令的测试数据集和回归脚本。

影响：文档、示例、脚本随时间漂移时不易被及时发现。

建议：

1. 新增 `plugins/plugin-dev/skills/plugin-settings/scripts/tests/`。
2. 覆盖场景：正常 frontmatter、正文含 `---`、无 marker、重复字段、非法布尔与范围值。
3. 在 `plugin-dev` 验证流程中加入 settings parser smoke test。
