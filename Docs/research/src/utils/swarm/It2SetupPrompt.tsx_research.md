# It2SetupPrompt.tsx 研究文档

## 场景与职责

`It2SetupPrompt.tsx` 是一个 React/Ink 组件，用于在 iTerm2 环境下引导用户完成 `it2` CLI 工具的安装和配置。它是 Agent Swarm 功能在 iTerm2 终端中的关键入口点，确保用户能够使用原生 iTerm2 分屏功能来展示队友（teammate）窗口。

### 核心职责
1. **检测 Python 包管理器**: 自动检测系统上可用的 Python 包管理器（uvx、pipx、pip）
2. **引导安装流程**: 提供交互式 UI 引导用户安装 `it2` 工具
3. **验证 API 连接**: 验证 it2 是否能与 iTerm2 的 Python API 正常通信
4. **提供回退选项**: 当 it2 安装失败或用户选择不安装时，提供使用 tmux 的替代方案

## 功能点目的

### 1. 安装引导流程 (Setup Flow)
- **initial**: 初始状态，展示选项让用户选择安装 it2、使用 tmux 或取消
- **installing**: 正在安装 it2，显示加载动画
- **install-failed**: 安装失败，提供重试、使用 tmux 或取消的选项
- **verify-api**: 准备验证 it2 与 iTerm2 的连接
- **api-instructions**: 显示启用 Python API 的说明
- **verifying**: 正在验证连接
- **success**: 验证成功，准备完成
- **failed**: 验证失败，提供故障排除建议

### 2. 包管理器优先级
按照以下顺序检测和使用包管理器：
1. `uv` (uvx) - 首选，隔离环境
2. `pipx` - 次选，用户级隔离
3. `pip/pip3` - 最后选择，使用 `--user` 安装

### 3. 用户选择持久化
- 用户选择 "Use tmux instead" 会调用 `setPreferTmuxOverIterm2(true)`，将偏好保存到全局配置
- 成功完成 it2 安装后调用 `markIt2SetupComplete()`，避免重复提示

## 具体技术实现

### 关键数据结构

```typescript
type SetupStep = 'initial' | 'installing' | 'install-failed' | 
                'verify-api' | 'api-instructions' | 'verifying' | 
                'success' | 'failed';

type Props = {
  onDone: (result: 'installed' | 'use-tmux' | 'cancelled') => void;
  tmuxAvailable: boolean;
};
```

### 状态管理
使用 React 的 `useState` 管理：
- `step`: 当前安装步骤
- `packageManager`: 检测到的 Python 包管理器
- `error`: 错误信息

### 关键流程

#### 1. 初始检测 (useEffect)
```typescript
useEffect(() => {
  detectPythonPackageManager().then(pm => {
    setPackageManager(pm);
  });
}, []);
```

#### 2. 安装处理
```typescript
const handleInstall = async () => {
  if (!packageManager) {
    setError("No Python package manager found...");
    setStep('failed');
    return;
  }
  setStep('installing');
  const result = await installIt2(packageManager);
  if (result.success) {
    setStep('api-instructions');
  } else {
    setError(result.error || "Installation failed");
    setStep('install-failed');
  }
};
```

#### 3. 验证流程
用户按 Enter 后触发验证：
```typescript
useInput((_input, key) => {
  if (step === 'api-instructions' && key.return) {
    setStep('verifying');
    verifyIt2Setup().then(result => {
      if (result.success) {
        markIt2SetupComplete();
        setStep('success');
        setTimeout(onDone, 1500, 'installed');
      } else {
        setError(result.error || 'Verification failed');
        setStep('failed');
      }
    });
  }
});
```

### UI 渲染策略

组件使用 React Compiler (`_c` 函数) 进行自动记忆化，通过比较依赖项来决定是否复用之前的渲染结果。每个渲染函数（如 `renderInitialPrompt`、`renderInstalling` 等）都返回 Ink 组件树。

关键 UI 组件：
- `Pane`: 带颜色主题的容器
- `Select`: 自定义选择组件，支持选项描述
- `Spinner`: 加载动画
- `Box`/`Text`: Ink 基础布局组件

## 关键代码路径与文件引用

### 本文件导出
- `It2SetupPrompt`: 主组件函数

### 依赖导入

| 导入路径 | 用途 |
|---------|------|
| `../../components/CustomSelect/index.js` | Select 组件 |
| `../../components/design-system/Pane.js` | Pane 容器组件 |
| `../../components/Spinner.js` | Spinner 加载动画 |
| `../../hooks/useExitOnCtrlCDWithKeybindings.js` | Ctrl+C/D 退出处理 |
| `../../ink.js` | Box, Text, useInput |
| `../../keybindings/useKeybinding.js` | 键盘绑定 |
| `./backends/it2Setup.js` | 核心安装逻辑 |

### 依赖文件详细说明

#### `src/utils/swarm/backends/it2Setup.ts`
提供以下功能：
- `detectPythonPackageManager()`: 检测 uv/pipx/pip
- `getPythonApiInstructions()`: 返回启用 Python API 的说明
- `installIt2(pm)`: 使用指定包管理器安装 it2
- `markIt2SetupComplete()`: 标记安装完成
- `setPreferTmuxOverIterm2(bool)`: 设置 tmux 偏好
- `verifyIt2Setup()`: 验证 it2 连接

## 依赖与外部交互

### 外部依赖
1. **iTerm2**: 需要 iTerm2 终端环境
2. **Python 包管理器**: uv、pipx 或 pip
3. **it2 CLI**: 通过 Python 包安装

### 配置持久化
- 使用 `src/utils/config.ts` 中的 `getGlobalConfig()` 和 `saveGlobalConfig()`
- 配置键：`iterm2It2SetupComplete` (boolean)、`preferTmuxOverIterm2` (boolean)

### 键盘交互
- `Esc`: 取消（通过 `useKeybinding("confirm:no", handleCancel)`）
- `Enter`: 在 api-instructions 步骤确认并继续
- `Ctrl+C/D`: 退出（通过 `useExitOnCtrlCDWithKeybindings`）

## 风险、边界与改进建议

### 风险点

1. **包管理器检测失败**
   - 如果系统 PATH 中没有 uv/pipx/pip，会显示 "No Python package manager found"
   - 用户需要手动安装 Python 环境

2. **安装目录安全**
   - `it2Setup.ts` 中安装时指定 `cwd: homedir()`，避免读取项目级的 pip.conf/uv.toml
   - 这是为了防止恶意配置重定向到攻击者的 PyPI 服务器

3. **Python API 启用**
   - 用户需要在 iTerm2 设置中手动启用 Python API
   - 启用后可能需要重启 iTerm2

### 边界情况

1. **tmux 不可用**
   - 如果 `tmuxAvailable` 为 false，"Use tmux instead" 选项不会显示
   - 用户只能选择安装 it2 或取消

2. **重复安装提示**
   - 通过 `iterm2It2SetupComplete` 配置避免重复提示
   - 用户可以通过重置配置重新触发安装流程

3. **并发安装**
   - 安装过程中显示 Spinner，防止用户重复触发
   - 安装状态通过 `step === 'installing'` 控制

### 改进建议

1. **离线安装支持**
   - 当前仅支持从 PyPI 在线安装
   - 可考虑支持本地 wheel 文件安装

2. **安装进度显示**
   - 当前仅显示 "Installing..."
   - 可解析包管理器输出显示详细进度

3. **自动重试机制**
   - 网络失败时可自动重试
   - 提供代理配置选项

4. **多版本管理**
   - 当前不检查 it2 版本
   - 可考虑添加版本检查和更新提示

5. **错误恢复**
   - 安装失败后提供更详细的故障排除步骤
   - 链接到官方文档或 FAQ
