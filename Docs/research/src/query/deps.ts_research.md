# src/query/deps.ts 研究文档

## 场景与职责

`deps.ts` 是 Claude Code 查询模块的依赖注入层，为 `query()` 函数提供 I/O 依赖的抽象。通过 `QueryDeps` 类型和 `productionDeps()` 工厂函数，实现测试时可注入 mock 对象，避免每个测试文件都需要 `spyOn-per-module` 的繁琐操作。

### 核心设计原则
1. **依赖注入**: 通过 `QueryParams.deps` 参数允许测试直接注入 fake 实现
2. **类型同步**: 使用 `typeof fn` 保持签名与实际实现自动同步
3. **窄范围**: 当前仅包含 4 个核心依赖，证明模式后可逐步扩展
4. **零额外成本**: 测试导入此文件获取类型时，已通过 `query.ts` 的导入链加载，无新增模块图成本

## 功能点目的

### QueryDeps 类型
定义查询函数所需的 I/O 依赖接口：
- `callModel`: 调用 Claude API 进行流式模型查询
- `microcompact`: 执行微压缩（microcompact）优化消息历史
- `autocompact`: 按需自动压缩对话上下文
- `uuid`: 生成唯一标识符（用于查询链追踪）

### productionDeps() 函数
返回生产环境实际依赖实现：
- `queryModelWithStreaming`: 实际的 API 调用实现
- `microcompactMessages`: 微压缩服务
- `autoCompactIfNeeded`: 自动压缩服务
- `randomUUID`: Node.js crypto 模块的 UUID 生成

## 具体技术实现

### 关键流程

```typescript
export type QueryDeps = {
  callModel: typeof queryModelWithStreaming
  microcompact: typeof microcompactMessages
  autocompact: typeof autoCompactIfNeeded
  uuid: () => string
}

export function productionDeps(): QueryDeps {
  return {
    callModel: queryModelWithStreaming,
    microcompact: microcompactMessages,
    autocompact: autoCompactIfNeeded,
    uuid: randomUUID,
  }
}
```

### 数据结构

```typescript
export type QueryDeps = {
  // -- model
  callModel: typeof queryModelWithStreaming

  // -- compaction
  microcompact: typeof microcompactMessages
  autocompact: typeof autoCompactIfNeeded

  // -- platform
  uuid: () => string
}
```

### 依赖模块

| 依赖 | 用途 |
|------|------|
| `crypto` | `randomUUID` 原生 UUID 生成 |
| `../services/api/claude.js` | `queryModelWithStreaming` API 调用 |
| `../services/compact/autoCompact.js` | `autoCompactIfNeeded` 自动压缩 |
| `../services/compact/microCompact.js` | `microcompactMessages` 微压缩 |

## 关键代码路径与文件引用

### 调用方
- `src/query.ts:103` - 导入类型和工厂函数
  ```typescript
  import { productionDeps, type QueryDeps } from './query/deps.js'
  ```

### 使用点
- `src/query.ts:263` - 解析依赖（使用传入的或生产默认值）
  ```typescript
  const deps = params.deps ?? productionDeps()
  ```

- `src/query.ts:353` - 生成查询链 ID
  ```typescript
  chainId: deps.uuid(),
  ```

- `src/query.ts:414` - 微压缩调用
  ```typescript
  const microcompactResult = await deps.microcompact(
    messagesForQuery,
    toolUseContext,
    querySource,
  )
  ```

- `src/query.ts:454` - 自动压缩调用
  ```typescript
  const { compactionResult, consecutiveFailures } = await deps.autocompact(
    messagesForQuery,
    toolUseContext,
    {...},
    querySource,
    tracking,
    snipTokensFreed,
  )
  ```

- `src/query.ts:523` - 压缩后生成新的 turn ID
  ```typescript
  turnId: deps.uuid(),
  ```

- `src/query.ts:659` - 模型调用
  ```typescript
  for await (const message of deps.callModel({...}))
  ```

## 依赖与外部交互

### 上游依赖
1. **services/api/claude.ts**: 提供 `queryModelWithStreaming` 函数
2. **services/compact/autoCompact.ts**: 提供 `autoCompactIfNeeded` 函数
3. **services/compact/microCompact.ts**: 提供 `microcompactMessages` 函数

### 下游消费
1. **query.ts**: 主查询循环消费这些依赖

### 测试使用模式
```typescript
// 测试中注入 mock
const mockDeps: QueryDeps = {
  callModel: jest.fn(),
  microcompact: jest.fn(),
  autocompact: jest.fn(),
  uuid: () => 'test-uuid',
}

const result = query({ ..., deps: mockDeps })
```

## 风险、边界与改进建议

### 风险点
1. **范围有限**: 当前仅 4 个依赖，其他常见 mock（如 `runTools`, `handleStopHooks`, `logEvent`, queue ops）仍需 `spyOn`
2. **类型耦合**: 使用 `typeof fn` 虽然保持同步，但意味着测试必须导入实际模块才能获取类型

### 边界情况
1. **模块图成本**: 注释说明测试导入此文件已通过 `query.ts` 链加载，无额外成本，但这依赖于具体的模块加载顺序
2. **部分 mock**: 可以只覆盖部分依赖，其余使用生产实现

### 改进建议
1. **扩展依赖范围**: 按照注释中的 TODO，逐步添加 `runTools`, `handleStopHooks`, `logEvent`, queue ops 等
2. **抽象接口**: 考虑将 `typeof fn` 改为显式接口定义，减少测试对实际模块的依赖
3. **工厂模式**: 添加 `createTestDeps()` 辅助函数，提供常用 mock 的默认实现
4. **文档化**: 添加更多使用示例，展示如何在不同测试场景中注入依赖

### 架构演进
该文件代表了从模块级 mock 向依赖注入的架构演进：
- **之前**: 每个测试文件使用 `jest.spyOn()` 或 `vi.mock()` 模拟模块
- **现在**: 通过 `deps` 参数直接注入 fake 实现
- **未来**: 可能发展为完整的 DI 容器，支持更复杂的依赖生命周期管理
