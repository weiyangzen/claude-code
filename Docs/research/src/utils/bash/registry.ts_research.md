# Research: src/utils/bash/registry.ts

## 场景与职责

`registry.ts` 是 Claude Code 中 Bash 命令元数据的**统一注册表与查询入口**。它负责：
1. **定义命令 spec 的类型系统**（`CommandSpec`、`Argument`、`Option`）。
2. **聚合本地手写 spec**（`src/utils/bash/specs/` 目录下的命令补全规范）。
3. **动态加载 Fig autocomplete 规范**（`@withfig/autocomplete` npm 包）。
4. **提供 LRU 缓存的 spec 查询接口** `getCommandSpec(command)`，供前缀提取器、权限验证器、PowerShell 前缀提取器等模块使用。

该模块是**纯数据层**，无业务逻辑，但处于安全前缀提取和命令补全的**关键数据依赖路径**上。

## 功能点目的

### 1. 类型定义：`CommandSpec`、`Argument`、`Option`
- **目的**：为 Bash/Shell 命令的结构化描述提供 TypeScript 契约。
- **关键字段**：
  - `CommandSpec.name`：命令名（如 `git`、`kubectl`）。
  - `CommandSpec.subcommands`：子命令数组（支持递归嵌套）。
  - `CommandSpec.args`：位置参数描述（可为单个或数组）。
  - `CommandSpec.options`：命令行选项（flags）描述。
  - `Argument.isCommand`：标记该参数为"被包装的命令"（如 `timeout 5s <cmd>`）。
  - `Argument.isModule`：标记该参数为模块名（如 `python -m <module>`）。
  - `Argument.isScript`：标记该参数为脚本文件（如 `node script.js`）。
  - `Argument.isDangerous`：标记危险参数（影响前缀深度计算）。
  - `Argument.isVariadic`：标记可变长参数（如 `echo hello world`）。

### 2. `loadFigSpec(command)`
- **目的**：动态加载 `@withfig/autocomplete` 中对应命令的预构建 JS 模块。
- **安全过滤**：
  - 拒绝空命令、含 `/` 或 `\` 的路径型命令、含 `..` 的目录遍历、以 `-` 开头的非法命令。
- **加载方式**：`import(`@withfig/autocomplete/build/${command}.js`)`，利用 ES 动态导入。
- **失败处理**：`catch` 返回 `null`，不抛异常。

### 3. `getCommandSpec(command)`
- **目的**：提供统一的、带缓存的命令 spec 查询。
- **查询优先级**：
  1. **本地 specs**（`src/utils/bash/specs/index.ts` 导出的数组）—— 优先匹配，用于覆盖或补充 Fig 规范。
  2. **Fig autocomplete** —— 通过 `loadFigSpec()` 动态加载。
  3. **返回 `null`** —— 无任何规范时回退到启发式前缀提取（`specPrefix.ts` 中默认深度为 2）。
- **缓存策略**：`memoizeWithLRU`，最大缓存 100 条，以命令名字符串为 key。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 类型系统详细结构
```ts
export type CommandSpec = {
  name: string
  description?: string
  subcommands?: CommandSpec[]
  args?: Argument | Argument[]
  options?: Option[]
}

export type Argument = {
  name?: string
  description?: string
  isDangerous?: boolean
  isVariadic?: boolean
  isOptional?: boolean
  isCommand?: boolean      // wrapper commands e.g. timeout, sudo
  isModule?: string | boolean // python -m, ruby -r
  isScript?: boolean       // node script.js
}

export type Option = {
  name: string | string[]  // 支持多别名，如 ['-h', '--help']
  description?: string
  args?: Argument | Argument[]
  isRequired?: boolean
}
```

### 本地 Spec 聚合
```ts
import specs from './specs/index.js'
// specs 为 CommandSpec[] 数组，包含：alias, nohup, pyright, sleep, srun, time, timeout
```

### Fig 动态加载的安全门
```ts
if (!command || command.includes('/') || command.includes('\\')) return null
if (command.includes('..')) return null
if (command.startsWith('-') && command !== '-') return null
```
这些检查防止了：
- **路径遍历攻击**：`../../../etc/passwd`
- **目录遍历**：`foo/../bar`
- **选项注入**：`-rf`

### LRU 缓存
```ts
export const getCommandSpec = memoizeWithLRU(
  async (command: string): Promise<CommandSpec | null> => {
    const spec =
      specs.find(s => s.name === command) ||
      (await loadFigSpec(command)) ||
      null
    return spec
  },
  (command: string) => command,
)
```
- 缓存大小：默认 100（`memoizeWithLRU` 的默认值）。
- 无 TTL，进程生命周期内有效；内存受 LRU 上限约束。

## 关键代码路径与文件引用

### 依赖（被引用）
| 文件 | 引用方式 | 作用 |
|------|---------|------|
| `src/utils/memoize.ts` | `import { memoizeWithLRU } from '../memoize.js'` | LRU 缓存封装 |
| `src/utils/bash/specs/index.ts` | `import specs from './specs/index.js'` | 本地 spec 聚合 |
| `@withfig/autocomplete/build/${command}.js` | 动态 import | 第三方命令补全规范 |

### 调用方（引用本模块）
| 文件 | 引用符号 | 作用 |
|------|---------|------|
| `src/utils/bash/prefix.ts` | `getCommandSpec` | Bash 前缀提取 |
| `src/utils/shell/specPrefix.ts` | `CommandSpec` (type) | 共享前缀构建逻辑 |
| `src/utils/powershell/staticPrefix.ts` | `getCommandSpec` | PowerShell 前缀提取 |
| `src/utils/bash/specs/*.ts` | `CommandSpec` (type) | 各本地 spec 的类型约束 |

## 依赖与外部交互

- **`@withfig/autocomplete` npm 包**：提供数千个 CLI 工具的 autocomplete 规范。该包在构建时被打包为独立 JS 文件，运行时通过动态 `import()` 按需加载。
- **`lru-cache`（间接）**：`memoizeWithLRU` 内部使用 `lru-cache` 库实现缓存驱逐。
- **本地 spec 目录**：`src/utils/bash/specs/` 包含 7 个手写 spec（`alias.ts`, `nohup.ts`, `pyright.ts`, `sleep.ts`, `srun.ts`, `time.ts`, `timeout.ts`），用于覆盖 Fig 未支持或行为特殊的命令。

## 风险、边界与改进建议

### 风险
1. **动态导入的 I/O 开销**：`loadFigSpec` 每次 miss 都会触发一次文件系统读取（或 bundler 的模块解析）。虽然 LRU 缓存缓解了高频命令的重复读取，但首次调用冷门命令时会有异步 I/O 延迟。
2. **Fig 规范与本地 spec 的语义冲突**：若本地 spec 和 Fig spec 对同一命令的参数定义不一致（如 `isCommand` 标记位置不同），可能导致前缀提取行为在不同环境（ant build vs external build）下出现差异。
3. **无缓存预热**：在 CLI 启动时未预加载常用命令的 spec，导致用户第一次输入 `git` 或 `npm` 时可能感知到前缀计算的轻微延迟。

### 边界
- `loadFigSpec` 仅接受纯命令名，**不支持带路径的命令**（如 `./script.sh`、`/usr/bin/git`）。这类命令永远返回 `null`，回退到默认深度 2 的启发式前缀提取。
- Fig 规范依赖运行时动态导入，在**某些打包环境**（如纯 browser bundle 或严格 CSP）可能完全不可用。
- `getCommandSpec` 的缓存 key 仅为命令名字符串，**不考虑版本或环境变化**。若用户升级了 `@withfig/autocomplete` 包，需要重启进程才能刷新缓存。

### 改进建议
1. **预加载热门命令**：在应用启动时异步预加载 `git`、`npm`、`docker`、`kubectl` 等高频命令的 spec，消除首次交互延迟。
2. **支持路径型命令的 basename 提取**：对于 `/usr/bin/git` 类的输入，可尝试提取 basename `git` 后再查询 spec，提升覆盖率。
3. **缓存失效机制**：考虑在 `memoizeWithLRU` 层增加基于文件修改时间的缓存失效，或在包更新时自动清理缓存。
4. **类型安全增强**：`Argument.isModule` 当前为 `string | boolean`，语义较模糊。建议拆分为更明确的联合类型，如 `{ type: 'module', language: 'python' }`。
