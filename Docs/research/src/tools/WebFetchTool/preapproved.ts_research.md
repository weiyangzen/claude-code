# WebFetchTool/preapproved.ts 研究文档

## 场景与职责

preapproved.ts 是 WebFetchTool 的**预批准域名管理模块**，定义了 WebFetch 工具无需用户明确授权即可访问的域名白名单。该模块的核心职责：

1. **维护预批准域名列表** - 包含编程语言文档、框架官网、技术文档等可信来源
2. **提供域名匹配功能** - 高效检查给定域名是否在白名单中
3. **支持路径级精确控制** - 某些域名仅允许特定路径前缀（如 github.com/anthropics）

**安全边界**：该白名单**仅适用于 WebFetch 工具**（GET 请求），沙箱系统的网络限制不会继承此列表，防止数据外泄风险。

## 功能点目的

### 1. PREAPPROVED_HOSTS - 预批准域名集合

**目的**：定义允许 WebFetch 直接访问的域名列表，无需用户每次确认。

**域名分类**（共 130+ 个域名）：

| 类别 | 示例域名 | 数量 |
|------|----------|------|
| Anthropic 相关 | platform.claude.com, modelcontextprotocol.io | 5 |
| 编程语言文档 | docs.python.org, go.dev, doc.rust-lang.org | 12 |
| Web/JS 框架 | react.dev, vuejs.org, nextjs.org, tailwindcss.com | 15 |
| Python 生态 | docs.djangoproject.com, fastapi.tiangolo.com, pytorch.org | 11 |
| PHP 框架 | laravel.com, symfony.com, wordpress.org | 3 |
| Java 生态 | docs.spring.io, hibernate.org, gradle.org | 5 |
| .NET/C# | asp.net, dotnet.microsoft.com, nuget.org | 4 |
| 移动开发 | reactnative.dev, docs.flutter.dev, developer.apple.com | 4 |
| 数据科学/ML | keras.io, huggingface.co, www.kaggle.com | 4 |
| 数据库 | redis.io, www.postgresql.org, prisma.io | 6 |
| 云/DevOps | docs.aws.amazon.com, kubernetes.io, www.docker.com | 9 |
| 测试/监控 | cypress.io, selenium.dev | 2 |
| 游戏开发 | docs.unity.com, docs.unrealengine.com | 2 |
| 其他工具 | git-scm.com, nginx.org | 2 |

**代码位置**：行 14-131

### 2. isPreapprovedHost - 域名匹配函数

**目的**：高效检查给定 hostname 和 pathname 是否匹配预批准列表。

**实现优化**：
- 模块加载时预先将域名分为两类：
  - `HOSTNAME_ONLY`：纯域名（Set 结构，O(1) 查找）
  - `PATH_PREFIXES`：带路径前缀的域名（Map<hostname, path[]>）
- 路径匹配时强制执行路径段边界检查（防止 `/anthropics` 匹配 `/anthropics-evil`）

**代码位置**：行 136-166

## 具体技术实现

### 数据结构

```typescript
// 原始域名集合（包含路径前缀格式的条目）
export const PREAPPROVED_HOSTS = new Set([
  'docs.python.org',
  'github.com/anthropics',  // 带路径前缀
  // ...
]);

// 模块加载时拆分后的结构
const {
  HOSTNAME_ONLY: Set<string>,      // 纯域名集合
  PATH_PREFIXES: Map<string, string[]>  // 域名 -> 路径前缀数组
}
```

### 路径前缀处理逻辑

```typescript
// 模块加载时执行的一次性拆分
const { HOSTNAME_ONLY, PATH_PREFIXES } = (() => {
  const hosts = new Set<string>();
  const paths = new Map<string, string[]>();
  
  for (const entry of PREAPPROVED_HOSTS) {
    const slash = entry.indexOf('/');
    if (slash === -1) {
      // 纯域名，如 "docs.python.org"
      hosts.add(entry);
    } else {
      // 带路径，如 "github.com/anthropics"
      const host = entry.slice(0, slash);      // "github.com"
      const path = entry.slice(slash);          // "/anthropics"
      // 添加到该主机的路径前缀列表
      const prefixes = paths.get(host);
      if (prefixes) prefixes.push(path);
      else paths.set(host, [path]);
    }
  }
  return { HOSTNAME_ONLY: hosts, PATH_PREFIXES: paths };
})();
```

### 匹配算法

```typescript
export function isPreapprovedHost(hostname: string, pathname: string): boolean {
  // 1. 快速检查：纯域名匹配（O(1)）
  if (HOSTNAME_ONLY.has(hostname)) return true;
  
  // 2. 路径前缀检查
  const prefixes = PATH_PREFIXES.get(hostname);
  if (prefixes) {
    for (const p of prefixes) {
      // 严格路径段边界检查：
      // - 完全匹配: pathname === p
      // - 子路径匹配: pathname.startsWith(p + '/')
      // 防止 "/anthropics" 匹配 "/anthropics-evil/malware"
      if (pathname === p || pathname.startsWith(p + '/')) return true;
    }
  }
  return false;
}
```

## 关键代码路径与文件引用

### 导出内容

```typescript
// 行 14-131
export const PREAPPROVED_HOSTS: Set<string>

// 行 154-166
export function isPreapprovedHost(hostname: string, pathname: string): boolean
```

### 被调用方

| 调用方 | 文件路径 | 用途 |
|--------|----------|------|
| WebFetchTool.ts | ./WebFetchTool.ts | checkPermissions 中检查预批准域名 |
| utils.ts | ./utils.ts | isPreapprovedUrl 包装函数 |

## 依赖与外部交互

### 导入依赖

```typescript
// 无外部导入 - 纯数据模块
```

### 外部交互

- **无网络请求**：纯静态数据模块
- **无文件系统操作**：所有数据内联在代码中

## 风险、边界与改进建议

### 安全风险

1. **预批准域名的安全性**
   - **风险**：如果预批准域名被攻破或包含恶意内容，用户可能获取到危险信息
   - **缓解**：列表仅限于知名的技术文档和工具网站，由维护团队审核
   - **缓解**：路径前缀限制（如 github.com/anthropics）减少攻击面

2. **沙箱隔离**
   - **重要**：代码注释明确说明沙箱系统**不会**继承此列表
   - **原因**：防止任意网络访问（POST、上传等）导致数据外泄
   - **验证**：test/utils/sandbox/webfetch-preapproved-separation.test.ts 验证此隔离

3. **路径边界检查**
   - 已实现：严格检查路径段边界，防止前缀匹配攻击
   - 示例：`/anthropics` 不会匹配 `/anthropics-evil/malware`

### 边界情况

| 场景 | 处理 |
|------|------|
| 子域名 | 需要显式添加（如 'docs.python.org' 和 'www.python.org' 是不同条目）|
| 带端口的 hostname | 当前实现会不匹配（需确保传入的 hostname 已去除端口）|
| 大小写 | hostname 通常为小写，pathname 匹配需确保传入格式正确 |
| 路径编码 | 需确保 pathname 已解码（如 `%2F` → `/`）后再匹配 |

### 改进建议

1. **域名分类管理**
   - 当前所有域名在一个大 Set 中，可考虑按类别分组管理
   - 便于生成文档和审核

2. **动态更新机制**
   - 当前需要代码更新才能添加新域名
   - 可考虑从远程配置加载，但需权衡安全性和灵活性

3. **过期域名检测**
   - 建议定期扫描列表中的域名可用性
   - 移除不再使用或已下线的域名

4. **通配符支持**
   - 当前不支持通配符（如 `*.readthedocs.io`）
   - 如需支持需仔细评估安全风险

5. **文档生成**
   - 可从代码自动生成预批准域名文档
   - 帮助用户了解哪些网站可以直接访问

### 新增域名流程

如需添加新域名到预批准列表：

1. 评估域名的可信度和安全性
2. 如果是路径级授权，确保格式为 `hostname/path`
3. 添加到 `PREAPPROVED_HOSTS` Set 中
4. 添加注释说明域名用途
5. 考虑是否需要同步更新沙箱隔离测试

### 测试建议

- 未发现针对 preapproved.ts 的专门测试
- 建议添加：
  - `isPreapprovedHost` 的各种边界情况测试
  - 路径前缀边界检查测试（防止匹配攻击）
  - 性能测试（确保 O(1) 查找性能）
