# getWorktreePathsPortable.ts 深度研究文档

## 场景与职责

`getWorktreePathsPortable.ts` 是 `getWorktreePaths.ts` 的轻量级替代版本，提供无依赖的 Git 工作树检测功能：

1. **SDK 兼容**：用于 SDK 场景，避免引入 CLI 的依赖链
2. **依赖最小化**：仅使用 Node.js 内置的 `child_process`，无分析日志、无 bootstrap 依赖
3. **快速检测**：轻量级实现，适合需要快速工作树检测的场景

该模块是便携版本的工作树检测，被会话管理和桥接模块使用。

## 功能点目的

### 1. 便携工作树检测
- 使用原生 `child_process.execFile` 执行 git 命令
- 无外部依赖（execa、cross-spawn、which 等）
- 无分析日志记录

### 2. 路径规范化
- 对提取的路径进行 NFC Unicode 规范化

## 具体技术实现

### 关键流程

```typescript
const execFileAsync = promisify(execFileCb)

export async function getWorktreePathsPortable(cwd: string): Promise<string[]> {
  try {
    const { stdout } = await execFileAsync(
      'git',  // 直接使用 'git'，不通过 which 解析
      ['worktree', 'list', '--porcelain'],
      { cwd, timeout: 5000 },
    )
    if (!stdout) return []
    return stdout
      .split('\n')
      .filter(line => line.startsWith('worktree '))
      .map(line => line.slice('worktree '.length).normalize('NFC'))
  } catch {
    return []
  }
}
```

### 与完整版的区别

| 特性 | getWorktreePaths | getWorktreePathsPortable |
|------|------------------|-------------------------|
| Git 解析 | `gitExe()`（缓存的 which 结果） | 直接使用 `'git'` |
| 分析日志 | 有 | 无 |
| 错误处理 | 详细 | 静默（catch 返回 []） |
| 工作树排序 | 当前优先 | 无排序 |
| 依赖 | execa、分析服务 | 仅 child_process |
| 超时 | 默认 | 5000ms |

## 关键代码路径与文件引用

### 核心导出
- `getWorktreePathsPortable(cwd: string): Promise<string[]>` - 便携工作树路径获取

### 依赖关系

**被以下模块导入**：
- `src/bridge/bridgePointer.ts` - 桥接指针
- `src/utils/listSessionsImpl.ts` - 会话列表实现（SDK）
- `src/utils/sessionStoragePortable.ts` - 便携会话存储

**依赖的模块**：
- 仅 Node.js 内置模块

### 文件位置
- 源码：`src/utils/getWorktreePathsPortable.ts` (27 行)

## 依赖与外部交互

### Node.js 内置模块
- `child_process` - `execFile` 回调版本
- `util` - `promisify`

### 项目内部依赖
- 无

### 外部依赖
- Git 命令行工具（需要在 PATH 中）

## 风险、边界与改进建议

### 已知风险

1. **Git 可执行文件定位**：直接使用 `'git'` 可能找不到 git（如果不在 PATH 中）
2. **无错误信息**：所有错误都被静默处理，难以调试
3. **无排序**：返回的工作树未排序，当前工作树不优先

### 边界情况

1. **非 Git 目录**：返回空数组
2. **Git 未安装**：返回空数组
3. **超时**：5 秒超时后返回空数组
4. **空输出**：返回空数组

### 改进建议

1. **错误暴露**：考虑添加可选的错误回调或日志参数
2. **Git 定位**：考虑接受可选的 git 路径参数
3. **排序支持**：添加与完整版相同的排序逻辑
4. **测试覆盖**：当前没有专门的测试文件，建议添加单元测试
5. **文档说明**：在函数文档中明确说明与完整版的差异

### 使用场景建议

- **使用 `getWorktreePathsPortable`**：SDK 场景、需要最小依赖、不需要分析日志
- **使用 `getWorktreePaths`**：CLI 场景、需要分析日志、需要正确的工作树排序
