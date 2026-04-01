# generatedFiles.ts 深度研究文档

## 场景与职责

`generatedFiles.ts` 提供基于 GitHub Linguist 风格的生成文件检测功能，用于：

1. **提交归因过滤**：在计算代码贡献统计时排除生成的文件
2. **代码审查优化**：减少不必要的生成文件差异显示
3. **语言统计准确性**：确保 Linguist 风格的生成文件不被计入项目统计

该模块是 `commitAttribution.ts` 的依赖，用于过滤不应计入用户贡献的文件。

## 功能点目的

### 1. 生成文件检测
基于多维度规则检测生成文件：
- **精确文件名匹配**：如 `package-lock.json`, `yarn.lock` 等
- **扩展名匹配**：如 `.lock`, `.min.js`, `.d.ts` 等
- **目录模式匹配**：如 `node_modules/`, `dist/`, `build/` 等
- **正则表达式模式**：如 `*.min.*`, `*.generated.*`, `*_pb2.py` 等

### 2. 文件列表过滤
提供批量过滤功能，从文件列表中移除生成文件。

## 具体技术实现

### 检测规则

#### 精确文件名（大小写不敏感）
```typescript
const EXCLUDED_FILENAMES = new Set([
  'package-lock.json', 'yarn.lock', 'pnpm-lock.yaml',
  'bun.lockb', 'bun.lock', 'composer.lock', 'gemfile.lock',
  'cargo.lock', 'poetry.lock', 'pipfile.lock',
  'shrinkwrap.json', 'npm-shrinkwrap.json',
])
```

#### 扩展名模式
```typescript
const EXCLUDED_EXTENSIONS = new Set([
  '.lock', '.min.js', '.min.css', '.min.html',
  '.bundle.js', '.bundle.css', '.generated.ts',
  '.generated.js', '.d.ts',
])
```

#### 目录模式
```typescript
const EXCLUDED_DIRECTORIES = [
  '/dist/', '/build/', '/out/', '/output/',
  '/node_modules/', '/vendor/', '/vendored/',
  '/third_party/', '/third-party/', '/external/',
  '/.next/', '/.nuxt/', '/.svelte-kit/',
  '/coverage/', '/__pycache__/', '/.tox/',
  '/venv/', '/.venv/', '/target/release/', '/target/debug/',
]
```

#### 正则表达式模式
```typescript
const EXCLUDED_FILENAME_PATTERNS = [
  /^.*\.min\.[a-z]+$/i,      // *.min.*
  /^.*\.generated\.[a-z]+$/i, // *.generated.*
  /^.*\.pb\.(go|js|ts|py|rb)$/i,  // Protocol buffer
  /^.*_pb2?\.py$/i,          // Python protobuf
  /^.*\.grpc\.[a-z]+$/i,     // gRPC
  /^.*\.swagger\.[a-z]+$/i,  // Swagger
  /^.*\.openapi\.[a-z]+$/i,  // OpenAPI
]
```

### 检测算法

```typescript
export function isGeneratedFile(filePath: string): boolean {
  // 1. 规范化路径分隔符为 POSIX 风格
  const normalizedPath = posix.sep + filePath.split(sep).join(posix.sep).replace(/^\/+/, '')
  const fileName = basename(filePath).toLowerCase()
  const ext = extname(filePath).toLowerCase()

  // 2. 检查精确文件名
  if (EXCLUDED_FILENAMES.has(fileName)) return true

  // 3. 检查扩展名
  if (EXCLUDED_EXTENSIONS.has(ext)) return true

  // 4. 检查复合扩展名（如 .min.js）
  const parts = fileName.split('.')
  if (parts.length > 2) {
    const compoundExt = '.' + parts.slice(-2).join('.')
    if (EXCLUDED_EXTENSIONS.has(compoundExt)) return true
  }

  // 5. 检查目录模式
  for (const dir of EXCLUDED_DIRECTORIES) {
    if (normalizedPath.includes(dir)) return true
  }

  // 6. 检查正则表达式模式
  for (const pattern of EXCLUDED_FILENAME_PATTERNS) {
    if (pattern.test(fileName)) return true
  }

  return false
}
```

## 关键代码路径与文件引用

### 核心导出
- `isGeneratedFile(filePath: string): boolean` - 检测单个文件是否为生成文件
- `filterGeneratedFiles(files: string[]): string[]` - 过滤文件列表

### 依赖关系

**被以下模块导入**：
- `src/utils/commitAttribution.ts` - 提交归因计算

**依赖的模块**：
- 仅依赖 Node.js 内置 `path` 模块

### 文件位置
- 源码：`src/utils/generatedFiles.ts` (136 行)

## 依赖与外部交互

### Node.js 内置模块
- `path` - `basename`, `extname`, `posix`, `sep`

### 项目内部依赖
- 无

### 外部依赖
- 无

## 风险、边界与改进建议

### 已知风险

1. **误报风险**：某些合法文件可能匹配生成文件模式（如 `data.min.json`）
2. **遗漏风险**：新的生成文件类型可能未被覆盖
3. **性能考虑**：正则表达式匹配在大量文件时可能有性能影响

### 边界情况

1. **大小写敏感**：所有文件名比较都转换为小写
2. **路径分隔符**：统一转换为 POSIX 风格（`/`）进行目录匹配
3. **复合扩展名**：支持 `.min.js` 等多部分扩展名

### 改进建议

1. **配置扩展**：允许用户通过 `.claude/settings.json` 添加自定义排除模式
2. **缓存优化**：对频繁检测的文件路径添加缓存
3. **规则优先级**：支持规则的优先级和覆盖机制
4. **测试覆盖**：当前没有专门的测试文件，建议添加单元测试
5. **Linguist 同步**：定期同步 GitHub Linguist 的规则更新
6. **性能优化**：对于已知的大型目录（如 `node_modules`），提前短路返回

### 与 GitHub Linguist 的关系

该模块的规则基于 GitHub Linguist 的 vendored 和 generated 文件模式，但进行了简化：
- 移除了 Linguist 的启发式内容检测
- 专注于文件名和路径模式匹配
- 添加了前端/Node.js 生态特定的模式（`.next/`, `.nuxt/` 等）
