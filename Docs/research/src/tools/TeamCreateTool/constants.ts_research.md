# constants.ts 研究文档

## 场景与职责

`constants.ts` 是 `TeamCreateTool` 的常量定义文件，承担单一职责：为工具提供一个稳定、可引用的名称标识符。该常量被多个模块跨目录引用，是打破潜在循环依赖、保证工具名一致性的关键节点。

## 功能点目的

- **消除魔法字符串**：避免在工具实现、权限分类器、Coordinator 模式白名单、UI 组件等各处硬编码 `'TeamCreate'`。
- **支持 Tree-Shaking 与编译优化**：作为独立的叶子模块，它几乎无依赖，Bun 打包时可以被安全地内联或按需保留。
- **类型安全引用**：TypeScript 项目通过导入该常量获得编译时检查，防止拼写错误导致的运行时工具匹配失败。

## 具体技术实现

### 代码内容

```ts
export const TEAM_CREATE_TOOL_NAME = 'TeamCreate'
```

- 仅一行导出，无运行时逻辑。
- 采用 PascalCase 字符串 `'TeamCreate'`，与 Claude Code 工具命名规范保持一致（如 `TeamDelete`、`Agent`、`Bash` 等）。

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/tools/TeamCreateTool/constants.ts` | 本文件，定义 `TEAM_CREATE_TOOL_NAME` |
| `src/tools/TeamCreateTool/TeamCreateTool.ts` | 导入作为 `buildTool({ name: TEAM_CREATE_TOOL_NAME })` |
| `src/tools.ts` | 动态 `require` 引入 `TeamCreateTool` 后，用该常量进行类型断言与注册 |
| `src/coordinator/coordinatorMode.ts` | 将常量加入 `INTERNAL_WORKER_TOOLS` 白名单 |
| `src/utils/swarm/inProcessRunner.ts` | 引用常量进行 in-process 队友的工具权限过滤 |
| `src/tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts` | 引用常量用于 plan mode 上下文判断 |
| `src/utils/permissions/classifierDecision.ts` | 将常量加入 `SAFE_YOLO_ALLOWLISTED_TOOLS` |
| `src/components/permissions/ExitPlanModePermissionRequest/ExitPlanModePermissionRequest.tsx` | 引用常量进行 UI 层面的工具名匹配 |

## 依赖与外部交互

### 调用方

本文件作为纯常量模块，被以下模块静态导入：

- `src/tools/TeamCreateTool/TeamCreateTool.ts`
- `src/coordinator/coordinatorMode.ts`
- `src/utils/swarm/inProcessRunner.ts`
- `src/tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts`
- `src/utils/permissions/classifierDecision.ts`
- `src/components/permissions/ExitPlanModePermissionRequest/ExitPlanModePermissionRequest.tsx`

### 被调用方/依赖模块

- 无。该模块是零依赖的叶子节点。

## 风险、边界与改进建议

### 风险与边界

1. **重命名成本高**
   - 由于该常量被 6 个以上模块直接引用，若产品决定更改工具显示名称或内部标识符，需要同步修改所有引用点以及可能的外部文档。不过当前字符串 `'TeamCreate'` 已稳定使用，变更概率低。

2. **无版本控制或兼容性别名**
   - 如果未来对工具进行重命名（例如改为 `CreateTeam`），本文件没有提供 `aliases` 字段（那是 `Tool` 接口层面的能力）。需要在 `TeamCreateTool.ts` 的 `buildTool` 中额外配置 `aliases` 以支持向后兼容。

### 改进建议

1. **保持现状即可**
   - 该文件职责单一、边界清晰，符合"一个文件只做一件事"的原则。50 字节的大小也证明了它没有过度设计。

2. **若未来扩展，可考虑共置相关常量**
   - 如果 `TeamCreateTool` 需要更多配置常量（如默认团队描述、最大团队名称长度等），可继续在本文件中追加导出，保持命名空间内聚。但应避免引入运行时依赖或复杂逻辑。
