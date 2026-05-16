# FILE `plugins/hookify/core/__init__.py` 研究文档

## 场景与职责

`plugins/hookify/core/__init__.py` 是 Hookify 核心包（`hookify.core`）的包边界文件。  
它本身无业务逻辑（0 bytes），但在运行链路里承担“可导入性锚点”职责：

1. 让 `hookify.core` 在解释器看来是一个显式 package。
2. 配合 `hooks/*.py` 的路径注入逻辑，确保 `from hookify.core...` 导入稳定。
3. 为 `config_loader.py` 与 `rule_engine.py` 提供统一命名空间入口。

典型运行场景：

1. Claude Hook 触发 `PreToolUse/PostToolUse/Stop/UserPromptSubmit`。
2. `plugins/hookify/hooks/*.py` 先通过 `CLAUDE_PLUGIN_ROOT` 调整 `sys.path`。
3. 执行 `from hookify.core.config_loader import load_rules`、`from hookify.core.rule_engine import RuleEngine`。
4. 解释器解析 `hookify.core` 包时会触达该 `__init__.py`（即便文件为空，也完成包初始化步骤）。

## 功能点目的

1. 包初始化占位：保证 `hookify.core` 命名空间与下游模块的导入语义一致。
2. 结构契约表达：在目录层面明确“这是可导入 Python 包，不只是源码文件夹”。
3. 最小副作用：保持空实现，避免在包导入阶段引入任何运行时开销或副作用。

## 具体技术实现（关键流程/数据结构/协议/命令）

该文件无代码语句，技术实现依赖 Python import 机制本身：

1. `hooks/*.py` 的启动逻辑读取环境变量 `CLAUDE_PLUGIN_ROOT` 并向 `sys.path` 注入路径。
2. 导入 `hookify.core.config_loader` 时，解释器先解析包层级 `hookify -> core`。
3. `core` 层级由本文件定义为常规 package；随后加载 `config_loader.py` / `rule_engine.py`。

与协议的关系：

- 它不直接读写 Hook JSON 协议（stdin/stdout），但它是协议执行链路可导入的前提模块之一。
- 一旦导入失败，hook 脚本会降级输出 `systemMessage` 并 `exit 0`，表现为规则系统失效但主流程不硬失败。

## 关键代码路径与文件引用

- 包边界文件：`plugins/hookify/core/__init__.py`
- 直接依赖该包路径的调用方：
  - `plugins/hookify/hooks/pretooluse.py:26`
  - `plugins/hookify/hooks/posttooluse.py:22`
  - `plugins/hookify/hooks/stop.py:22`
  - `plugins/hookify/hooks/userpromptsubmit.py:22`
- 导入路径注入逻辑：
  - `plugins/hookify/hooks/pretooluse.py:14-23`
  - `plugins/hookify/hooks/posttooluse.py:13-19`
  - `plugins/hookify/hooks/stop.py:13-19`
  - `plugins/hookify/hooks/userpromptsubmit.py:13-19`
- 下游实际业务模块：
  - `plugins/hookify/core/config_loader.py`
  - `plugins/hookify/core/rule_engine.py`

## 依赖与外部交互

1. 依赖 Python 导入系统（package/module resolution）。
2. 依赖 hook 入口脚本设置的 `sys.path` 与 `CLAUDE_PLUGIN_ROOT` 环境变量。
3. 与外部系统无直接 I/O、网络、命令执行交互。
4. 间接影响：导入链成功与否会影响 hook 执行器是否能进入规则加载与规则判定流程。

## 风险、边界与改进建议

1. 风险：空文件不暴露显式 API 边界。
- 当前 `hookify.core` 没有 `__all__` 或统一导出，调用方都要写到具体模块路径，后续重构模块名时改动面较大。
- 改进：可在该文件中增加轻量 re-export（如 `Rule`, `Condition`, `RuleEngine`, `load_rules`）并声明 `__all__`。

2. 风险：包级无注释，阅读成本转移到目录上下文。
- 对新维护者来说，`__init__.py` 为空会让“它是否故意为空”不够明确。
- 改进：补一个简短模块注释，说明该文件仅用于 package 边界与未来公共 API 聚合。

3. 边界：不应在该文件引入重逻辑。
- 任何 I/O 或复杂初始化都会在每次 hook 进程导入时触发，直接占用 `hooks.json` 的 10 秒超时预算。
- 改进：保持“零副作用导入”原则，业务逻辑继续留在 `config_loader.py` 与 `rule_engine.py`。
