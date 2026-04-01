# statusNoticeHelpers.ts 研究文档

## 场景与职责

`statusNoticeHelpers.ts` 是 `statusNoticeDefinitions.tsx` 的辅助模块，专门负责 Agent 描述 Token 数的计算。该模块将 Token 估算逻辑从通知定义中分离，实现关注点分离和代码复用。

## 功能点目的

### Agent 描述 Token 阈值检查
- **常量**: `AGENT_DESCRIPTIONS_THRESHOLD = 15_000` (15,000 tokens)
- **功能**: 计算所有非内置 Agent 的描述总 Token 数
- **用途**: 
  - 在 `statusNoticeDefinitions.tsx` 中显示大 Agent 描述警告
  - 在 `doctorContextWarnings.ts` 中进行上下文健康检查

## 具体技术实现

### Token 估算方法

```typescript
export function getAgentDescriptionsTotalTokens(
  agentDefinitions?: AgentDefinitionsResult,
): number {
  if (!agentDefinitions) return 0

  return agentDefinitions.activeAgents
    .filter(a => a.source !== 'built-in')  // 排除内置 Agent
    .reduce((total, agent) => {
      const description = `${agent.agentType}: ${agent.whenToUse}`
      return total + roughTokenCountEstimation(description)
    }, 0)
}
```

### 估算策略

1. **内容构造**: 将 `agentType` 和 `whenToUse` 拼接为 `"type: description"` 格式
2. **粗略估算**: 使用 `roughTokenCountEstimation`，基于字符数/4 的算法
3. **过滤内置**: 排除 `source === 'built-in'` 的 Agent，避免误报

## 关键代码路径与文件引用

### 本文件导出
- `AGENT_DESCRIPTIONS_THRESHOLD`: Token 阈值常量 (15,000)
- `getAgentDescriptionsTotalTokens(agentDefinitions)`: 计算函数

### 依赖模块

| 模块 | 用途 |
|------|------|
| `../services/tokenEstimation.js` | `roughTokenCountEstimation` |
| `../tools/AgentTool/loadAgentsDir.js` | `AgentDefinitionsResult` 类型 |

### 调用方

| 文件 | 用途 |
|------|------|
| `src/utils/statusNoticeDefinitions.tsx` | 大 Agent 描述警告的激活条件和渲染 |
| `src/utils/doctorContextWarnings.ts` | Doctor 命令的上下文健康检查 |

## 依赖与外部交互

### Token 估算服务
- 使用 `roughTokenCountEstimation` 进行快速估算
- 算法：字符数 / 4（基于平均每个 Token 4 个字符的假设）
- 适用于粗略估算，不保证精确度

### Agent 定义系统
- 依赖 `AgentDefinitionsResult` 类型
- 访问 `activeAgents` 数组
- 使用 `agentType` 和 `whenToUse` 字段

## 风险、边界与改进建议

### 潜在风险

1. **估算不准确**: 粗略估算可能低估或高估实际 Token 数，特别是对于非英语内容
2. **性能问题**: 如果 Agent 数量极多，reduce 操作可能成为热点

### 边界情况

1. **空 Agent 列表**: 正确处理 `undefined` 和空数组情况
2. **缺失字段**: 假设 `agentType` 和 `whenToUse` 始终存在

### 改进建议

1. **精确 Token 计数**: 考虑使用 API 进行精确计数（对于大量 Agent）
2. **缓存机制**: 如果 Agent 定义不频繁变化，可以缓存计算结果
3. **逐个 Agent 报告**: 当前只返回总数，可以返回每个 Agent 的 Token 数便于调试

```typescript
// 建议：返回详细分解
export function getAgentDescriptionsTokenBreakdown(
  agentDefinitions?: AgentDefinitionsResult,
): { total: number; agents: Array<{ name: string; tokens: number }> } {
  // ...
}
```

4. **配置化阈值**: 将阈值从硬编码改为可配置
