# plugins/learning-output-style 研究

## 场景与职责
`plugins/learning-output-style` 是一个纯 Hook 型插件目录，核心职责是在 Claude Code 会话启动时（`SessionStart`）注入一段额外系统上下文，把已下线/未发布的 output-style 行为迁移到插件机制中。

它在仓库中的定位是：
- 在插件总览中被定义为“交互式学习输出风格”插件：`plugins/README.md:23`
- 明确声明结合两类能力：
  - Learning：要求用户在关键决策点贡献 5-10 行关键代码
  - Explanatory：在实现前后提供教育性 insight
  见：`plugins/learning-output-style/README.md:3-5,11-23,56-70`

上下文场景（历史演进）：
- 早期版本发布过内建 output styles（含 Learning/Explanatory）：`CHANGELOG.md:1837`
- 后续弃用 output styles 并建议迁移到插件/系统提示：`CHANGELOG.md:1528`
- 最新又明确 output style 固定在会话开始阶段（利于缓存）：`CHANGELOG.md:206`
- `SessionStart` Hook 本身在 1.0.62 引入：`CHANGELOG.md:1908`

## 功能点目的
本目录功能集中在一个目标：通过会话级注入让模型“默认采用互动教学风格”。

拆分为 4 个可见目的：
1. 用插件替代旧 output-style 配置入口
- README 明确“Migration from Output Styles”：`plugins/learning-output-style/README.md:76-82`

2. 让用户参与关键实现决策，而不是被动看 AI 全自动写完
- 何时请求用户贡献：业务逻辑、错误处理、算法/数据结构/架构选择：`plugins/learning-output-style/README.md:28-37`
- 何时不请求：样板代码、明显实现、配置、简单 CRUD：`plugins/learning-output-style/README.md:38-44`

3. 强化可解释输出
- 通过固定 insight 格式输出 2-3 条与代码库相关的实现洞察：`plugins/learning-output-style/README.md:56-64`

4. 通过插件形式可分发、可启停，而非写死在仓库 `CLAUDE.md`
- README 直接说明 SessionStart hook 模式与 `CLAUDE.md` 近似但更易分发：`plugins/learning-output-style/README.md:82`

## 具体技术实现（关键流程/数据结构/协议/命令）
### 1) 目录与配置面
目录仅 4 个文件：
- 元数据：`plugins/learning-output-style/.claude-plugin/plugin.json`
- 说明文档：`plugins/learning-output-style/README.md`
- Hook 配置：`plugins/learning-output-style/hooks/hooks.json`
- Hook 处理器：`plugins/learning-output-style/hooks-handlers/session-start.sh`

`plugin.json` 只包含轻量元信息（name/version/description/author），未定义自定义组件路径：`plugins/learning-output-style/.claude-plugin/plugin.json:1-9`

根据插件结构规范，未覆写时默认从 `./hooks/hooks.json` 加载 Hook：
- `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:259-263`
- 自动发现链路说明：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:343-348`

### 2) 关键调用链（调用方 -> 被调用方）
1. Claude Code 插件加载器读取 `.claude-plugin/plugin.json`（发现插件）
2. Hook 运行时读取 `hooks/hooks.json`（发现 SessionStart 事件绑定）
3. `SessionStart` 触发时执行 command hook：
   - `${CLAUDE_PLUGIN_ROOT}/hooks-handlers/session-start.sh`
   - 配置位置：`plugins/learning-output-style/hooks/hooks.json:4-10`
4. `session-start.sh` 向 stdout 输出结构化 JSON：
   - `hookSpecificOutput.hookEventName = "SessionStart"`
   - `hookSpecificOutput.additionalContext = <长文本提示词>`
   - 输出位置：`plugins/learning-output-style/hooks-handlers/session-start.sh:8-10`
5. Claude Code 将该 `additionalContext` 注入会话上下文，影响后续回答风格

### 3) 核心数据结构与协议
#### Hook 配置结构（插件包装格式）
`hooks/hooks.json` 使用了如下包装：
- 顶层 `description`
- 顶层 `hooks` 对象
- 事件键 `SessionStart` 对应数组
- 子项中 `hooks[0].type = command`
- `hooks[0].command = ${CLAUDE_PLUGIN_ROOT}/hooks-handlers/session-start.sh`

见：`plugins/learning-output-style/hooks/hooks.json:1-15`

#### Hook 输出结构
`session-start.sh` 输出 JSON：
```json
{
  "hookSpecificOutput": {
    "hookEventName": "SessionStart",
    "additionalContext": "..."
  }
}
```
见：`plugins/learning-output-style/hooks-handlers/session-start.sh:7-12`

关键字段语义：
- `hookEventName`：声明当前输出属于哪个 hook 事件
- `additionalContext`：追加到模型上下文的提示文本（本插件的核心行为载体）

### 4) 关键命令与实测结果
本次对目标目录进行了可执行验证：

1. 生成 SessionStart 样例输入
```bash
bash plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh --create-sample SessionStart > /tmp/learning-sessionstart-input.json
jq -r '.hook_event_name' /tmp/learning-sessionstart-input.json
```
结果：输出 `SessionStart`。

2. 执行 hook 处理器
```bash
bash plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh \
  plugins/learning-output-style/hooks-handlers/session-start.sh \
  /tmp/learning-sessionstart-input.json
```
结果：`Exit Code: 0`，stdout 可被 `jq` 解析，包含 `hookSpecificOutput.additionalContext`。

3. 执行 schema 校验脚本（观察兼容性）
```bash
bash plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh \
  plugins/learning-output-style/hooks/hooks.json
```
结果：
- 脚本把顶层 `description/hooks` 识别为“未知事件”告警
- 随后在逐事件校验阶段 `jq` 报错 `Cannot index string with number`
- 退出码为 `5`

该结果说明通用校验脚本假定“事件在顶层”，与当前插件的包装格式（`{"hooks": {...}}`）不兼容。

## 关键代码路径与文件引用
核心实现路径（最短执行链）：
1. `plugins/learning-output-style/.claude-plugin/plugin.json:1-9`
2. `plugins/learning-output-style/hooks/hooks.json:1-15`
3. `plugins/learning-output-style/hooks-handlers/session-start.sh:1-15`

仓库级入口与规范：
- `plugins/README.md:23,49-61`（插件声明与标准目录结构）
- `plugins/plugin-dev/skills/plugin-structure/SKILL.md:343-348`（自动发现顺序）
- `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:259-263`（默认 hooks 路径）

测试与工具链参考：
- `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh:25-100,170-252`
- `plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh:41-75`

同类/对照实现：
- `plugins/explanatory-output-style/hooks/hooks.json:1-15`
- `plugins/explanatory-output-style/hooks-handlers/session-start.sh:1-15`
- `plugins/explanatory-output-style/README.md:3-4,20-28`

历史与行为边界依据：
- `CHANGELOG.md:1837,1528,206,1908,192,597,1079`

## 依赖与外部交互
### 运行时依赖
- Claude Code 插件系统（manifest + hooks 自动发现）
- Hook 事件系统（`SessionStart`）
- shell 执行环境（`#!/usr/bin/env bash`）
- 环境变量替换（`${CLAUDE_PLUGIN_ROOT}`）

### 开发/验证依赖
- `jq`、`timeout`（由 `test-hook.sh` 与 `validate-hook-schema.sh` 使用）
- `bash`（所有测试命令与 handler 均基于 bash）

### 外部交互特征
- 本插件运行时不访问网络、不读写仓库业务文件
- 主要外部交互是“向 Claude 上下文注入额外提示文本”
- README 引用了外部文档链接（plugins/subagents docs），但仅文档层，不影响运行时

## 风险、边界与改进建议
### 风险与边界
1. Token 成本与交互开销固定增加
- README 已明确 warning：会增加 token 成本且交互更重：`plugins/learning-output-style/README.md:7`
- `additionalContext` 文本较长，会在每次会话启动注入。

2. 会话内动态切换能力有限
- output style 机制已固定在 SessionStart（缓存优化背景）：`CHANGELOG.md:206`
- 含义是中途切换风格不如命令级机制灵活。

3. 与 explanatory 插件存在文案漂移风险
- learning 插件内嵌 explanatory 能力，两个插件各自维护相似 prompt 文案：
  - `plugins/learning-output-style/hooks-handlers/session-start.sh:10`
  - `plugins/explanatory-output-style/hooks-handlers/session-start.sh:10`
- 后续更新若不同步，行为会逐步分叉。

4. 缺乏目录内自动化测试资产
- 目标目录未包含 test/CI 脚本；目前依赖 plugin-dev 通用工具手工验证。

5. 通用 hook schema 校验脚本与当前 hooks 包装格式不兼容
- 会导致误报或异常退出（实测退出码 5）。

### 改进建议
1. 增加最小 smoke test（建议放入插件目录）
- 校验 `session-start.sh` 输出 JSON 可解析、`hookEventName == SessionStart`、`additionalContext` 非空。

2. 统一/复用 explanatory 片段模板
- 将 insight 相关文案提取到共享模板（或脚本拼接片段），降低双插件维护漂移。

3. 给 `hooks/hooks.json` 增加可机读注释文档（或 README 附录）
- 明确当前使用“插件包装格式”，避免开发者直接套用不兼容校验脚本。

4. 增加“轻量模式”说明
- 可提供短版 `additionalContext`（低 token 成本版本）供用户复制定制。

5. 在 plugin-dev 的 `validate-hook-schema.sh` 中兼容 `{"hooks": {...}}` 包装
- 先自动下钻到 `.hooks` 再校验事件键，可直接覆盖本插件与 explanatory 插件场景。
