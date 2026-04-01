# glob.ts 深度研究

## 场景与职责

本模块提供基于 ripgrep 的高性能文件 glob 匹配功能，用于替代 Node.js 原生的 glob 实现。主要服务于 `GlobTool` 和其他需要文件列表的场景，在大型代码库中提供显著的性能优势。

**核心场景：**
1. **GlobTool 文件搜索**：用户通过 glob 模式查找文件
2. **文件列表获取**：批量获取匹配特定模式的文件路径
3. **权限过滤**：自动应用文件读取权限规则，隐藏用户无权限访问的文件

## 功能点目的

### 1. 高性能文件匹配
- **目的**：在大型仓库中快速列出文件
- **实现**：利用 ripgrep 的 `--files` 和 `--glob` 选项
- **优势**：比 Node.js glob 库内存效率更高，速度更快

### 2. 绝对路径支持
- **目的**：支持 `/path/to/*.ts` 这类绝对路径模式
- **实现**：自动提取基础目录，转换为相对模式

### 3. 权限感知过滤
- **目的**：尊重用户的文件读取权限设置
- **实现**：将 deny 规则转换为 ripgrep 的排除模式

### 4. 分页支持
- **目的**：处理大量结果时避免内存溢出
- **实现**：支持 `limit` 和 `offset` 参数

## 具体技术实现

### 核心函数

#### extractGlobBaseDirectory(pattern: string)
从 glob 模式中提取静态基础目录：
```typescript
// 输入: "/home/user/src/*.ts"
// 输出: { baseDir: "/home/user/src", relativePattern: "*.ts" }

// 输入: "src/**/*.ts"
// 输出: { baseDir: "", relativePattern: "src/**/*.ts" }
```

**处理逻辑：**
1. 查找第一个 glob 特殊字符 (`*`, `?`, `[`, `{`)
2. 提取静态前缀
3. 找到最后一个路径分隔符
4. 处理根目录和 Windows 驱动器路径特殊情况

#### glob(filePattern, cwd, options, abortSignal, toolPermissionContext)
主 glob 函数：
```typescript
async function glob(
  filePattern: string,
  cwd: string,
  { limit, offset }: { limit: number; offset: number },
  abortSignal: AbortSignal,
  toolPermissionContext: ToolPermissionContext,
): Promise<{ files: string[]; truncated: boolean }>
```

**ripgrep 参数构建：**
```
rg --files                    # 列出文件而非搜索内容
   --glob <pattern>           # 匹配模式
   --sort=modified            # 按修改时间排序（最旧在前）
   --no-ignore                # 忽略 .gitignore（默认，可通过环境变量关闭）
   --hidden                   # 包含隐藏文件（默认，可通过环境变量关闭）
   --glob !<ignore-pattern>   # 排除权限规则中的 deny 模式
```

**环境变量控制：**
- `CLAUDE_CODE_GLOB_NO_IGNORE`: 控制 `--no-ignore`（默认 `true`）
- `CLAUDE_CODE_GLOB_HIDDEN`: 控制 `--hidden`（默认 `true`）

### 关键流程

```
1. 解析模式，提取基础目录和相对模式
2. 获取文件读取权限的 ignore 模式
3. 构建 ripgrep 参数
4. 添加插件缓存排除模式
5. 执行 ripgrep
6. 转换相对路径为绝对路径
7. 应用分页（limit/offset）
8. 返回结果和截断标志
```

## 关键代码路径与文件引用

### 本文件导出
| 导出 | 类型 | 用途 |
|------|------|------|
| `extractGlobBaseDirectory` | 函数 | 提取 glob 基础目录 |
| `glob` | 函数 | 执行 glob 匹配 |

### 调用方
1. **GlobTool.ts**: 主要调用方，处理用户文件搜索请求
2. **main.tsx**: 启动时的文件扫描

### 依赖模块
```typescript
import { basename, dirname, isAbsolute, join, sep } from 'path'
import type { ToolPermissionContext } from '../Tool.js'
import { isEnvTruthy } from './envUtils.js'
import { getFileReadIgnorePatterns, normalizePatternsToPath } from './permissions/filesystem.js'
import { getPlatform } from './platform.js'
import { getGlobExclusionsForPluginCache } from './plugins/orphanedPluginFilter.js'
import { ripGrep } from './ripgrep.js'
```

## 依赖与外部交互

### 上游依赖

1. **ripgrep.ts**: 核心搜索能力
   - `ripGrep()`: 执行 ripgrep 命令
   - 处理超时、错误重试、EAGAIN 恢复

2. **permissions/filesystem.ts**: 权限规则
   - `getFileReadIgnorePatterns()`: 获取 deny 规则模式
   - `normalizePatternsToPath()`: 规范化模式路径

3. **plugins/orphanedPluginFilter.ts**: 插件缓存排除
   - `getGlobExclusionsForPluginCache()`: 获取孤儿插件目录排除模式

### 配置与常量

**默认行为：**
- 默认忽略 `.gitignore`（`--no-ignore`）
- 默认包含隐藏文件（`--hidden`）
- 按修改时间排序（最旧在前）

**超时设置：**
- 默认 20 秒（WSL 60 秒）
- 可通过 `CLAUDE_CODE_GLOB_TIMEOUT_SECONDS` 覆盖

## 风险、边界与改进建议

### 已知风险

1. **ripgrep 不可用**
   - 风险：ripgrep 未安装或损坏时功能失效
   - 缓解：`ripgrep.ts` 有可用性检测和回退机制

2. **大仓库内存压力**
   - 风险：超大型仓库可能返回数万文件路径
   - 缓解：分页支持（limit/offset），ripgrep 20MB 缓冲区限制

3. **权限规则绕过**
   - 风险：复杂的符号链接场景可能绕过权限检查
   - 缓解：路径在权限系统中进行二次验证

4. **排序不一致**
   - 风险：`--sort=modified` 在不同平台行为可能略有差异
   - 影响：分页结果在边界处可能不稳定

### 边界情况

1. **空模式**：返回空数组
2. **不存在的目录**：ripgrep 返回空结果
3. **无权限目录**：ripgrep 静默跳过
4. **循环符号链接**：ripgrep 自动检测并跳过
5. **Windows 路径**：自动处理驱动器根路径（`C:` → `C:\`）

### 改进建议

1. **智能缓存**
   - 建议：缓存文件列表，监听文件系统变化
   - 收益：重复查询时延迟显著降低

2. **并行查询优化**
   - 建议：多个 glob 查询合并为一次 ripgrep 调用
   - 实现：使用 `--glob` 多实例和分组输出

3. **模糊匹配支持**
   - 建议：支持 `**/*.{ts,tsx}` 这类扩展名集合
   - 现状：已支持，但可优化性能

4. **实时结果流**
   - 建议：支持流式返回结果，而非等待全部完成
   - 场景：大型仓库中快速显示首批结果

5. **忽略文件分层**
   - 建议：支持项目级 `.claudeignore` 文件
   - 场景：项目特定的排除规则

### 性能优化点

1. **ripgrep 模式优化**
   - 当前：`--sort=modified` 需要收集全部结果后排序
   - 优化：如无需排序，可移除该选项提升速度

2. **权限规则预处理**
   - 当前：每次查询都重新计算 ignore 模式
   - 优化：缓存权限规则的 glob 表示

3. **目录遍历限制**
   - 建议：添加最大深度限制选项
   - 场景：防止 `**/*` 在超深目录树中耗时过长
