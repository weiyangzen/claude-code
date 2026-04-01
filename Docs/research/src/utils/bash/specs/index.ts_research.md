# index.ts 研究文档

## 场景与职责

`index.ts` 是 `src/utils/bash/specs/` 目录的聚合入口（barrel file），负责将散落在同目录下的各个命令规格模块统一导出为一个 `CommandSpec[]` 数组。该数组被 `src/utils/bash/registry.ts` 引入，作为 `getCommandSpec` 函数的第一查找来源——优先于动态加载的 `@withfig/autocomplete` 外部规格库。

在整个 bash 安全与解析子系统中，该文件处于“数据汇聚层”：它本身不含业务逻辑，但决定了哪些命令拥有本地覆盖规格、哪些命令必须回退到 Fig 动态加载。其导出顺序会轻微影响 `specs.find()` 的匹配优先级（虽然各 spec 的 `name` 唯一，不存在冲突）。

## 功能点目的

1. **统一规格注册**：将 `alias.ts`、`nohup.ts`、`pyright.ts`、`sleep.ts`、`srun.ts`、`time.ts`、`timeout.ts` 等模块聚合为单一数组，方便 `registry.ts` 批量消费。
2. **提供本地覆盖能力**：对于 Fig 库中缺失或不够精确的命令（如 `pyright`、`srun`、`sleep`），通过本地 spec 保证前缀提取与安全校验的准确性。
3. **控制加载时依赖**：采用静态 ES Module 导入，使构建工具（Bun bundler）在编译期就能完成模块链接，避免运行时动态解析目录的开销与不确定性。
4. **支持离线/受限环境**：在无法访问 `node_modules/@withfig/autocomplete` 或动态 `import()` 被限制的运行时（如某些 native build），本地 specs 是唯一的命令语义来源。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 数据结构

文件导出一个通过 `satisfies CommandSpec[]` 约束的数组字面量：

```typescript
export default [
  pyright,
  timeout,
  sleep,
  alias,
  nohup,
  time,
  srun,
] satisfies CommandSpec[]
```

- `satisfies`（TypeScript 4.9+）确保数组中每个元素都符合 `CommandSpec` 类型，但保留各元素的精确字面量类型，便于下游类型推断。
- 数组顺序为：`pyright` → `timeout` → `sleep` → `alias` → `nohup` → `time` → `srun`。

### 关键流程

1. **初始化流程**：当 `registry.ts` 首次被加载时，`import specs from './specs/index.js'` 触发该模块求值。所有被导入的子模块同步执行，最终数组赋值给 `specs` 变量。
2. **查询流程**：`getCommandSpec(command)` 被调用时，执行 `specs.find(s => s.name === command)`。由于 `specs` 是本地静态数组，查找为同步 O(n) 操作，结果通过 `memoizeWithLRU` 缓存。
3. **回退流程**：若 `find` 未命中，则进入 `loadFigSpec(command)`，尝试异步动态导入 `@withfig/autocomplete/build/${command}.js`。

### 与构建系统的关系

- Bun bundler 处理该文件时，会将所有静态导入的 spec 模块打包到同一 chunk（或相关 chunk）中。由于不存在动态 `import()` 表达式，这些模块在启动时即驻留内存。
- 对于 native build，若 `@withfig/autocomplete` 未被包含在产物中，本地 specs 的静态存在保证了核心命令（如 `timeout`、`nohup`）仍有规格可用。

## 关键代码路径与文件引用

- **本文件**：`src/utils/bash/specs/index.ts`（18 行）
- **消费方**：`src/utils/bash/registry.ts`（第 2 行导入，第 47 行使用）
- **子模块**：
  - `src/utils/bash/specs/alias.ts`
  - `src/utils/bash/specs/nohup.ts`
  - `src/utils/bash/specs/pyright.ts`
  - `src/utils/bash/specs/sleep.ts`
  - `src/utils/bash/specs/srun.ts`
  - `src/utils/bash/specs/time.ts`
  - `src/utils/bash/specs/timeout.ts`
- **类型定义**：`src/utils/bash/registry.ts`（`CommandSpec`，第 4–10 行）
- **缓存层**：`src/utils/memoize.ts`（`memoizeWithLRU`，被 `registry.ts` 第 44 行使用）

## 依赖与外部交互

### 内部依赖

| 依赖文件 | 说明 |
|---------|------|
| `./alias.js` 等 7 个同级模块 | 静态导入，构成数组元素。 |
| `../registry.js` | 引入 `CommandSpec` 类型用于 `satisfies` 约束。 |

### 外部交互

- **无直接外部依赖**：不调用第三方库、网络或文件系统。
- **间接影响 Fig 回退策略**：本地数组越完整，需要异步加载 Fig spec 的场景越少，前缀提取的同步路径命中率越高，延迟越低。

## 风险、边界与改进建议

### 风险

1. **顺序无意义但可能被误读**：当前数组顺序看起来是随意的（既非字母序也非重要性排序）。新维护者可能误以为顺序会影响优先级或加载顺序。实际上由于 `find` 按 `name` 精确匹配，顺序只影响极微量的遍历开销。
2. **规模膨胀隐患**：随着项目演进，可能会有更多命令被加入本地 specs。如果数量增长到数十上百个，静态数组的 O(n) 查找虽然仍在 LRU 缓存保护下，但首次加载的内存占用和 bundler 分析时间会线性增长。
3. **与 Fig 规格冲突风险**：若未来 Fig 官方库也提供了 `pyright` 或 `srun` 的 spec，本地数组会优先命中，可能掩盖 Fig 版本的更新。这通常是设计意图（本地覆盖），但需要维护者意识到本地 spec 的“最高优先级”地位。

### 边界

- **只读聚合**：该文件不对 spec 对象做任何修改（如添加、删除字段），各 spec 模块的导出对象保持原样。
- **无运行时配置**：无法通过环境变量或配置文件在运行时增减 spec，必须修改源码并重新构建。

### 改进建议

1. **改为字母序排列**：将数组按 `name` 字母顺序排列（`alias`、`nohup`、`pyright`、`sleep`、`srun`、`time`、`timeout`），降低未来合并冲突概率，并明确传达“顺序无关”的信息。
2. **考虑构建时生成**：如果本地 spec 数量继续增长，可引入构建脚本扫描 `src/utils/bash/specs/*.ts`（排除 `index.ts`）并自动生成该数组，避免手动增删的遗漏。
3. **增加注释说明优先级**：在数组上方添加注释，明确指出本地 spec 优先于 Fig 动态加载，提醒维护者本地覆盖的语义权重。
4. **导出映射（Map）替代数组**：若 spec 数量显著增加，可在 `registry.ts` 中将数组转换为 `Map<string, CommandSpec>`，将查找复杂度从 O(n) 降至 O(1)。当前 7 个元素无此必要，但可作为未来重构方向。
