# DIR `plugins/hookify/core/__pycache__` 研究文档

## 场景与职责

`plugins/hookify/core/__pycache__` 是 Hookify 核心模块（`core`）的 Python 字节码缓存目录，不承载业务规则定义本身，而是承载运行期导入优化产物。该目录下当前存在 3 个 `.pyc` 文件：

- `__init__.cpython-312.pyc`
- `config_loader.cpython-312.pyc`
- `rule_engine.cpython-312.pyc`

在 Hookify 的执行链路里，4 个 hook 执行器都会在进程启动时导入 `hookify.core.config_loader` 与 `hookify.core.rule_engine`。这些导入由 Python import 机制优先尝试命中 `__pycache__`，从而减少每次 hook 触发时的源码编译成本（特别是 `hooks.json` 中所有事件都设置了 10 秒超时，启动开销会直接影响可用时间窗口）。

## 功能点目的

1. 导入加速
- 通过缓存 `.py` 编译结果，减少 `PreToolUse/PostToolUse/Stop/UserPromptSubmit` 每次执行时的解析与编译开销。

2. 运行时版本绑定
- 文件名中的 `cpython-312` 明确绑定 CPython 3.12 ABI，避免跨主版本误复用缓存。

3. 源码映射与调试线索
- `.pyc` 内部记录 `co_filename`、函数名与首行号，可用于定位其对应源码（`config_loader.py`、`rule_engine.py`）并辅助排障。

4. 非业务真源（non-source-of-truth）
- 该目录只反映某次导入后的缓存快照；真正业务语义仍由 `.py` 源码定义。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 关键流程：从 Hook 触发到 `__pycache__` 命中

1. Claude Code 触发 Hook 事件。
2. `plugins/hookify/hooks/hooks.json` 调用 `python3 ${CLAUDE_PLUGIN_ROOT}/hooks/*.py`。
3. hook 脚本通过 `CLAUDE_PLUGIN_ROOT` 注入 `sys.path`，导入 `hookify.core.config_loader` / `hookify.core.rule_engine`。
4. Python import 机制检查 `plugins/hookify/core/__pycache__/` 中对应 `*.cpython-312.pyc`。
5. 若头信息（magic/timestamp/source_size）可用则直接加载字节码；否则回退源码重新编译并写入缓存。
6. 执行 `load_rules(...) -> RuleEngine().evaluate_rules(...)`，最终输出 JSON 到 stdout。

### 2) 字节码缓存文件头与一致性证据

基于本地检查：

- 三个 `.pyc` 文件均为 `timestamp-based`（bitfield=0）。
- magic number 均为 `cb0d0d0a`（CPython 3.12）。
- `config_loader.cpython-312.pyc` 头中的 `source_size=9690`，与 `config_loader.py` 文件大小一致。
- `rule_engine.cpython-312.pyc` 头中的 `source_size=10727`，与 `rule_engine.py` 文件大小一致。
- 源码时间戳为 `2026-03-19 07:24:47 UTC`，与 `.pyc` 头记录一致。

这说明缓存与当前源码版本是可对应的，而非明显过期缓存。

### 3) 代码对象指纹（从 `.pyc` 提取）

从 `.pyc` 反序列化后可见：

- `config_loader.cpython-312.pyc` 含 `Condition`、`Rule`、`extract_frontmatter`、`load_rules`、`load_rule_file`。
- `rule_engine.cpython-312.pyc` 含 `compile_regex`、`RuleEngine` 及 `_rule_matches/_check_condition/_extract_field/_regex_match` 等方法。
- `__init__.cpython-312.pyc` 仅包含空模块返回逻辑（源文件本身为 0 字节）。

这与对应源码结构一一匹配，证明 `__pycache__` 是 `core` 业务模块的直接编译产物。

### 4) 与业务协议的关系（由被缓存源码决定）

虽然 `__pycache__` 仅是缓存，但其承载的编译逻辑直接决定 Hook 输出协议：

- `PreToolUse/PostToolUse` 阻断时返回：
  - `hookSpecificOutput.permissionDecision = "deny"`
  - `systemMessage`
- `Stop` 阻断时返回：
  - `decision = "block"`
  - `reason`
  - `systemMessage`
- 非阻断或警告场景返回 `systemMessage` 或空 `{}`。

因此，缓存损坏/版本不匹配会直接影响 Hook 协议行为。

### 5) 本次研究使用的关键命令

- `file plugins/hookify/core/__pycache__/*.pyc`
- `stat -c '%n | size=%s | mtime=%y' ...`
- `python3` + `marshal`/`dis` 解析 `.pyc` 代码对象与首行号
- `rg -n "from hookify.core|load_rules|RuleEngine"` 追踪调用链
- `echo ... | CLAUDE_PLUGIN_ROOT=... python3 plugins/hookify/hooks/pretooluse.py` 做最小运行验证

## 关键代码路径与文件引用

- `plugins/hookify/hooks/hooks.json:4`  
  注册 `PreToolUse/PostToolUse/Stop/UserPromptSubmit` 的 Python 命令入口（10s timeout）。

- `plugins/hookify/hooks/pretooluse.py:14`  
  通过 `CLAUDE_PLUGIN_ROOT` 注入路径，触发对 `hookify.core` 的导入（其后即可能命中 `__pycache__`）。

- `plugins/hookify/hooks/posttooluse.py:22`  
  导入 `load_rules` 与 `RuleEngine`，走与 PreToolUse 相同缓存加载路径。

- `plugins/hookify/hooks/stop.py:37`  
  加载 `event='stop'` 规则并执行规则引擎。

- `plugins/hookify/hooks/userpromptsubmit.py:37`  
  加载 `event='prompt'` 规则并执行规则引擎。

- `plugins/hookify/core/config_loader.py:198`  
  规则发现入口：扫描 `.claude/hookify.*.local.md`，并在 `load_rule_file` 中解析 frontmatter。

- `plugins/hookify/core/rule_engine.py:35`  
  规则聚合评估入口，决定 block/warn 与返回协议结构。

- `plugins/hookify/core/__pycache__/config_loader.cpython-312.pyc`  
  `config_loader.py` 的 CPython 3.12 字节码缓存。

- `plugins/hookify/core/__pycache__/rule_engine.cpython-312.pyc`  
  `rule_engine.py` 的 CPython 3.12 字节码缓存。

- `plugins/hookify/.gitignore:2`  
  定义 `__pycache__/` 忽略规则，说明该目录预期为运行期临时产物。

## 依赖与外部交互

1. Python 运行时与导入机制
- 依赖 CPython 的 import/cache 机制、`.pyc` 头校验规则、`__pycache__` 命名约定。

2. 环境变量
- `CLAUDE_PLUGIN_ROOT` 决定 hook 脚本导入路径，间接决定能否命中对应 `__pycache__`。

3. 文件系统 I/O
- 读取 `.claude/hookify.*.local.md`（规则文件）。
- `stop` 事件可能读取 `transcript_path` 文件内容。

4. 标准输入/输出协议
- hook 脚本从 stdin 接收 JSON，向 stdout 输出 JSON；`__pycache__` 中缓存的是该逻辑的字节码实现。

5. 开发工具链交互
- 插件开发工具脚本 `validate-hook-schema.sh` 当前与 plugin wrapper 格式（`{"description","hooks":{...}}`）并非完全兼容，无法直接验证 hookify 的 `hooks.json`。

## 风险、边界与改进建议

1. 版本边界风险（Python 主版本）
- 风险：`cpython-312` 缓存仅适配 3.12；若运行环境切换到 3.11/3.13，会回退重编译或行为差异。
- 建议：在 README/诊断命令中明确 `python3 --version` 要求，并在故障提示里输出解释。

2. 缓存非真源导致的可重复性边界
- 风险：`__pycache__` 是导入副产物，可能因环境、路径、编译时机不同而变化，不适合作为审计或评审依据。
- 建议：继续以 `.py` 为唯一真源；研究与 code review 中把 `.pyc` 作为“证据补充”而非判定依据。

3. 路径信息泄漏细节
- 观察：`__init__.cpython-312.pyc` 的 `co_filename` 为绝对路径（含本机目录），而另外两个为相对路径。
- 风险：若错误地分发缓存文件，可能泄漏构建机路径信息。
- 建议：不分发 `.pyc`；必要时使用统一编译参数（如去路径前缀）构建发布包。

4. 缓存失效与冷启动性能抖动
- 风险：规则文件频繁变更、解释器或时间戳变化时会触发重编译，在 10 秒 hook 超时预算内造成抖动。
- 建议：维持规则数量与 regex 复杂度可控；必要时做一次预热导入或度量 hook 平均时延。

5. 测试与验证覆盖不足
- 现状：`plugins/hookify` 无独立测试目录，`core` 仅有 `if __name__ == '__main__'` 的手工测试片段。
- 建议：
  - 增加最小自动化 smoke tests（至少覆盖 `load_rules` 与 `evaluate_rules`）。
  - 为 plugin wrapper hooks.json 提供兼容的 schema validator，避免现有脚本在 hookify 上报错中断。

