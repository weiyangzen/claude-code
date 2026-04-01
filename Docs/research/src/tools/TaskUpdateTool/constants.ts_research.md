# 研究报告：src/tools/TaskUpdateTool/constants.ts

## 场景与职责

`constants.ts` 是 `TaskUpdateTool` 模块的单一职责常量文件，仅导出工具名称字符串 `TASK_UPDATE_TOOL_NAME`。它在整个 Claude Code 仓库中作为该工具的全局标识符（symbolic name）被广泛使用，起到“名称单一来源（single source of truth）”的作用。任何需要以字符串形式引用 `TaskUpdate` 工具的代码（工具注册、权限分类器、消息附件、swarm 运行器等）都应导入此常量，而非硬编码 `'TaskUpdate'`。

## 功能点目的

1. **消除魔法字符串**：将工具名从业务逻辑中抽离，避免在数十个文件中分散书写 `'TaskUpdate'`，降低拼写错误和重构遗漏风险。
2. **支持编译时/类型安全引用**：TypeScript 导入机制使得重命名或查找所有引用变得可靠；若直接写字符串字面量，IDE 的“重命名符号”功能无法覆盖全部场景。
3. **作为权限与编排系统的标识符**：Claude Code 的权限系统（classifierDecision、constants/tools）、消息提醒系统（messages、attachments）以及 swarm in-process runner 均以该常量作为白名单/黑名单的匹配键。

## 具体技术实现

文件内容极为精简，仅有一行导出：

```ts
export const TASK_UPDATE_TOOL_NAME = 'TaskUpdate'
```

无运行时逻辑、无依赖、无 side effect。模块加载成本可忽略不计。

## 关键代码路径与文件引用

以下列出了所有直接导入 `TASK_UPDATE_TOOL_NAME` 的源码位置及其用途：

| 引用文件 | 行号 | 用途说明 |
|----------|------|----------|
| `src/tools/TaskUpdateTool/TaskUpdateTool.ts` | 30 | 作为 `buildTool({ name: TASK_UPDATE_TOOL_NAME })` 的注册名 |
| `src/tools.ts` | 84 | 导入 `TaskUpdateTool` 本身；工具组装逻辑间接依赖该名称 |
| `src/utils/permissions/classifierDecision.ts` | 17 | 将工具名加入自动安全分类器（auto-classifier）可识别的工具集合 |
| `src/utils/messages.ts` | 145 | 在任务提醒系统提示中引用，建议模型使用 `TaskUpdate` 更新任务状态 |
| `src/utils/attachments.ts` | 27 | 在计算距离上次任务管理行为的轮次时，将 `TaskUpdate` 视为任务管理事件 |
| `src/utils/swarm/inProcessRunner.ts` | 55 | 为 in-process teammate 构建强制工具白名单时，确保 `TaskUpdate` 始终可用 |
| `src/constants/tools.ts` | 24 | 在 `ASYNC_AGENT_ALLOWED_TOOLS`、`CUSTOM_AGENT_DISALLOWED_TOOLS` 等常量数组中引用 |

## 依赖与外部交互

- **无上游运行时依赖**：该文件不 import 任何其他模块。
- **下游广泛静态依赖**：上述 7 个文件在编译期依赖此常量。由于它是值导出（value export）而非类型导出，任何修改都会触发这些模块的重新类型检查，但不会改变运行时行为（只要字符串值不变）。
- **与 `TaskUpdateTool.ts` 的共生关系**：`TaskUpdateTool.ts` 在定义 `buildTool` 时直接消费该常量；若常量值被修改，工具在模型侧的 wire name、权限系统的匹配键、UI 渲染中的名称会同步变化。

## 风险、边界与改进建议

### 风险与边界

1. **重命名成本被低估**：虽然常量集中了字符串值，但 `TASK_UPDATE_TOOL_NAME` 这个标识符本身在 7 个文件中被显式 import。若未来决定给工具改名（例如从 `TaskUpdate` 改为 `TaskModify`），除了修改本文件中的字符串，还需要更新所有 import 语句中的标识符名（虽然值改一处即可，但符号重命名仍需 IDE 全局重构）。
2. **无版本或别名机制**：与 `AgentTool` 拥有 `LEGACY_AGENT_TOOL_NAME` 不同，`TaskUpdateTool` 目前没有任何向后兼容的别名。若未来需要改名，已持久化的权限规则、会话恢复数据、hook 配置中硬编码的 `'TaskUpdate'` 将全部失效，除非额外引入迁移逻辑。
3. **字符串与运行时行为强耦合**：某些外部系统（如用户自定义的 `.claude/settings.json` 权限规则）可能由用户手写 `'TaskUpdate'`。常量文件无法约束这些外部输入，导致“源码内统一、源码外仍散落魔法字符串”的局面。

### 改进建议

1. **考虑引入遗留别名常量**：若产品层面预计未来可能 rebranding 该工具，可提前在 `constants.ts` 中增加 `export const LEGACY_TASK_UPDATE_TOOL_NAME = 'TaskUpdate'`，并在 `buildTool` 的 `aliases` 字段中注册，为未来平滑迁移留有余地。
2. **与权限规则解析器联动**：在权限系统的规则解析阶段，增加对已知工具名别名的自动归一化（normalization），使得即使用户在外部配置中写入了旧名称，也能被正确映射到当前常量值。
3. **保持极简风格**：该文件目前职责单一、边界清晰，不建议在此处堆砌其他配置（如超时时间、结果大小限制等）。那些属于工具行为配置，应留在 `TaskUpdateTool.ts` 或专门的配置文件中。
