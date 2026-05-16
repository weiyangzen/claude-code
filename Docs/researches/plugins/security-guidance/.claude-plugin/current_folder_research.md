# plugins/security-guidance/.claude-plugin 研究

## 场景与职责

`plugins/security-guidance/.claude-plugin` 是该插件的 manifest 目录，当前仅包含 `plugin.json`。它不承载运行时代码，核心职责是向 Claude Code 插件装载器与市场索引提供插件身份元数据。

- 上游调用方：
  - Claude Code 插件发现/加载流程（按约定读取 `plugin-root/.claude-plugin/plugin.json`）
  - 仓库级 marketplace 索引（`.claude-plugin/marketplace.json`）在打包/分发阶段引用 `./plugins/security-guidance`
- 下游被调用方：
  - 同插件根目录下的自动发现组件（本插件主要是 `hooks/hooks.json` 与 `hooks/security_reminder_hook.py`）
- 与上下文的关系：
  - 本目录负责“声明是谁（metadata）”
  - `hooks/` 目录负责“做什么（安全提醒逻辑）”

## 功能点目的

1. 插件身份声明
- 通过 `name` 唯一标识插件：`security-guidance`。

2. 版本与归属信息声明
- 通过 `version`、`author` 支撑管理、发布与追踪。

3. 功能定位声明
- 通过 `description` 向使用方说明插件价值：编辑阶段安全提醒（命令注入、XSS、危险模式）。

4. 与仓库分发索引保持一致
- `plugin.json` 的核心字段与 `.claude-plugin/marketplace.json` 中 `security-guidance` 条目保持一致，便于仓库内分发与目录索引。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 数据结构：manifest JSON

目录内唯一文件为标准 JSON：

- `name: string`
- `version: string`（语义化版本）
- `description: string`
- `author: { name: string, email: string }`

该 manifest 采用“最小元数据集”，未声明自定义路径字段（如 `hooks`、`commands`），因此依赖默认目录约定自动发现组件。

### 2) 关键流程 A：插件发现与装载

1. 运行时扫描插件目录。
2. 读取 `.claude-plugin/plugin.json` 识别插件元数据。
3. 按默认约定在插件根目录发现组件（本插件命中 `hooks/hooks.json`）。
4. 进入 hook 注册链路：`PreToolUse` -> `python3 ${CLAUDE_PLUGIN_ROOT}/hooks/security_reminder_hook.py`。

结论：`.claude-plugin` 本身不执行逻辑，但决定插件是否能被识别并纳入后续 hook 执行链。

### 3) 关键流程 B：仓库级分发索引对齐

- `.claude-plugin/marketplace.json` 为 `security-guidance` 维护同名条目，包含 `source: ./plugins/security-guidance` 与 `category: security`。
- 对比结果显示：`name/version/description/author` 与本目录 `plugin.json` 一致。

### 4) 协议与命令（研究/验证）

可复用的本地验证命令：

```bash
# 读取 manifest
jq . plugins/security-guidance/.claude-plugin/plugin.json

# 校验 marketplace 对齐
jq '.plugins[] | select(.name=="security-guidance") | {name,version,description,author,source,category}' .claude-plugin/marketplace.json

# 查看同插件运行配置（由 manifest 间接关联）
jq . plugins/security-guidance/hooks/hooks.json
```

可复用的通用 hook 工具链（来自 `plugin-dev`）：

```bash
plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh plugins/security-guidance/hooks/hooks.json
plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh --create-sample PreToolUse
plugins/plugin-dev/skills/hook-development/scripts/hook-linter.sh plugins/security-guidance/hooks/security_reminder_hook.py
```

注：第三个脚本主要面向 shell hook，直接用于 Python 脚本时参考价值有限。

## 关键代码路径与文件引用

目标目录：

- `plugins/security-guidance/.claude-plugin/plugin.json`
  - 本目录唯一实现对象；定义插件元数据。

直接上下文（调用方/被调用方/配置/文档/脚本）：

- `.claude-plugin/marketplace.json`
  - 仓库分发索引；`security-guidance` 条目引用 `./plugins/security-guidance`。
- `plugins/security-guidance/hooks/hooks.json`
  - 运行期 hook 注册入口（`PreToolUse` + `Edit|Write|MultiEdit` matcher）。
- `plugins/security-guidance/hooks/security_reminder_hook.py`
  - 被 `hooks.json` 调起的实际安全提醒与阻断实现。
- `plugins/README.md`
  - 插件目录总览，定义该插件为 PreToolUse 安全提醒插件。
- `plugins/plugin-dev/skills/plugin-structure/SKILL.md`
  - 说明 `.claude-plugin/plugin.json` 的规范职责与默认目录约定。
- `plugins/plugin-dev/skills/hook-development/SKILL.md`
  - 说明 plugin hook wrapper 格式、`${CLAUDE_PLUGIN_ROOT}` 与 PreToolUse 协议。
- `plugins/plugin-dev/skills/hook-development/scripts/*`
  - 提供 hooks 配置校验、样例输入生成、脚本测试流程。

测试现状：

- `plugins/security-guidance/.claude-plugin` 目录内无独立测试。
- 仓库内未见专门针对 `plugin.json` 的 CI 断言脚本（当前以约定和人工维护为主）。

## 依赖与外部交互

1. 运行时依赖
- 本目录自身无语言运行时依赖（纯 JSON 元数据）。

2. 内部交互依赖
- 依赖 Claude Code 插件装载器按规范读取 manifest。
- 依赖插件根目录默认自动发现策略，将控制权继续传递到 `hooks/`。

3. 外部交互
- 无直接网络/系统调用。
- 间接参与 marketplace 分发元数据编排（由仓库根 `.claude-plugin/marketplace.json` 完成）。

## 风险、边界与改进建议

1. 元数据双写漂移风险
- 风险：`plugin.json` 与 `.claude-plugin/marketplace.json` 存在重复字段，后续可能出现版本/描述不一致。
- 建议：增加一致性检查脚本（CI 中比对 `name/version/description/author`）。

2. manifest 信息维度偏少
- 风险：缺少 `homepage/repository/license/keywords`，影响可发现性与治理信息完整性。
- 建议：按插件治理需要补充可选元数据字段。

3. 职责边界容易被误读
- 边界：`.claude-plugin` 只负责声明，不包含策略实现；安全能力实际在 `hooks/`。
- 建议：在插件 README 中显式标注“manifest 与运行逻辑分层”。

4. 缺少面向 manifest 的自动校验资产
- 风险：未来字段变更可能在运行时才暴露问题。
- 建议：在仓库脚本中增加 manifest schema 校验或统一 `claude plugin validate` 流程。

5. 子目录文档缺失
- 风险：维护者只看 `.claude-plugin` 容易不清楚真实执行链路。
- 建议：在本研究文档或插件 README 增加“元数据 -> hooks -> 脚本”调用图。

