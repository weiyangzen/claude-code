# it2Setup.ts 深度研究文档

## 场景与职责

it2Setup.ts 是 **iTerm2 it2 CLI 工具的安装和配置管理模块**，负责检测、安装和验证 it2 工具。当用户在 iTerm2 中运行 Claude 但未安装 it2 时，此模块提供自动安装和设置引导功能。

**核心定位：**
- it2 工具的生命周期管理（检测、安装、验证）
- 多包管理器支持（uvx、pipx、pip）
- 用户偏好持久化（tmux vs iTerm2 偏好）
- 设置完成状态追踪

**使用场景：**
1. 首次在 iTerm2 中运行 Claude，提示安装 it2
2. 用户选择安装 it2，自动检测并使用最佳包管理器
3. 安装后验证 Python API 是否启用
4. 用户偏好持久化（记住选择 tmux 而非 iTerm2）

---

## 功能点目的

### 1. 包管理器检测
- `detectPythonPackageManager()`: 检测系统上可用的 Python 包管理器
- 优先级：uvx > pipx > pip

### 2. it2 安装
- `isIt2CliAvailable()`: 检测 it2 是否已安装
- `installIt2()`: 使用指定的包管理器安装 it2
- 安全考虑：从 home 目录运行安装，避免读取项目级配置

### 3. 安装验证
- `verifyIt2Setup()`: 验证 it2 是否能与 iTerm2 通信
- 检测 Python API 是否启用
- 提供启用指导

### 4. 用户偏好管理
- `markIt2SetupComplete()`: 标记设置完成，避免重复提示
- `setPreferTmuxOverIterm2()`: 设置 tmux 偏好
- `getPreferTmuxOverIterm2()`: 获取 tmux 偏好

---

## 具体技术实现

### 关键数据结构

```typescript
// 包管理器类型（按优先级排序）
type PythonPackageManager = 'uvx' | 'pipx' | 'pip'

// 安装结果
interface It2InstallResult {
  success: boolean
  error?: string
  packageManager?: PythonPackageManager
}

// 验证结果
interface It2VerifyResult {
  success: boolean
  error?: string
  needsPythonApiEnabled?: boolean  // 特定错误标识
}
```

### 核心流程

#### 1. 包管理器检测（detectPythonPackageManager）

```
1. 检查 uv（which uv）
   - 存在则返回 'uvx'
2. 检查 pipx（which pipx）
   - 存在则返回 'pipx'
3. 检查 pip（which pip）
   - 存在则返回 'pip'
4. 检查 pip3（which pip3）
   - 存在则返回 'pip'
5. 都不存在则返回 null
```

**优先级逻辑：**

```typescript
// uv 优先（现代、快速、隔离环境）
const uvResult = await execFileNoThrow('which', ['uv'])
if (uvResult.code === 0) return 'uvx'

// pipx 次之（用户级隔离环境）
const pipxResult = await execFileNoThrow('which', ['pipx'])
if (pipxResult.code === 0) return 'pipx'

// pip 最后（系统级，可能污染）
const pipResult = await execFileNoThrow('which', ['pip'])
if (pipResult.code === 0) return 'pip'
```

#### 2. it2 安装（installIt2）

```
1. 根据 packageManager 选择安装命令：
   
   uvx: uv tool install it2
   pipx: pipx install it2
   pip: pip install --user it2（失败则尝试 pip3）

2. 安全考虑：
   - 从 home 目录运行安装
   - 避免读取项目级 pip.conf/uv.toml
   - 防止恶意配置重定向到攻击者的 PyPI 服务器

3. 返回安装结果
```

**安全实现：**

```typescript
// 从 home 目录运行，避免项目级配置
import { homedir } from 'os'

await execFileNoThrowWithCwd('uv', ['tool', 'install', 'it2'], {
  cwd: homedir(),  // 关键安全设置
})
```

**各包管理器命令：**

| 管理器 | 命令 | 特点 |
|-------|------|------|
| uv | `uv tool install it2` | 全局隔离环境，最快 |
| pipx | `pipx install it2` | 用户级隔离环境 |
| pip | `pip install --user it2` | 用户级安装，无隔离 |

#### 3. 安装验证（verifyIt2Setup）

```
1. 检查 it2 是否已安装（which it2）
2. 尝试运行 it2 session list
   - 成功：验证通过
   - 失败：分析错误原因
3. 错误分析：
   - 包含 "api" / "python" / "connection refused" / "not enabled"
     → Python API 未启用
   - 其他错误 → 一般错误
```

**关键设计：**

```typescript
// 使用 session list 而非 --version
const result = await execFileNoThrow('it2', ['session', 'list'])

if (result.code !== 0) {
  const stderr = result.stderr.toLowerCase()
  if (stderr.includes('api') || 
      stderr.includes('python') || 
      stderr.includes('not enabled')) {
    return {
      success: false,
      error: 'Python API not enabled in iTerm2 preferences',
      needsPythonApiEnabled: true,  // 特定标识
    }
  }
}
```

为什么不用 `--version`？因为即使 Python API 禁用时，`it2 --version` 也会成功。

#### 4. Python API 启用指导

```typescript
export function getPythonApiInstructions(): string[] {
  return [
    'Almost done! Enable the Python API in iTerm2:',
    '',
    '  iTerm2 → Settings → General → Magic → Enable Python API',
    '',
    'After enabling, you may need to restart iTerm2.',
  ]
}
```

#### 5. 用户偏好持久化

**设置完成标记：**

```typescript
export function markIt2SetupComplete(): void {
  const config = getGlobalConfig()
  if (config.iterm2It2SetupComplete !== true) {
    saveGlobalConfig(current => ({
      ...current,
      iterm2It2SetupComplete: true,
    }))
  }
}
```

**Tmux 偏好：**

```typescript
export function setPreferTmuxOverIterm2(prefer: boolean): void {
  const config = getGlobalConfig()
  if (config.preferTmuxOverIterm2 !== prefer) {
    saveGlobalConfig(current => ({
      ...current,
      preferTmuxOverIterm2: prefer,
    }))
  }
}

export function getPreferTmuxOverIterm2(): boolean {
  return getGlobalConfig().preferTmuxOverIterm2 === true
}
```

---

## 关键代码路径与文件引用

### 内部依赖

| 文件路径 | 用途 |
|---------|------|
| `src/utils/config.ts` | `getGlobalConfig()`, `saveGlobalConfig()` |
| `src/utils/execFileNoThrow.ts` | `execFileNoThrow()`, `execFileNoThrowWithCwd()` |
| `src/utils/debug.ts` | `logForDebugging()` |
| `src/utils/log.ts` | `logError()` |

### 外部命令

| 命令 | 用途 |
|-----|------|
| `which uv` | 检测 uv |
| `which pipx` | 检测 pipx |
| `which pip/pip3` | 检测 pip |
| `which it2` | 检测 it2 是否安装 |
| `uv tool install it2` | 使用 uv 安装 it2 |
| `pipx install it2` | 使用 pipx 安装 it2 |
| `pip install --user it2` | 使用 pip 安装 it2 |
| `it2 session list` | 验证 it2 配置 |

### 关键代码位置

- **包管理器检测**: 行 40-72
- **it2 可用性检测**: 行 79-82
- **it2 安装**: 行 90-144
- **安装验证**: 行 152-195
- **Python API 指导**: 行 200-208
- **设置完成标记**: 行 214-223
- **Tmux 偏好**: 行 229-245

---

## 依赖与外部交互

### 与 Registry 的交互

```typescript
// registry.ts
import { getPreferTmuxOverIterm2 } from './it2Setup.js'

// 检测流程中检查用户偏好
if (inITerm2) {
  const preferTmux = getPreferTmuxOverIterm2()
  if (preferTmux) {
    // 跳过 iTerm2 检测，直接使用 tmux
  }
}
```

### 与 UI 的交互

设置流程通常由 UI 层触发：

```
1. UI 检测到在 iTerm2 中运行但 it2 不可用
2. 显示设置提示，提供选项：
   a. 安装 it2
   b. 使用 tmux 替代
   c. 使用 in-process 模式
3. 用户选择后调用相应函数
4. 完成后调用 markIt2SetupComplete()
```

### 配置文件

用户偏好存储在全局配置中：

```json
{
  "iterm2It2SetupComplete": true,
  "preferTmuxOverIterm2": false
}
```

---

## 风险、边界与改进建议

### 已知风险

1. **供应链攻击风险**
   - 从 PyPI 安装 it2 可能面临供应链攻击
   - **缓解**: 从 home 目录运行安装，避免项目级配置

2. **包管理器权限问题**
   - pip 安装可能需要 sudo（使用 `--user` 缓解）
   - uv/pipx 通常无此问题
   - **缓解**: 优先使用 uv/pipx

3. **Python API 启用遗漏**
   - 用户可能安装 it2 后忘记启用 Python API
   - **缓解**: 验证步骤检测并提供明确指导

4. **配置持久化延迟**
   - `saveGlobalConfig` 是同步的，但可能涉及文件 I/O
   - **影响**: 极低概率的配置丢失

5. **多用户冲突**
   - `--user` 安装在多用户系统上可能冲突
   - **缓解**: 使用 uv/pipx 的隔离环境

### 边界情况

| 场景 | 处理方式 |
|-----|---------|
| 无包管理器可用 | 返回 null，UI 应提供手动安装指导 |
| pip 失败 | 自动回退到 pip3 |
| 安装成功但验证失败 | 返回 needsPythonApiEnabled=true |
| 重复标记完成 | 检查现有值，避免不必要的写入 |
| 偏好已设置相同值 | 检查现有值，避免不必要的写入 |

### 改进建议

1. **安装进度反馈**
   - 当前安装过程无进度指示
   - 可添加安装步骤回调

2. **版本锁定**
   - 当前安装最新版 it2
   - 可考虑锁定已知兼容版本

3. **离线安装支持**
   - 当前需要网络连接
   - 可考虑支持离线安装包

4. **自动重试**
   - 网络失败时自动重试
   - 指数退避策略

5. **安装后自动验证**
   - 安装后立即验证
   - 失败时自动回滚或提示

6. **多 Python 版本支持**
   - 当前使用系统默认 Python
   - 可检测并选择兼容的 Python 版本

7. **it2 更新检测**
   - 定期检查 it2 更新
   - 提示用户更新以获得新功能
