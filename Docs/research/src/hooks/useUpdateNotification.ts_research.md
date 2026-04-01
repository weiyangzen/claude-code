# useUpdateNotification.ts 研究文档

## 场景与职责

`useUpdateNotification` 是一个 React Hook，用于管理自动更新通知的显示逻辑。它解决了以下核心问题：

1. **版本变更检测**：当自动更新器检测到新版本并完成安装后，需要向用户显示更新成功的通知
2. **通知去重**：避免在同一版本上重复显示更新通知
3. **语义化版本处理**：将完整的版本号（可能包含构建元数据）标准化为 `major.minor.patch` 格式进行比较

该 Hook 被 `NativeAutoUpdater.tsx` 和 `AutoUpdater.tsx` 组件使用，这两个组件分别处理原生安装和 npm 安装的自动更新流程。

## 功能点目的

### 1. 语义化版本提取 (`getSemverPart`)

```typescript
export function getSemverPart(version: string): string {
  return `${major(version, { loose: true })}.${minor(version, { loose: true })}.${patch(version, { loose: true })}`
}
```

- 使用 `semver` 库的 `major`、`minor`、`patch` 函数提取版本的核心部分
- `loose: true` 选项允许解析非严格格式的版本号
- 将 `1.2.3+build.123` 转换为 `1.2.3`，忽略构建元数据

### 2. 更新通知判断 (`shouldShowUpdateNotification`)

```typescript
export function shouldShowUpdateNotification(
  updatedVersion: string,
  lastNotifiedSemver: string | null,
): boolean {
  const updatedSemver = getSemverPart(updatedVersion)
  return updatedSemver !== lastNotifiedSemver
}
```

- 比较新版本的 semver 部分与上次通知的版本
- 如果不同，说明是新版本，应该显示通知
- 纯函数设计，便于测试和独立使用

### 3. Hook 主逻辑 (`useUpdateNotification`)

```typescript
export function useUpdateNotification(
  updatedVersion: string | null | undefined,
  initialVersion: string = MACRO.VERSION,
): string | null {
  const [lastNotifiedSemver, setLastNotifiedSemver] = useState<string | null>(
    () => getSemverPart(initialVersion),
  )

  if (!updatedVersion) {
    return null
  }

  const updatedSemver = getSemverPart(updatedVersion)
  if (updatedSemver !== lastNotifiedSemver) {
    setLastNotifiedSemver(updatedSemver)
    return updatedSemver
  }
  return null
}
```

- 使用 `useState` 的懒加载初始化，只在首次渲染时计算初始版本
- 当 `updatedVersion` 为 null/undefined 时返回 null（无更新或无通知）
- 检测到新版本时，更新状态并返回标准化后的版本号
- 返回的版本号可用于 UI 显示（如 "✓ Update installed · Restart to update"）

## 具体技术实现

### 关键流程

1. **初始化阶段**：
   - Hook 首次渲染时，使用 `MACRO.VERSION`（编译时注入的当前版本）初始化 `lastNotifiedSemver`
   - 使用懒加载函数避免每次渲染都重新计算

2. **更新检测阶段**：
   - 当自动更新器完成更新后，传入 `updatedVersion`（来自 `AutoUpdaterResult.version`）
   - Hook 提取新版本的 semver 部分进行比较

3. **状态更新阶段**：
   - 如果版本不同，调用 `setLastNotifiedSemver` 更新状态
   - 返回新版本号供调用方显示通知

### 数据结构

- **输入**：`updatedVersion: string | null | undefined` - 来自自动更新器的结果
- **状态**：`lastNotifiedSemver: string | null` - 上次通知的版本（semver 格式）
- **输出**：`string | null` - 需要显示的版本号，null 表示无需显示

### 依赖的外部库

- `semver`：用于解析和提取语义化版本的各个部分

## 关键代码路径与文件引用

### 本文件
- `/home/sansha/Github/claude-code-instructkr/src/hooks/useUpdateNotification.ts` - Hook 实现

### 调用方
- `/home/sansha/Github/claude-code-instructkr/src/components/NativeAutoUpdater.tsx` (第 64 行)
  ```typescript
  const updateSemver = useUpdateNotification(autoUpdaterResult?.version);
  ```
- `/home/sansha/Github/claude-code-instructkr/src/components/AutoUpdater.tsx` (第 36 行)
  ```typescript
  const updateSemver = useUpdateNotification(autoUpdaterResult?.version);
  ```

### 使用场景
两个组件都在渲染逻辑中使用返回值：
```typescript
{autoUpdaterResult?.status === 'success' && showSuccessMessage && updateSemver && (
  <Text color="success" wrap="truncate">
    ✓ Update installed · Restart to update
  </Text>
)}
```

## 依赖与外部交互

### 运行时依赖
| 依赖 | 用途 |
|------|------|
| `semver` (major, minor, patch) | 解析和提取版本号的核心部分 |
| `react` (useState) | React Hook API |
| `MACRO.VERSION` | 编译时注入的当前应用版本 |

### 无外部副作用
- 纯计算逻辑，无副作用
- 不调用外部服务或修改全局状态
- 状态仅限于 Hook 内部

## 风险、边界与改进建议

### 潜在风险

1. **版本格式不兼容**：
   - 如果传入的版本号不是有效的 semver 格式，`semver` 库的 `loose: true` 会尝试宽松解析
   - 极端情况下可能返回 `null.0.0` 或抛出异常

2. **MACRO.VERSION 未定义**：
   - 如果编译时未注入 `MACRO.VERSION`，初始化会使用 `undefined`
   - 可能导致首次比较时行为异常

3. **并发更新**：
   - 如果在同一渲染周期内多次调用 `setLastNotifiedSemver`，React 会批量处理
   - 但快速连续的版本变更可能导致通知丢失

### 边界情况

| 场景 | 行为 |
|------|------|
| `updatedVersion` 为 null | 返回 null，不显示通知 |
| `updatedVersion` 与当前版本相同 | 返回 null，不重复通知 |
| `updatedVersion` 为预发布版本 (如 `1.0.0-beta`) | 提取 `1.0.0` 进行比较 |
| 版本号包含构建元数据 | 忽略构建元数据，只比较核心版本 |

### 改进建议

1. **添加版本格式验证**：
   ```typescript
   if (!semver.valid(updatedVersion, { loose: true })) {
     return null;
   }
   ```

2. **支持预发布版本通知**：
   - 当前实现会忽略预发布标识符
   - 如果需要区分 `1.0.0-alpha` 和 `1.0.0-beta`，需要修改比较逻辑

3. **持久化通知状态**：
   - 当前状态在组件卸载后丢失
   - 可考虑使用 localStorage 或全局状态持久化，避免页面刷新后重复通知

4. **添加测试覆盖**：
   - 测试各种版本格式（带构建元数据、预发布版本、无效格式）
   - 测试 Hook 的重复渲染行为
