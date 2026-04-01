# parseMarketplaceInput.ts 深度研究文档

## 场景与职责

`parseMarketplaceInput.ts` 实现了**市场源输入解析**功能。它处理用户通过各种格式输入的市场源（如 GitHub 简写、SSH URL、HTTP URL、本地路径等），将其标准化为 `MarketplaceSource` 对象。

### 核心职责

1. **多格式输入解析**：支持 Git SSH、HTTP/HTTPS、GitHub 简写、本地路径等多种格式
2. **智能类型推断**：根据输入特征推断合适的市场源类型
3. **引用支持**：支持 `#ref` 或 `@ref` 指定分支/标签
4. **路径验证**：本地路径验证文件/目录存在性和类型
5. **错误反馈**：为无效输入提供清晰的错误信息

### 在系统架构中的位置

```
CLI 层
    └── cli/handlers/plugins.ts
        └── parseMarketplaceInput()  ← 调用本模块

命令层
    └── commands/plugin/AddMarketplace.tsx
        └── parseMarketplaceInput()  ← 调用本模块
```

---

## 功能点目的

### 1. 支持的输入格式

| 格式 | 示例 | 输出类型 |
|------|------|----------|
| Git SSH | `git@github.com:owner/repo.git` | `git` |
| Git SSH (带 ref) | `git@github.com:owner/repo.git#main` | `git` + ref |
| HTTP/HTTPS | `https://example.com/marketplace.json` | `url` 或 `git` |
| GitHub 简写 | `owner/repo` | `github` |
| GitHub 简写 (带 ref) | `owner/repo#v1.0` | `github` + ref |
| 本地 JSON 文件 | `./marketplace.json` | `file` |
| 本地目录 | `./my-marketplace` | `directory` |
| 家目录路径 | `~/marketplace.json` | `file` |

### 2. 特殊处理逻辑

#### Git URL 检测（HTTP/HTTPS）

```typescript
// 以 .git 结尾 或包含 /_git/ → 视为 git 仓库
if (url.endsWith('.git') || url.includes('/_git/')) {
  return { source: 'git', url, ref }
}
```

这是为了解决 Azure DevOps 的特殊情况：ADO 使用 `/_git/` 路径且不能加 `.git` 后缀。

#### GitHub URL 转换

```typescript
// github.com/owner/repo → 转换为 git 类型
if (url.hostname === 'github.com') {
  return { source: 'git', url: url + '.git', ref }
}
```

用户显式提供 GitHub HTTPS URL 时，转换为 git 类型以便克隆。

### 3. 引用解析

支持两种引用分隔符：
- `#ref`：URL fragment 风格（`owner/repo#main`）
- `@ref`：显示风格（`owner/repo@main`）

后者用于错误消息和托管设置，用户可能复制粘贴。

---

## 具体技术实现

### 核心函数

```typescript
export async function parseMarketplaceInput(
  input: string
): Promise<MarketplaceSource | { error: string } | null>
```

返回三种可能：
- `MarketplaceSource`：成功解析
- `{ error: string }`：输入无效，返回错误信息
- `null`：无法识别的格式

### 解析流程

```
parseMarketplaceInput(input)
    │
    ├── 1. Git SSH URL
    │   └── 正则: /^([a-zA-Z0-9._-]+@[^:]+:.+?(?:\.git)?)(#(.+))?$/
    │   └── 匹配 → { source: 'git', url, ref? }
    │
    ├── 2. HTTP/HTTPS URL
    │   ├── 提取 fragment (#ref)
    │   ├── 检查 .git 后缀或 /_git/ → git 类型
    │   ├── GitHub 域名 → 转换为 git 类型 + .git
    │   └── 其他 → url 类型
    │
    ├── 3. 本地路径
    │   ├── 识别前缀: ./, ../, /, ~, Windows 路径
    │   ├── 解析 ~ 为 homedir()
    │   ├── stat 路径
    │   │   ├── 文件 + .json 后缀 → file 类型
    │   │   ├── 目录 → directory 类型
    │   │   └── 其他 → 错误
    │   └── 不存在/无法访问 → 错误
    │
    ├── 4. GitHub 简写
    │   ├── 包含 / 且不包含 :
    │   ├── 提取 #ref 或 @ref
    │   └── → { source: 'github', repo, ref? }
    │
    └── 5. 无法识别
        └── return null
```

### 正则表达式详解

#### Git SSH 匹配

```typescript
/^([a-zA-Z0-9._-]+@[^:]+:.+?(?:\.git)?)(#(.+))?$/
```

| 部分 | 说明 |
|------|------|
| `[a-zA-Z0-9._-]+` | 用户名（支持字母、数字、点、下划线、连字符） |
| `@` | 分隔符 |
| `[^:]+` | 主机名（非冒号字符） |
| `:` | SSH URL 分隔符 |
| `.+?(?:\.git)?` | 路径，可选 .git 后缀 |
| `(#(.+))?` | 可选的 #ref |

支持多种 SSH 格式：
- 标准：`git@github.com:owner/repo.git`
- GitHub Enterprise 证书：`org-123456@github.com:owner/repo.git`
- 自定义用户名：`deploy@gitlab.com:group/project.git`
- IP 地址：`user@192.168.10.123:path/to/repo`

#### Windows 路径检测

```typescript
const isWindowsPath = isWindows && (
  trimmed.startsWith('.\\') ||
  trimmed.startsWith('..\\') ||
  /^[a-zA-Z]:[/\\]/.test(trimmed)  // C:\ 或 C:/
)
```

### 错误处理

| 场景 | 错误信息 |
|------|----------|
| 路径不存在 | `Path does not exist: {path}` |
| 路径无法访问 | `Cannot access path: {path} ({code})` |
| 文件非 JSON | `File path must point to a .json file...` |
| 路径类型未知 | `Path is neither a file nor a directory...` |

---

## 关键代码路径与文件引用

### 导出函数

| 函数 | 用途 | 调用方 |
|------|------|--------|
| `parseMarketplaceInput` | 解析市场源输入 | `cli/handlers/plugins.ts`, `AddMarketplace.tsx` |

### 调用关系图

```
cli/handlers/plugins.ts
    └── addMarketplaceCommand()
        └── parseMarketplaceInput(input)
            └── 根据结果执行不同操作
                ├── MarketplaceSource → 添加市场
                ├── { error } → 显示错误
                └── null → 显示"无法识别"

commands/plugin/AddMarketplace.tsx
    └── handleSubmit()
        └── parseMarketplaceInput(input)
            └── 类似处理
```

---

## 依赖与外部交互

### 直接依赖

| 模块 | 用途 |
|------|------|
| `utils/fsOperations.ts` | 文件系统抽象 (`getFsImplementation`) |
| `utils/errors.ts` | 错误码提取 (`getErrnoCode`) |
| `./schemas.ts` | `MarketplaceSource` 类型 |

### Node.js 内置模块

| 模块 | 用途 |
|------|------|
| `os.homedir` | 展开 `~` 为家目录 |
| `path.resolve` | 解析相对路径为绝对路径 |

---

## 风险、边界与改进建议

### 已知风险

1. **路径遍历攻击**
   - 风险：输入 `../../../etc/passwd` 可能访问敏感文件
   - 缓解：使用 `resolve()` 规范化路径，后续操作在目标目录内进行

2. **符号链接遍历**
   - 风险：本地路径可能是符号链接，指向意外位置
   - 现状：未显式处理，依赖后续操作的安全检查

3. **URL 解析错误**
   - 风险：`new URL()` 可能抛出，但已在 try/catch 中处理

4. **Windows 路径误判**
   - 风险：Unix 文件名可能包含反斜杠（虽然罕见）
   - 缓解：仅在 `process.platform === 'win32'` 时启用 Windows 路径检测

### 边界情况

| 场景 | 行为 |
|------|------|
| 空字符串 | 无法识别，返回 null |
| 仅空白字符 | trim 后为空，返回 null |
| 包含换行符 | 正常处理（trim 去除） |
| 路径存在但无权限 | 返回访问错误 |
| 损坏的符号链接 | stat 失败，返回错误 |
| GitHub 简写包含冒号 | 返回 null（避免与 SSH 冲突） |
| 以 @ 开头 | 返回 null（可能是 @mention） |
| NPM 包名 | 返回 null（尚未实现） |

### 改进建议

1. **NPM 包支持**
   - 建议：实现 `source: 'npm'` 类型支持

2. **路径验证增强**
   - 建议：验证本地路径在市场允许列表中

3. **模糊匹配建议**
   - 建议：无法识别时提供相似格式建议

4. **历史记录**
   - 建议：记录成功解析的输入，提供自动完成

5. **交互式确认**
   - 建议：解析成功后显示确认信息（如 "将添加 GitHub 仓库 owner/repo"）

---

## 测试要点

1. **格式覆盖**：测试所有支持的输入格式
2. **引用解析**：验证 `#ref` 和 `@ref` 的正确提取
3. **错误场景**：验证各种无效输入的错误信息
4. **平台差异**：验证 Windows 和 Unix 的路径处理
5. **边界情况**：测试空字符串、特殊字符、长路径等
