# SandboxDependenciesTab.tsx 深度研究文档

## 1. 场景与职责

**SandboxDependenciesTab** 是 Claude Code CLI 沙箱设置界面的"Dependencies"标签页组件，负责：

1. **检测和展示沙箱运行所需的系统依赖**
2. **提供平台特定的安装指导**
3. **区分错误（阻塞）和警告（非阻塞）状态**

### 使用场景

- **用户首次启用沙箱时**：检查系统是否满足运行条件
- **执行 `/doctor` 命令时**：作为诊断信息的一部分展示
- **沙箱启动失败时**：帮助用户定位缺失的依赖
- **跨平台支持**：根据 macOS/Linux 显示不同的依赖要求

### 平台差异

| 平台 | 必需依赖 | 可选依赖 |
|------|----------|----------|
| macOS | ripgrep (rg) | seatbelt（系统内置） |
| Linux/WSL | ripgrep, bubblewrap (bwrap), socat | seccomp（用于 Unix Socket 阻断） |

---

## 2. 功能点目的

### 2.1 依赖检测

```typescript
type Props = {
  depCheck: SandboxDependencyCheck  // 来自 sandbox-adapter 的检测结果
}
```

检测结果结构：
```typescript
interface SandboxDependencyCheck {
  errors: string[]    // 缺失的必需依赖
  warnings: string[]  // 可选依赖缺失或警告
}
```

### 2.2 平台适配

组件根据 `getPlatform()` 返回值调整展示：

- **macOS**: 显示 seatbelt 内置状态，ripgrep 安装提示使用 `brew install ripgrep`
- **Linux/WSL**: 显示 bwrap、socat、seccomp 状态，安装提示使用 `apt install ...`

### 2.3 安装指导

| 依赖 | macOS 安装命令 | Linux 安装命令 |
|------|----------------|----------------|
| ripgrep | `brew install ripgrep` | `apt install ripgrep` |
| bubblewrap | N/A | `apt install bubblewrap` |
| socat | N/A | `apt install socat` |
| seccomp | N/A | `npm install -g @anthropic-ai/sandbox-runtime` |

---

## 3. 具体技术实现

### 3.1 组件架构

```tsx
export function SandboxDependenciesTab({ depCheck }: Props): React.ReactNode {
  const platform = getPlatform()
  const isMac = platform === 'macos'
  
  // 检测特定错误
  const rgMissing = depCheck.errors.some(e => e.includes('ripgrep'))
  const bwrapMissing = depCheck.errors.some(e => e.includes('bwrap'))
  const socatMissing = depCheck.errors.some(e => e.includes('socat'))
  const seccompMissing = depCheck.warnings.length > 0
  
  // 过滤其他错误
  const otherErrors = depCheck.errors.filter(
    e => !e.includes('ripgrep') && !e.includes('bwrap') && !e.includes('socat')
  )
  
  return (
    <Box flexDirection="column" paddingY={1} gap={1}>
      {/* macOS: seatbelt 状态 */}
      {isMac && <seatbeltStatus />}
      
      {/* ripgrep 状态（全平台） */}
      <ripgrepStatus />
      
      {/* Linux: bwrap, socat, seccomp */}
      {!isMac && <linuxDependencies />}
      
      {/* 其他错误 */}
      {otherErrors.map(...)}
    </Box>
  )
}
```

### 3.2 React Compiler 记忆化

组件使用 24 个记忆化槽位（`const $ = _c(24)`），对以下数据进行缓存：

1. `platform` - 平台信息（不变）
2. `rgMissing`, `bwrapMissing`, `socatMissing` - 依赖缺失状态
3. `otherErrors` - 过滤后的错误列表
4. 各 JSX 片段的渲染结果

### 3.3 错误分类逻辑

```typescript
// 必需依赖错误（阻塞沙箱运行）
const rgMissing = depCheck.errors.some(e => e.includes('ripgrep'))
const bwrapMissing = depCheck.errors.some(e => e.includes('bwrap'))
const socatMissing = depCheck.errors.some(e => e.includes('socat'))

// 可选依赖警告（非阻塞，功能受限）
const seccompMissing = depCheck.warnings.length > 0
```

---

## 4. 关键代码路径与文件引用

### 4.1 直接依赖

| 导入 | 路径 | 用途 |
|------|------|------|
| React | 'react' | JSX 运行时 |
| Box, Text | '../../ink.js' | Ink 终端 UI 组件 |
| getPlatform | '../../utils/platform.js' | 平台检测 |
| SandboxDependencyCheck | '../../utils/sandbox/sandbox-adapter.js' | 类型定义 |

### 4.2 依赖检测流程

```
SandboxDependenciesTab.tsx
  ↓ 接收 props
SandboxDependencyCheck (来自 sandbox-adapter.ts)
  ↓ 调用
BaseSandboxManager.checkDependencies() (@anthropic-ai/sandbox-runtime)
  ↓ 系统检测
检查以下命令是否存在：
  - rg (ripgrep)
  - bwrap (bubblewrap, Linux only)
  - socat (Linux only)
  - seccomp BPF 文件 (Linux only)
```

### 4.3 平台检测实现

**src/utils/platform.ts**:
```typescript
export const getPlatform = memoize((): Platform => {
  if (process.platform === 'darwin') return 'macos'
  if (process.platform === 'win32') return 'windows'
  if (process.platform === 'linux') {
    // 检查 /proc/version 判断是否 WSL
    const procVersion = readFileSync('/proc/version', 'utf8')
    if (procVersion.toLowerCase().includes('microsoft') || 
        procVersion.toLowerCase().includes('wsl')) {
      return 'wsl'
    }
    return 'linux'
  }
  return 'unknown'
})
```

---

## 5. 依赖与外部交互

### 5.1 外部包依赖

| 包名 | 用途 |
|------|------|
| @anthropic-ai/sandbox-runtime | 提供依赖检测逻辑 |
| react | React 框架 |
| ink | 终端渲染框架 |

### 5.2 系统依赖详情

#### ripgrep (rg)
- **用途**: 扫描危险目录，用于沙箱路径安全检测
- **全平台必需**: 是
- **检测方式**: 执行 `rg --version`

#### bubblewrap (bwrap)
- **用途**: Linux 沙箱容器化工具
- **平台**: Linux/WSL 必需
- **检测方式**: 执行 `bwrap --version`

#### socat
- **用途**: 网络代理转发，用于沙箱内网络拦截
- **平台**: Linux/WSL 必需
- **检测方式**: 执行 `socat -V`

#### seccomp
- **用途**: 系统调用过滤，用于阻断 Unix Socket
- **平台**: Linux/WSL 可选
- **检测方式**: 检查 BPF 文件是否存在
  - `/usr/local/lib/node_modules/@anthropic-ai/sandbox-runtime/seccomp/`
  - 或通过 `sandbox.seccomp.bpfPath` 配置的路径

#### seatbelt
- **用途**: macOS 内置沙箱机制
- **平台**: macOS 内置
- **状态**: 始终可用，无需检测

---

## 6. 风险、边界与改进建议

### 6.1 已知问题与修复

**Issue #31804**: 之前在 macOS 上错误地显示 Linux 依赖安装指令

```typescript
// 修复前：无条件渲染 Linux 依赖
// 修复后：平台条件渲染
const isMac = platform === 'macos'
{isMac && <seatbeltStatus />}
{!isMac && <linuxDependencies />}
```

### 6.2 边界情况

| 场景 | 行为 |
|------|------|
| 平台检测失败 | 默认按 Linux 处理（显示所有依赖） |
| depCheck.errors 为空 | 不渲染错误列表 |
| depCheck.warnings 为空 | 不渲染 seccomp 警告 |
| 其他未知错误 | 原样渲染错误文本 |

### 6.3 安全风险

1. **命令注入**: 安装提示中的命令是硬编码的，不依赖用户输入
2. **路径遍历**: 依赖检测使用系统 PATH，不受用户控制
3. **信息泄露**: 错误信息可能包含系统路径，但仅限于当前用户可见范围

### 6.4 改进建议

1. **自动安装**: 提供一键安装脚本（如 `/sandbox install-deps`）
2. **版本检查**: 不仅检查存在性，还检查最低版本要求
3. **容器环境检测**: 在 Docker 等容器中给出特殊提示
4. **离线安装指导**: 提供离线环境下手动安装的指导
5. **依赖冲突检测**: 检测与其他安全软件的冲突（如 SELinux）

### 6.5 测试要点

- 各平台（macOS/Linux/WSL）的正确识别
- 依赖存在/缺失的不同组合
- 错误信息包含特定关键词的检测
- 安装提示的正确性
- 长错误列表的渲染

---

## 附录：相关文件索引

| 文件 | 描述 |
|------|------|
| `src/components/sandbox/SandboxSettings.tsx` | 沙箱设置主界面 |
| `src/components/sandbox/SandboxDoctorSection.tsx` | Doctor 中的沙箱状态展示 |
| `src/utils/sandbox/sandbox-adapter.ts` | 依赖检测实现 |
| `src/utils/platform.ts` | 平台检测工具 |
| `src/utils/ripgrep.ts` | ripgrep 命令配置 |
