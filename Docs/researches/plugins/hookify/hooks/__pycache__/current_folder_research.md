# DIR `plugins/hookify/hooks/__pycache__` 研究文档

## 场景与职责

`plugins/hookify/hooks/__pycache__` 是 Hookify 四个事件执行器的 CPython 字节码缓存目录，当前包含：

- `posttooluse.cpython-312.pyc`
- `pretooluse.cpython-312.pyc`
- `stop.cpython-312.pyc`
- `userpromptsubmit.cpython-312.pyc`

该目录不承载“规则业务真源”，其职责是把 `hooks/*.py` 的源码编译产物缓存到本地，减少 Hook 触发时的导入编译开销。由于 `plugins/hookify/hooks/hooks.json` 为四类事件都配置了 `timeout: 10`，冷启动与导入性能直接影响可用预算。

上下文定位：

- 上游调用方：Claude Code hook runtime（按 `hooks.json` 调起命令）。
- 同层源码：`plugins/hookify/hooks/pretooluse.py`、`posttooluse.py`、`stop.py`、`userpromptsubmit.py`。
- 下游被调用方：`hookify.core.config_loader.load_rules` 与 `hookify.core.rule_engine.RuleEngine`。
- 规则来源：项目 `.claude/hookify.*.local.md`。

## 功能点目的

1. 导入加速与超时预算保护
- 通过 `.pyc` 复用解释器编译结果，降低每次 hook 进程启动时的解析开销。
- 对 10 秒 timeout 的命令 hook，缓存命中可降低“规则逻辑以外”的固定损耗。

2. 解释器版本隔离
- 文件名后缀 `cpython-312` 绑定 CPython 3.12 字节码 ABI，避免不同主版本误加载。

3. 源码一致性校验锚点
- `.pyc` 头中记录 `timestamp/source_size`，可与源码大小、修改时间交叉验证缓存是否过期。

4. 运行行为快照（非权威）
- `__pycache__` 能反映某一时刻 `hooks/*.py` 的可执行形态，但不应替代源码评审与审计。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 关键流程：事件触发 -> Python 命令 -> 字节码命中

1. Claude Code 触发 `PreToolUse/PostToolUse/Stop/UserPromptSubmit`。
2. `plugins/hookify/hooks/hooks.json` 执行 `python3 ${CLAUDE_PLUGIN_ROOT}/hooks/*.py`。
3. 脚本读取 stdin JSON，并依据事件加载规则：
   - Pre/Post：按 `tool_name` 折叠成 `bash` 或 `file`。
   - Stop：固定 `event='stop'`。
   - UserPromptSubmit：固定 `event='prompt'`。
4. 脚本导入 `hookify.core.*` 时，解释器优先检查 `__pycache__` 下对应 `.pyc`。
5. 缓存可用则直接执行字节码；不可用则回退源码编译并刷新缓存。
6. `RuleEngine.evaluate_rules(...)` 生成 JSON 决策并输出到 stdout，脚本统一 `exit 0`（故障默认放行）。

### 2) 缓存文件头与源码映射（实测）

对 `plugins/hookify/hooks/__pycache__/*.pyc` 解析得到：

- `magic=cb0d0d0a`（CPython 3.12）。
- `bitfield=0`（timestamp-based pyc）。
- `src_ts=1773905087`（`2026-03-19 07:24:47 UTC`）。
- `src_size` 分别为 `1776/2205/1557/1543`，与对应源码完全一致：
  - `posttooluse.py` 1776
  - `pretooluse.py` 2205
  - `stop.py` 1557
  - `userpromptsubmit.py` 1543

这证明当前缓存与仓库中的 hook 源码一一对应，非明显脏/陈旧产物。

### 3) 代码对象符号与入口一致性（实测）

对 `.pyc` 做 `marshal` 反序列化后，四个模块都包含 `main` 入口，且 `co_filename` 指向源码路径：

- `posttooluse.cpython-312.pyc` -> `plugins/hookify/hooks/posttooluse.py`，`main` 起始第 30 行
- `pretooluse.cpython-312.pyc` -> `plugins/hookify/hooks/pretooluse.py`，`main` 起始第 35 行
- `stop.cpython-312.pyc` -> `plugins/hookify/hooks/stop.py`，`main` 起始第 30 行
- `userpromptsubmit.cpython-312.pyc` -> `plugins/hookify/hooks/userpromptsubmit.py`，`main` 起始第 30 行

说明缓存内容与实际可执行入口匹配，不存在“模块名对上但逻辑偏移”的迹象。

### 4) 与 Hook 协议的关系

`__pycache__` 本身不定义协议，但缓存的是协议实现代码：

- Block（Pre/Post）走 `hookSpecificOutput.permissionDecision = "deny"`
- Block（Stop）走 `decision = "block" + reason`
- Warn 或无命中返回 `systemMessage` 或 `{}`

若缓存损坏/解释器版本不匹配，最终表现会落在“导入失败后输出系统错误并放行”路径（脚本 `except` + `exit 0`）。

### 5) 研究中使用的关键命令

- `file plugins/hookify/hooks/__pycache__/*`
- `stat -c '%n | size=%s | mtime=%y' plugins/hookify/hooks/*.py plugins/hookify/hooks/__pycache__/*.pyc`
- `python3` + `struct/marshal` 解析 `.pyc` 头与代码对象
- `rg -n "pretooluse.py|posttooluse.py|stop.py|userpromptsubmit.py|load_rules\(|RuleEngine\(" plugins/hookify`
- `printf '<json>' | CLAUDE_PLUGIN_ROOT=plugins/hookify python3 plugins/hookify/hooks/pretooluse.py`（以及其余 3 个 hook）
- `bash plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh ...`（通用 hook 测试脚本联调）

## 关键代码路径与文件引用

- `plugins/hookify/hooks/__pycache__/pretooluse.cpython-312.pyc`  
  对应 `pretooluse.py` 的 CPython 3.12 字节码缓存。

- `plugins/hookify/hooks/__pycache__/posttooluse.cpython-312.pyc`  
  对应 `posttooluse.py` 的 CPython 3.12 字节码缓存。

- `plugins/hookify/hooks/__pycache__/stop.cpython-312.pyc`  
  对应 `stop.py` 的 CPython 3.12 字节码缓存。

- `plugins/hookify/hooks/__pycache__/userpromptsubmit.cpython-312.pyc`  
  对应 `userpromptsubmit.py` 的 CPython 3.12 字节码缓存。

- `plugins/hookify/hooks/hooks.json:4-47`  
  四类事件与 Python 命令入口映射，均设置 `timeout: 10`。

- `plugins/hookify/hooks/pretooluse.py:14-23,35-70`  
  路径注入、stdin 读取、事件映射、规则评估与统一 `exit 0`。

- `plugins/hookify/hooks/posttooluse.py:13-19,30-62`  
  PostToolUse 入口逻辑，与 PreToolUse 基本同构。

- `plugins/hookify/hooks/stop.py:13-19,30-55`  
  固定 `event='stop'` 规则评估。

- `plugins/hookify/hooks/userpromptsubmit.py:13-19,30-54`  
  固定 `event='prompt'` 规则评估。

- `plugins/hookify/core/config_loader.py:198-241`  
  规则发现与事件过滤（`.claude/hookify.*.local.md`）。

- `plugins/hookify/core/rule_engine.py:35-94,182-254`  
  决策聚合与字段提取，决定最终 hook 协议输出。

- `plugins/hookify/.gitignore:1-3`  
  明确将 `__pycache__/` 与 `*.py[cod]` 视为应忽略的运行时产物。

- `plugins/hookify/README.md:122-128,298-301`  
  对 event 语义与 Python 运行时要求的文档约束。

- `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh`  
  提供 hook 输入样例生成与执行验证。

- `plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh`  
  提供通用 `hooks.json` 校验，但与 hookify 的 wrapper 结构并不完全兼容。

## 依赖与外部交互

1. Python 解释器与导入缓存机制
- 依赖 CPython 的 `__pycache__` 与 `.pyc` 头校验行为。
- 当前缓存命名显示运行/编译环境为 3.12。

2. Claude Hook Runtime 协议
- 输入：stdin JSON（`hook_event_name/tool_name/tool_input/user_prompt/transcript_path` 等）。
- 输出：stdout JSON（`systemMessage`、`hookSpecificOutput`、`decision/reason`）。

3. 环境变量
- `CLAUDE_PLUGIN_ROOT` 决定导入路径注入；设置错误会走 import error 放行路径。
- `test-hook.sh` 还会使用 `CLAUDE_PROJECT_DIR`、`CLAUDE_ENV_FILE` 组织测试环境。

4. 文件系统
- 规则文件读取：`.claude/hookify.*.local.md`（相对当前工作目录）。
- Stop 场景可能读取 `transcript_path` 指向文件。
- `__pycache__` 是导入副产物，生命周期受解释器和文件变更驱动。

5. 文档与命令层
- `/hookify`、`/hookify:list`、`/hookify:configure` 负责生产/维护规则文件，间接决定 hook 运行输入质量。
- README/skill 文档决定规则格式约束与运维预期（如“无需重启立即生效”）。

## 风险、边界与改进建议

1. 字节码版本边界
- 风险：`cpython-312` 缓存不能跨 3.11/3.13 复用，跨版本环境会重编译或出现行为偏差。
- 建议：在排障脚本与 README 中显式输出 `python3 --version` 与缓存版本信息。

2. 缓存被纳入仓库的稳定性风险
- 现状：`plugins/hookify/.gitignore` 忽略 `__pycache__`，但目录中仍存在已跟踪 `.pyc`。
- 风险：不同机器/路径生成的缓存进入版本库，可能制造无意义 diff 与噪声。
- 建议：将 `.pyc` 明确视作构建产物，只保留源码；若需调试证据，放入研究文档而非源码树。

3. Fail-open 语义的安全边界
- 现状：四个 hook 执行器在异常场景统一 `exit 0`。
- 风险：当 import 或解析失败时，规则体系失效但主流程继续，安全型规则可能被绕过。
- 建议：增加可选“严格模式”（关键事件支持 fail-closed），默认仍保持可用性优先。

4. 测试工具与 hookify 配置结构不一致
- 现状：`validate-hook-schema.sh` 预期顶层是事件键；hookify 使用 `{"description","hooks":{...}}` wrapper，直接校验会报错。
- 风险：研发流程误判配置非法，或团队放弃自动校验。
- 建议：扩展校验器兼容 wrapper 格式，或在 hookify 文档显式说明需先取 `.hooks` 节点再校验。

5. 规则路径的工作目录边界
- 现状：规则扫描路径固定为相对 `.claude/hookify.*.local.md`。
- 风险：若运行 cwd 与项目根不一致，会出现“规则存在但不生效”。
- 建议：优先使用 `CLAUDE_PROJECT_DIR`（若可用）作为规则根目录，并在日志中打印最终扫描路径。

6. `__pycache__` 的审计边界
- 风险：`.pyc` 仅能说明某次编译状态，不是长期可追溯的业务真源。
- 建议：保持“源码主审计、字节码辅证据”的治理策略；研究类文档可保留 `.pyc` 校验结果以辅助溯源。
