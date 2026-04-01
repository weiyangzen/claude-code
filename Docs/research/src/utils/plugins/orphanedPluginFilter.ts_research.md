# orphanedPluginFilter.ts 深度研究文档

## 场景与职责

`orphanedPluginFilter.ts` 提供了**孤儿插件版本过滤**功能。当插件版本更新时，旧版本会被标记为孤儿（通过 `.orphaned_at` 文件），但保留 7 天以防并发会话仍在使用。在此期间，Grep/Glob 工具需要排除这些孤儿目录，避免返回过时代码。

### 核心职责

1. **检测孤儿标记**：通过 ripgrep 查找所有 `.orphaned_at` 文件
2. **生成排除模式**：将孤儿目录转换为 ripgrep 的 `--glob` 排除模式
3. **会话级缓存**：缓存排除列表，避免重复扫描
4. **路径重叠检测**：仅在搜索路径与插件缓存目录重叠时返回排除模式

### 在系统架构中的位置

```
工具层
    ├── GrepTool.ts
    │   └── getGlobExclusionsForPluginCache()  ← 调用本模块
    ├── glob.ts
    │   └── getGlobExclusionsForPluginCache()  ← 调用本模块
    └── orphanedPluginFilter.ts                # ← 本文件

启动流程
    └── main.tsx
        └── 孤儿清理后预热缓存
```

---

## 功能点目的

### 1. 孤儿版本生命周期

```
插件更新
    │
    ├── 新版本安装
    ├── 旧版本标记 .orphaned_at（包含时间戳）
    ├── 7天保留期（并发会话安全窗口）
    ├── 7天后清理删除
    └── 在此期间：Grep/Glob 需要排除
```

### 2. 排除模式生成

将孤儿目录转换为 ripgrep glob 模式：

```
插件缓存结构：
~/.claude/plugins/cache/
    └── {marketplace}/
        └── {plugin}/
            └── {version}/
                └── .orphaned_at  ← 标记文件

排除模式：
!**/{marketplace}/{plugin}/{version}/**
```

### 3. 缓存策略

- **会话级缓存**：`cachedExclusions` 变量，进程内共享
- **预热机制**：`main.tsx` 在孤儿清理后调用，提前填充缓存
- **手动清除**：`clearPluginCacheExclusions()` 供 `/reload-plugins` 使用

---

## 具体技术实现

### 关键常量

```typescript
const ORPHANED_AT_FILENAME = '.orphaned_at'  // 孤儿标记文件名
```

### 核心算法

#### 1. 获取排除模式 (`getGlobExclusionsForPluginCache`)

```typescript
async function getGlobExclusionsForPluginCache(searchPath?: string): Promise<string[]> {
  const cachePath = normalize(join(getPluginsDirectory(), 'cache'))
  
  // 1. 路径重叠检测
  if (searchPath && !pathsOverlap(searchPath, cachePath)) {
    return []  // 搜索路径与插件缓存无关，无需排除
  }
  
  // 2. 检查缓存
  if (cachedExclusions !== null) {
    return cachedExclusions
  }
  
  // 3. 扫描孤儿标记
  try {
    const markers = await ripGrep([
      '--files',
      '--hidden',
      '--no-ignore',
      '--max-depth', '4',  // cache/<marketplace>/<plugin>/<version>/.orphaned_at
      '--glob', ORPHANED_AT_FILENAME,
    ], cachePath, signal)
    
    // 4. 生成排除模式
    cachedExclusions = markers.map(markerPath => {
      const versionDir = dirname(markerPath)
      const rel = isAbsolute(versionDir) 
        ? relative(cachePath, versionDir) 
        : versionDir
      const posixRelative = rel.replace(/\\/g, '/')
      return `!**/${posixRelative}/**`
    })
    
    return cachedExclusions
  } catch {
    // 5. 失败回退：返回空数组，不阻断搜索
    cachedExclusions = []
    return cachedExclusions
  }
}
```

#### 2. 路径重叠检测 (`pathsOverlap`)

```typescript
function pathsOverlap(a: string, b: string): boolean {
  const na = normalizeForCompare(a)
  const nb = normalizeForCompare(b)
  return (
    na === nb ||           // 完全相同
    na === sep ||          // 一个是根目录
    nb === sep ||
    na.startsWith(nb + sep) ||  // a 在 b 下
    nb.startsWith(na + sep)     // b 在 a 下
  )
}

function normalizeForCompare(p: string): string {
  const n = normalize(p)
  return process.platform === 'win32' ? n.toLowerCase() : n
}
```

### ripgrep 参数详解

| 参数 | 用途 |
|------|------|
| `--files` | 只输出文件路径，不搜索内容 |
| `--hidden` | 包含隐藏文件（`.` 开头） |
| `--no-ignore` | 忽略 `.gitignore` 等忽略文件 |
| `--max-depth 4` | 限制递归深度：cache/1/2/3/4/.orphaned_at |
| `--glob .orphaned_at` | 只匹配孤儿标记文件 |

---

## 关键代码路径与文件引用

### 导出函数

| 函数 | 用途 | 调用方 |
|------|------|--------|
| `getGlobExclusionsForPluginCache` | 获取排除模式 | `GrepTool.ts`, `glob.ts` |
| `clearPluginCacheExclusions` | 清除缓存 | `refresh.ts`, `/reload-plugins` |

### 调用关系图

```
main.tsx
    └── 启动流程
        └── cleanupOrphanedPluginVersionsInBackground()  # cacheUtils.ts
            └── （完成后）getGlobExclusionsForPluginCache()  # 预热缓存

tools/GrepTool/GrepTool.ts
    └── execute()
        └── getGlobExclusionsForPluginCache(searchPath)
            └── 添加到 ripgrep --glob 参数

utils/glob.ts
    └── globWithRipgrep()
        └── getGlobExclusionsForPluginCache(searchPath)
            └── 添加到 ripgrep --glob 参数

utils/plugins/refresh.ts
    └── clearPluginCaches()
        └── clearPluginCacheExclusions()  # 清除缓存
```

---

## 依赖与外部交互

### 直接依赖

| 模块 | 用途 |
|------|------|
| `utils/ripgrep.ts` | ripgrep 调用 (`ripGrep`) |
| `utils/plugins/pluginDirectories.ts` | 获取插件目录 (`getPluginsDirectory`) |

### 路径结构依赖

```
{pluginsDirectory}/
    └── cache/                    # 插件缓存根
        └── {marketplace}/        # 市场名称
            └── {plugin}/         # 插件名称
                └── {version}/    # 版本号
                    └── .orphaned_at  # 孤儿标记
```

### 相关模块

| 模块 | 关系 |
|------|------|
| `cacheUtils.ts` | 创建 `.orphaned_at` 标记、清理孤儿版本 |
| `refresh.ts` | 调用 `clearPluginCacheExclusions()` |

---

## 风险、边界与改进建议

### 已知风险

1. **ripgrep 失败**
   - 风险：ripgrep 不可用或执行失败
   - 缓解：try/catch 包裹，失败返回空数组，不阻断搜索

2. **缓存过期**
   - 风险：会话期间孤儿版本被清理，缓存未更新
   - 缓解：缓存只在会话开始时预热，后续磁盘变化不影响；`/reload-plugins` 可清除缓存

3. **路径大小写（Windows）**
   - 风险：Windows 路径大小写不敏感，可能导致重叠检测失败
   - 缓解：`normalizeForCompare` 在 Windows 上转小写

4. **深度限制**
   - 风险：如果目录结构超过 4 层，可能漏检
   - 现状：当前结构固定为 4 层，足够使用

### 边界情况

| 场景 | 行为 |
|------|------|
| 搜索路径在插件缓存外 | 返回空数组，不添加排除 |
| 缓存已填充 | 直接返回缓存结果 |
| 无孤儿版本 | 返回空数组 |
| ripgrep 失败 | 返回空数组，记录错误 |
| Windows 路径 | 统一转小写后比较 |
| 根目录搜索 | 正确处理（`na === sep`） |

### 改进建议

1. **缓存失效策略**
   - 当前：缓存永久有效，直到 `/reload-plugins`
   - 建议：添加时间戳，定期刷新或监听文件系统事件

2. **增量更新**
   - 当前：全量扫描所有 `.orphaned_at`
   - 建议：维护索引文件，记录孤儿版本列表

3. **性能优化**
   - 当前：每个 Grep/Glob 调用都检查缓存
   - 建议：将排除模式直接集成到 ripgrep 配置，减少重复计算

4. **监控统计**
   - 建议：添加孤儿版本数量、排除模式数量的遥测

---

## 测试要点

1. **路径重叠检测**：验证各种路径组合的检测结果
2. **排除模式生成**：验证 glob 模式的正确性
3. **缓存行为**：验证缓存命中、清除的正确性
4. **错误恢复**：验证 ripgrep 失败时的回退行为
5. **平台兼容**：验证 Windows 大小写处理
