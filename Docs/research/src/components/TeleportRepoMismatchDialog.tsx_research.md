# TeleportRepoMismatchDialog.tsx 深度研究文档

## 场景与职责

TeleportRepoMismatchDialog 是 Claude Code CLI 中处理 **Teleport 仓库不匹配** 场景的交互式对话框。当用户尝试恢复一个属于不同 GitHub 仓库的远程会话时，系统会检测到此不匹配并展示此对话框，允许用户选择正确的本地仓库路径。

该组件的核心职责：
1. **仓库路径选择**：展示已知路径列表供用户选择
2. **路径验证**：验证所选路径是否包含正确的仓库
3. **无效路径清理**：自动从配置中移除失效的路径映射
4. **用户体验优化**：提供取消选项和错误反馈

## 功能点目的

### 1. 仓库不匹配检测
当 `teleportResumeCodeSession` 检测到当前目录的仓库与会话目标仓库不匹配时，会触发此对话框。系统通过 `validateSessionRepository` 函数检测以下情况：
- 当前不在 Git 仓库中
- 当前仓库与会话目标仓库不同
- 跨 GitHub Enterprise 实例的仓库不匹配

### 2. 路径选择交互
- 展示用户曾经使用过的目标仓库的本地路径列表
- 支持键盘导航（↑/↓）和数字快捷选择
- 提供 "Cancel" 选项退出流程

### 3. 实时验证
- 用户选择路径后，异步验证该路径是否包含正确的仓库
- 验证期间显示 loading spinner
- 验证失败时显示错误并移除无效路径

## 具体技术实现

### 关键数据结构

```typescript
// 组件 Props
type Props = {
  targetRepo: string;           // 目标仓库 "owner/repo" 格式
  initialPaths: string[];       // 初始路径列表
  onSelectPath: (path: string) => void;  // 选择回调
  onCancel: () => void;         // 取消回调
};

// 内部状态
const [availablePaths, setAvailablePaths] = useState(initialPaths);
const [errorMessage, setErrorMessage] = useState<string | null>(null);
const [validating, setValidating] = useState(false);
```

### 核心流程

#### 1. 路径选择处理
```typescript
const handleChange = async (value: string) => {
  if (value === "cancel") {
    onCancel();
    return;
  }
  
  setValidating(true);
  setErrorMessage(null);
  
  // 验证路径
  const isValid = await validateRepoAtPath(value, targetRepo);
  
  if (isValid) {
    onSelectPath(value);
    return;
  }
  
  // 验证失败：移除无效路径并显示错误
  removePathFromRepo(targetRepo, value);
  const updatedPaths = availablePaths.filter(p => p !== value);
  setAvailablePaths(updatedPaths);
  setValidating(false);
  setErrorMessage(`${getDisplayPath(value)} no longer contains the correct repository. Select another path.`);
};
```

#### 2. 选项构建
```typescript
const options = [
  ...availablePaths.map(path => ({
    label: <Text>Use <Text bold>{getDisplayPath(path)}</Text></Text>,
    value: path
  })),
  { label: "Cancel", value: "cancel" }
];
```

#### 3. 条件渲染逻辑
```typescript
// 有可用路径时
availablePaths.length > 0 ? (
  <>
    {errorMessage && <Text color="error">{errorMessage}</Text>}
    <Text>Open Claude Code in <Text bold>{targetRepo}</Text>:</Text>
    {validating ? (
      <Box><Spinner /><Text> Validating repository…</Text></Box>
    ) : (
      <Select options={options} onChange={handleChange} />
    )}
  </>
) : (
  // 无可用路径时
  <Box flexDirection="column" gap={1}>
    {errorMessage && <Text color="error">{errorMessage}</Text>}
    <Text dimColor>Run claude --teleport from a checkout of {targetRepo}</Text>
  </Box>
)
```

## 关键代码路径与文件引用

### 本文件导出
- `TeleportRepoMismatchDialog` - 仓库不匹配对话框组件

### 依赖文件
| 文件路径 | 用途 |
|---------|------|
| `../ink.js` | Ink UI 组件（Box, Text） |
| `../utils/file.js` | `getDisplayPath` - 路径显示格式化 |
| `../utils/githubRepoPathMapping.js` | 路径映射管理 |
| `./CustomSelect/index.js` | Select 组件 |
| `./design-system/Dialog.js` | 对话框容器 |
| `./Spinner.js` | Loading 动画 |

### 调用方
| 文件路径 | 调用方式 |
|---------|---------|
| `src/dialogLaunchers.tsx` | `launchTeleportRepoMismatchDialog` 封装函数 |
| `src/main.tsx` | 通过 dialogLaunchers 调用 |

### 依赖的工具函数

#### githubRepoPathMapping.ts
```typescript
// 获取已知路径
export function getKnownPathsForRepo(repo: string): string[]

// 验证路径
export async function validateRepoAtPath(
  path: string, 
  expectedRepo: string
): Promise<boolean>

// 移除无效路径
export function removePathFromRepo(
  repo: string, 
  pathToRemove: string
): void
```

#### file.ts
```typescript
// 格式化路径显示（相对路径优先，家目录用 ~ 表示）
export function getDisplayPath(filePath: string): string
```

## 依赖与外部交互

### 配置持久化
路径映射存储在全局配置中（`~/.claude/config.json`）：
```json
{
  "githubRepoPaths": {
    "anthropic/claude-code": [
      "/home/user/projects/claude-code",
      "/workspace/claude-code"
    ]
  }
}
```

### 验证机制
`validateRepoAtPath` 的实现逻辑：
1. 获取指定路径的 Git 远程 URL
2. 解析仓库名称（owner/repo）
3. 不区分大小写比较

```typescript
export async function validateRepoAtPath(
  path: string,
  expectedRepo: string,
): Promise<boolean> {
  try {
    const remoteUrl = await getRemoteUrlForDir(path);
    if (!remoteUrl) return false;
    
    const actualRepo = parseGitHubRepository(remoteUrl);
    if (!actualRepo) return false;
    
    return actualRepo.toLowerCase() === expectedRepo.toLowerCase();
  } catch {
    return false;
  }
}
```

### 路径发现机制
路径映射通过 `updateGithubRepoPathMapping` 在启动时自动更新：
- 检测当前目录的 GitHub 仓库
- 将路径添加到对应仓库的列表
- 最近使用的路径排在最前

## 风险、边界与改进建议

### 已知风险

1. **路径验证阻塞**
   - 验证是同步阻塞的，如果 Git 命令卡住，UI 会冻结
   - 建议添加超时机制

2. **路径列表可能过时**
   - 用户可能移动或删除了仓库
   - 当前只在选择后才验证，建议预验证

3. **跨主机仓库混淆**
   - 代码中处理了 `sessionHost` 和 `currentHost`
   - 但对话框显示未明确区分不同 GHE 实例

### 边界情况

1. **空路径列表**
   - 当 `initialPaths` 为空时，显示提示信息引导用户
   - 用户需要手动切换到正确目录

2. **所有路径都无效**
   - 用户逐一选择后，路径列表会变空
   - 最终显示与空列表相同的提示

3. **验证期间取消**
   - 当前不支持在验证过程中取消
   - 用户需等待验证完成

### 改进建议

1. **预验证路径**
   ```typescript
   // 组件挂载时预验证所有路径
   useEffect(() => {
     Promise.all(
       initialPaths.map(async path => ({
         path,
         valid: await validateRepoAtPath(path, targetRepo)
       }))
     ).then(results => {
       const validPaths = results.filter(r => r.valid).map(r => r.path);
       setAvailablePaths(validPaths);
     });
   }, []);
   ```

2. **添加路径手动输入**
   - 当自动发现的路径都无效时，允许用户手动输入路径
   - 可集成 `Input` 组件实现

3. **显示仓库主机信息**
   - 对于 GHE 仓库，显示主机名帮助用户识别
   - 避免跨实例混淆

4. **超时处理**
   ```typescript
   const validateWithTimeout = async (path: string, timeoutMs: number) => {
     return Promise.race([
       validateRepoAtPath(path, targetRepo),
       new Promise<boolean>((_, reject) => 
         setTimeout(() => reject(new Error('Timeout')), timeoutMs)
       )
     ]);
   };
   ```

5. **批量清理无效路径**
   - 提供 "Clean up invalid paths" 选项
   - 一次性验证并移除所有失效路径

### 测试建议

1. **单元测试场景**：
   - 路径选择回调触发
   - 验证失败时的错误显示
   - 空路径列表的渲染
   - 取消操作处理

2. **集成测试场景**：
   - 与 `githubRepoPathMapping` 的交互
   - 实际 Git 仓库验证
   - 配置持久化验证

---

**文档生成时间**：2026-04-01  
**组件路径**：`src/components/TeleportRepoMismatchDialog.tsx`  
**关联研究文件**：`src/utils/githubRepoPathMapping.ts`, `src/dialogLaunchers.tsx`
