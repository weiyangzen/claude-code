# marketplace-considerations.md 研究

## 场景与职责

`plugins/plugin-dev/skills/command-development/references/marketplace-considerations.md` 是 command-development references 里“分发治理层”文档，目标是让命令从“本地可用”升级为“面向未知用户可发布、可维护”。

核心职责：

1. 分发前兼容性设计（跨平台、依赖检测、降级策略）。
2. 面向陌生用户的 UX（首跑引导、容错输入、诊断信息）。
3. 命名冲突与配置化策略。
4. 版本兼容、折旧、更新通知与 beta 策略。
5. 发布前 checklist 与质量标准。

## 功能点目的

1. Universal Compatibility（`11-64`）
- 避免平台绑定命令，减少 marketplace 失败安装后的使用门槛。

2. Minimal Dependencies + Graceful Degradation（`66-163`）
- 明确 required/optional 依赖并在缺失时可降级。

3. Unknown User UX（`165-292`）
- 提供首次使用 onboarding、纠错建议、可提交支持诊断信息。

4. Distribution 规范（`294-479`）
- 命名去冲突、`.local.md` 配置化、版本兼容检查与弃用引导。

5. 发布运营（`481-904`）
- 发现性文案、演示样例、反馈机制、预发布 checklist、beta 与升级策略。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 跨平台与依赖检测流程

1. `uname` 分支识别平台（`25-43`）。
2. `command -v` 扫描依赖并聚合缺失项（`85-104`）。
3. feature detection 后按能力降级（`141-160`）。

### 2) 用户配置协议

文档推荐读取 `.claude/plugin-name.local.md`（`362-374`），用 frontmatter 存储偏好（verbose/color/max_results）。

这与 plugin-settings 技能一致：

1. `plugins/plugin-dev/skills/plugin-settings/SKILL.md:11-19`
2. `plugins/plugin-dev/skills/plugin-settings/scripts/parse-frontmatter.sh:37-58`

### 3) 版本与更新协议

1. 版本兼容检查模板（`430-449`）。
2. 弃用警告模板（`459-479`）。
3. 更新提示模板（`853-873`），通过 `/plugin update plugin-name` 引导升级。

### 4) 可靠性协议

1. 幂等执行：flag 文件防重复（`666-690`）。
2. 原子操作：临时目录 + 校验通过后一次性移动（`699-723`）。

### 5) 发布门禁协议

`Pre-Release Checklist`（`731-769`）覆盖功能、体验、分发、质量、支持五类门禁，适合作为发布前统一验收模板。

## 关键代码路径与文件引用

核心文件：

1. `plugins/plugin-dev/skills/command-development/references/marketplace-considerations.md:1-904`

关键调用方：

1. `plugins/plugin-dev/skills/command-development/README.md:106`
2. `Docs/researches/plugins/plugin-dev/skills/command-development/current_folder_research.md:64,196`
3. `Docs/researches/plugins/plugin-dev/skills/command-development/references/current_folder_research.md:75-79,112-113`

关键依赖文件：

1. `plugins/plugin-dev/skills/plugin-settings/SKILL.md`
2. `plugins/plugin-dev/skills/plugin-settings/scripts/parse-frontmatter.sh`
3. `plugins/plugin-dev/skills/command-development/references/documentation-patterns.md`（发布文档与支持渠道联动）

## 依赖与外部交互

1. 依赖系统命令：`uname`、`command -v`、`grep`、`test`、`mktemp`、`mv`。
2. 依赖插件管理命令：`/plugin update plugin-name`。
3. 依赖用户本地配置文件协议：`.claude/plugin-name.local.md`。
4. 依赖外部支持渠道：Issue 链接、文档站点、release notes。

## 风险、边界与改进建议

### 风险

1. 版本比较示例 `if [ "$PLUGIN_VERSION" < "2.0.0" ]`（`435`）是字符串比较，不能正确处理语义化版本。
2. 配置读取示例用 `grep/cut` 解析 YAML（`366-368`）较脆弱，嵌套结构与空格会误判。
3. 文档中多个 `Bash(*)`（`18,73`）与安全最小权限原则不一致。
4. 模板含固定时间与版本（如 2025 年折旧日期、`2.1.0`），复制后容易过期。

### 边界

1. 本文件给的是“市场化建议模板”，不是强制实现标准。
2. 不包含真实 CI 矩阵或发布流水线脚本。

### 改进建议

1. 提供 semver 比较脚本/函数，替代字符串比较。
2. 复用 `plugin-settings/scripts/parse-frontmatter.sh` 或引入可靠 YAML 解析工具，避免 grep 解析。
3. 将 `Bash(*)` 样例降级为特例，默认改成前缀白名单。
4. 增加真实跨平台测试矩阵示例（Linux/macOS/Windows）并关联 CI。
