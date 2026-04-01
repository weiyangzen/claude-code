# stuck.ts 研究文档

## 场景与职责

`stuck.ts` 实现了 `/stuck` 内置技能，用于诊断当前机器上其他 Claude Code 会话是否出现冻结、卡住或极慢的情况。该技能仅对 Anthropic 内部员工（`USER_TYPE === 'ant'`）可用，指导模型通过 `ps`、`pgrep`、`sample` 等系统命令收集诊断信息，并将发现的异常会话报告发布到内部 Slack 频道 `#claude-code-feedback`。

## 功能点目的

1. **进程扫描**：列出所有名为 `claude` 或 `cli`（且命令行包含 "claude"）的进程，排除当前会话自身。
2. **异常指标识别**：
   - 高 CPU（≥90% 持续）→ 可能无限循环
   - 状态 `D`（不可中断睡眠）→ I/O 挂起
   - 状态 `T`（已停止）→ 用户可能误按 Ctrl+Z
   - 状态 `Z`（僵尸）→ 父进程未回收
   - 高 RSS（≥4GB）→ 可能内存泄漏
   - 子进程挂起 → `git`、`node` 等子进程可能冻结父进程
3. **诊断报告**：仅当确实发现异常时，通过 Slack MCP 工具向 `#claude-code-feedback` 发送两消息结构（顶层摘要 + 线程详情）。
4. **安全约束**：明确禁止杀死或向任何进程发送信号，仅做诊断。

## 具体技术实现

### 关键流程

- `registerStuckSkill()`：
  1. 若 `process.env.USER_TYPE !== 'ant'` 直接返回
  2. 否则 `registerBundledSkill({ name: 'stuck', ... })`
- `getPromptForCommand(args)` 入口：
  1. 返回固定的 `STUCK_PROMPT`
  2. 若用户提供了额外上下文（如特定 PID 或症状），追加 `## User-provided context`

### Prompt 中的核心调查命令

```bash
# 列出所有 Claude Code 进程
ps -axo pid=,pcpu=,rss=,etime=,state=,comm=,command= | grep -E '(claude|cli)' | grep -v grep

# 子进程
pgrep -lP <pid>

# macOS 原生栈采样（可选）
sample <pid> 3
```

### 报告结构

1. **Top-level message**：单行摘要（hostname + 版本 + 症状）
2. **Thread reply**：完整诊断信息（PID、CPU%、RSS、状态、运行时间、命令行、子进程、诊断结论、debug log 尾部或 sample 输出）

### 注册参数

| 字段 | 值 |
|------|-----|
| `name` | `'stuck'` |
| `userInvocable` | `true` |

## 关键代码路径与文件引用

- 源文件：`src/skills/bundled/stuck.ts`
- 注册入口：`src/skills/bundled/index.ts`
- 核心注册器：`src/skills/bundledSkills.ts`
- 无其他外部代码依赖

## 依赖与外部交互

- **纯 prompt 生成器**：`stuck.ts` 本身不执行任何系统命令，所有诊断操作由模型根据 prompt 中的指令通过 `Bash` 工具完成。
- **Slack MCP 依赖**：报告发布依赖于 Slack MCP 工具的可用性；若不可用，prompt 要求模型将报告格式化为用户可复制粘贴的文本。
- **Ant-only 门控**：通过 `process.env.USER_TYPE !== 'ant'` 在模块加载时直接短路。

## 风险、边界与改进建议

1. **边界：仅限 macOS/Linux**：prompt 中给出的 `ps` 命令和 `sample` 工具是 Unix/macOS 特有的，Windows 用户（或远程 Windows 环境）无法直接执行这些诊断步骤。
2. **边界：Slack 频道硬编码**：`C07VBSHV7EV`（`#claude-code-feedback`）和 `MACRO.FEEDBACK_CHANNEL` 在代码中隐式引用，若内部 Slack 工作区结构变化，需要同步修改。
3. **风险：进程名匹配过于宽泛**：`grep -E '(claude|cli)'` 可能匹配到不相关的 `claude` 或 `cli` 进程（如其他名为 `cli` 的应用），导致误报。
4. **风险：诊断完全依赖模型遵循 prompt**：没有强制性的安全检查来确保模型不会误杀进程，虽然 prompt 明确禁止，但这仍是软性约束。
5. **改进建议**：
   - 增加 Windows 平台的诊断命令（如 `Get-Process`、WMI 查询）到 prompt 中，或根据 `process.platform` 动态生成平台特定的诊断指南。
   - 将 Slack channel ID 提取为可配置常量，避免多处硬编码。
   - 在 prompt 中增加更精确的进程过滤规则（如要求 `command=` 列同时包含 `claude` 和 `node`/`bun` 关键字）。
   - 考虑将 `/stuck` 的核心诊断逻辑封装为一个真正的内置 agent（如 `stuck-diagnosis` agent），使其行为更可预测，并能在 agent 层增加安全校验（如禁止 `kill` 命令）。
