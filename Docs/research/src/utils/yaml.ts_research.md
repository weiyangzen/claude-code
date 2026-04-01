# yaml.ts 研究文档

## 场景与职责

`yaml.ts` 提供 YAML 解析功能的统一封装。根据运行时环境（Bun 或 Node.js）自动选择最优的解析实现，在 Bun 环境下使用原生内置的 `Bun.YAML`，在 Node.js 环境下回退到 `yaml` npm 包。

**核心使用场景：**
- 解析配置文件（如 `.github/workflows/*.yml`）
- 解析 API 响应中的 YAML 格式数据
- 解析用户提供的 YAML 输入

## 功能点目的

### YAML 解析 (`parseYaml`)
- **目的**：将 YAML 字符串解析为 JavaScript 对象
- **优化**：在 Bun 环境下使用零开销的原生实现
- **兼容性**：在 Node.js 环境下使用 `yaml` 包

## 具体技术实现

### 核心代码

```typescript
/**
 * YAML parsing wrapper.
 *
 * Uses Bun.YAML (built-in, zero-cost) when running under Bun, otherwise falls
 * back to the `yaml` npm package. The package is lazy-required inside the
 * non-Bun branch so native Bun builds never load the ~270KB yaml parser.
 */
export function parseYaml(input: string): unknown {
  if (typeof Bun !== 'undefined') {
    return Bun.YAML.parse(input)
  }
  // eslint-disable-next-line @typescript-eslint/no-require-imports
  return (require('yaml') as typeof import('yaml')).parse(input)
}
```

### 运行时检测

```typescript
// Bun 运行时检测
if (typeof Bun !== 'undefined') {
  return Bun.YAML.parse(input)
}
```

**说明：**
- `typeof Bun` 检查是检测 Bun 运行时的标准方法
- 在 Bun 环境下，`Bun` 是全局对象
- 在 Node.js 环境下，`Bun` 是 `undefined`

### 延迟加载策略

```typescript
// 使用 require 而非 import，实现延迟加载
return (require('yaml') as typeof import('yaml')).parse(input)
```

**优势：**
- Bun 构建时不会打包 `yaml` 包（约 270KB）
- 只在实际需要在 Node.js 环境执行时才加载
- 减少 Bun 构建产物的体积

## 关键代码路径与文件引用

### 导出函数
- `src/utils/yaml.ts:9` - `parseYaml(input: string): unknown`

### 依赖
| 依赖 | 用途 | 加载方式 |
|------|------|----------|
| `Bun.YAML` | Bun 原生 YAML 解析 | 全局对象 |
| `yaml` npm 包 | Node.js YAML 解析 | 延迟 require |

## 依赖与外部交互

### 外部依赖
```typescript
// 无显式 import，使用运行时检测和 require
```

### 内部依赖
无内部依赖。

### package.json 依赖
```json
{
  "dependencies": {
    "yaml": "^2.x.x"
  }
}
```

## 风险、边界与改进建议

### 已知风险

1. **运行时检测依赖**
   - 依赖 `typeof Bun` 检查，如果 Bun 改变全局对象名称，会失效
   - 但这是 Bun 的标准检测方法，稳定性较高

2. **require 的使用**
   - 使用 `require` 而非 ESM `import` 可能影响 Tree Shaking
   - 但这是实现延迟加载的必要手段
   - ESLint 已禁用相关规则

3. **类型断言**
   - 使用 `as typeof import('yaml')` 进行类型断言
   - 如果 `yaml` 包 API 改变，类型检查无法捕获

4. **无序列化功能**
   - 当前只有 `parseYaml`（解析），没有 `stringifyYaml`（序列化）
   - 如果需要输出 YAML，需要额外实现

### 边界情况

1. **无效 YAML**
   - 行为取决于底层实现（Bun.YAML 或 `yaml` 包）
   - 通常会抛出解析错误

2. **空字符串**
   - 行为取决于底层实现
   - 可能返回 `undefined`、`null` 或空对象

3. **多文档 YAML**
   - `Bun.YAML.parse` 可能只返回第一个文档
   - `yaml` 包有 `parseAllDocuments` 方法处理多文档

4. **大文件解析**
   - 没有流式解析支持
   - 整个 YAML 内容需要作为字符串传入

### 改进建议

1. **添加序列化功能**
   ```typescript
   export function stringifyYaml(value: unknown): string {
     if (typeof Bun !== 'undefined') {
       return Bun.YAML.stringify(value)
     }
     return require('yaml').stringify(value)
   }
   ```

2. **错误处理增强**
   ```typescript
   export class YamlParseError extends Error {
     constructor(message: string, public readonly input: string) {
       super(message)
     }
   }
   
   export function parseYaml(input: string): unknown {
     try {
       // ... 现有实现
     } catch (error) {
       throw new YamlParseError(
         error instanceof Error ? error.message : 'Unknown YAML parse error',
         input
       )
     }
   }
   ```

3. **多文档支持**
   ```typescript
   export function parseAllYaml(input: string): unknown[] {
     if (typeof Bun !== 'undefined') {
       // Bun 可能不支持，需要检查
       return Bun.YAML.parseAll?.(input) ?? [Bun.YAML.parse(input)]
     }
     return require('yaml').parseAllDocuments(input).map((doc: unknown) => 
       require('yaml').docToJS(doc)
     )
   }
   ```

4. **配置选项支持**
   ```typescript
   export interface ParseYamlOptions {
     // 是否允许重复键
     uniqueKeys?: boolean
     // 最大别名深度
     maxAliasDepth?: number
     // ... 其他选项
   }
   
   export function parseYaml(input: string, options?: ParseYamlOptions): unknown {
     // 传递选项到底层实现
   }
   ```

5. **性能优化**
   - 对于频繁解析的场景，考虑缓存解析器实例
   - 添加流式解析支持（如果底层实现支持）

6. **测试覆盖**
   - 在 Bun 和 Node.js 环境下分别测试
   - 测试各种 YAML 特性（锚点、别名、多行字符串等）
   - 测试错误处理
