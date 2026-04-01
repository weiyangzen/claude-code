# useIssueFlagBanner.ts 深度研究文档

## 场景与职责

`useIssueFlagBanner` 是一个内部（Ant-only）功能钩子，用于检测用户与 Claude 的交互摩擦信号，并在特定条件下显示反馈横幅。该功能仅对 Anthropic 内部员工启用（`USER_TYPE === 'ant'`）。

### 核心场景

1. **摩擦信号检测**：识别用户表达不满或纠正 Claude 的输入
2. **会话容器兼容性检查**：检测会话是否使用了外部命令（影响容器化）
3. **内部反馈收集**：帮助 Anthropic 识别需要改进的交互场景

### 功能限制

- **仅限内部用户**：通过 `process.env.USER_TYPE !== 'ant'` 检查
- **冷却期**：触发后 30 分钟内不再触发
- **最小提交次数**：至少 3 次提交后才可能触发

---

## 功能点目的

### 1. 摩擦信号检测

通过正则表达式匹配用户输入，识别以下摩擦模式：

| 模式类别 | 正则表达式 | 示例 |
|---------|-----------|------|
| 直接否定 | `/^no[,!]\s/i` | "No, that's wrong" |
| 错误指正 | `/\bthat'?s (wrong\|incorrect\|not (what\|right\|correct))\b/i` | "That's wrong" |
| 意图不符 | `/\bnot what I (asked\|wanted\|meant\|said)\b/i` | "Not what I asked" |
| 重复指令 | `/\bI (said\|asked\|wanted\|told you\|already said)\b/i` | "I already said..." |
| 质疑行为 | `/\bwhy did you\b/i` | "Why did you do that?" |
| 期望不符 | `/\byou should(n'?t\| not)? have\b/i` | "You shouldn't have..." |
| 明确重试 | `/\btry again\b/i` | "Try again" |
| 撤销请求 | `/\b(undo\|revert) (that\|this\|it\|what you)\b/i` | "Undo that" |

### 2. 会话容器兼容性检查

检测会话是否包含影响容器化运行的命令：

**外部命令模式**：
- `curl`, `wget` - 网络下载
- `ssh`, `kubectl`, `srun` - 远程/集群执行
- `docker` - 容器操作
- `bq`, `gsutil`, `gcloud`, `aws` - 云服务
- `git push/pull/fetch` - 远程 Git 操作
- `gh pr/issue` - GitHub CLI
- `nc`, `ncat`, `telnet`, `ftp` - 网络工具

**MCP 工具检测**：
- 工具名以 `mcp__` 开头的工具

### 3. 触发条件

必须同时满足：
1. 用户类型为 'ant'
2. 提交次数 >= 3
3. 距离上次触发 >= 30 分钟
4. 会话容器兼容（无外部命令）
5. 存在摩擦信号

---

## 具体技术实现

### 关键常量

```typescript
// 外部命令检测模式
const EXTERNAL_COMMAND_PATTERNS = [
  /\bcurl\b/, /\bwget\b/, /\bssh\b/, /\bkubectl\b,
  /\bsrun\b/, /\bdocker\b/, /\bbq\b/, /\bgsutil\b,
  /\bgcloud\b/, /\baws\b/, /\bgit\s+push\b/,
  /\bgit\s+pull\b/, /\bgit\s+fetch\b/, /\bgh\s+(pr|issue)\b/,
  /\bnc\b/, /\bncat\b/, /\btelnet\b/, /\bftp\b/,
]

// 摩擦信号检测模式
const FRICTION_PATTERNS = [
  /^no[,!]\s/i,
  /\bthat'?s (wrong|incorrect|not (what|right|correct))\b/i,
  /\bnot what I (asked|wanted|meant|said)\b/i,
  /\bI (said|asked|wanted|told you|already said)\b/i,
  /\bwhy did you\b/i,
  /\byou should(n'?t| not)? have\b/i,
  /\byou were supposed to\b/i,
  /\btry again\b/i,
  /\b(undo|revert) (that|this|it|what you)\b/i,
]

// 触发阈值
const MIN_SUBMIT_COUNT = 3
const COOLDOWN_MS = 30 * 60 * 1000  // 30 分钟
```

### 核心函数

#### 会话容器兼容性检查

```typescript
export function isSessionContainerCompatible(messages: Message[]): boolean {
  for (const msg of messages) {
    if (msg.type !== 'assistant') continue
    
    const content = msg.message.content
    if (!Array.isArray(content)) continue
    
    for (const block of content) {
      if (block.type !== 'tool_use' || !('name' in block)) continue
      
      const toolName = block.name as string
      
      // MCP 工具不兼容
      if (toolName.startsWith('mcp__')) return false
      
      // Bash 工具检查外部命令
      if (toolName === BASH_TOOL_NAME) {
        const command = (block.input?.command as string) || ''
        if (EXTERNAL_COMMAND_PATTERNS.some(p => p.test(command))) {
          return false
        }
      }
    }
  }
  return true
}
```

#### 摩擦信号检测

```typescript
export function hasFrictionSignal(messages: Message[]): boolean {
  // 从最新消息开始向前扫描
  for (let i = messages.length - 1; i >= 0; i--) {
    const msg = messages[i]!
    if (msg.type !== 'user') continue
    
    const text = getUserMessageText(msg)
    if (!text) continue
    
    return FRICTION_PATTERNS.some(p => p.test(text))
  }
  return false
}
```

#### 主钩子逻辑

```typescript
export function useIssueFlagBanner(messages: Message[], submitCount: number): boolean {
  // 仅限内部用户
  if (process.env.USER_TYPE !== 'ant') return false
  
  // 使用 ref 跟踪触发状态
  const lastTriggeredAtRef = useRef(0)
  const activeForSubmitRef = useRef(-1)
  
  // 使用 useMemo 缓存 O(n) 扫描结果
  const shouldTrigger = useMemo(
    () => isSessionContainerCompatible(messages) && hasFrictionSignal(messages),
    [messages]
  )
  
  // 保持横幅显示直到下次提交
  if (activeForSubmitRef.current === submitCount) return true
  
  // 检查冷却期
  if (Date.now() - lastTriggeredAtRef.current < COOLDOWN_MS) return false
  
  // 检查最小提交次数
  if (submitCount < MIN_SUBMIT_COUNT) return false
  
  // 检查触发条件
  if (!shouldTrigger) return false
  
  // 记录触发状态
  lastTriggeredAtRef.current = Date.now()
  activeForSubmitRef.current = submitCount
  return true
}
```

---

## 关键代码路径与文件引用

```
src/hooks/useIssueFlagBanner.ts
├── EXTERNAL_COMMAND_PATTERNS      # 行 6-25: 外部命令正则
├── FRICTION_PATTERNS              # 行 27-43: 摩擦信号正则
├── isSessionContainerCompatible() # 行 45-72: 容器兼容性检查
├── hasFrictionSignal()            # 行 74-87: 摩擦信号检测
└── useIssueFlagBanner()           # 行 92-133: 主钩子
    ├── 用户类型检查               # 行 96-97
    ├── useMemo 缓存              # 行 110-113
    ├── 冷却期检查                 # 行 120-121
    └── 触发条件判断               # 行 123-132
```

### 依赖文件

```
src/tools/BashTool/toolName.js     # BASH_TOOL_NAME 常量
src/types/message.js               # Message 类型
src/utils/messages.js              # getUserMessageText 函数
```

---

## 依赖与外部交互

### React Hooks 使用

- `useMemo`: 缓存 `isSessionContainerCompatible` 和 `hasFrictionSignal` 的计算结果
- `useRef`: 跟踪 `lastTriggeredAt` 和 `activeForSubmit` 状态

### 性能优化

- **消息扫描优化**：`messages` 在打字期间是稳定的，避免频繁重新计算
- **O(n) 扫描缓存**：`isSessionContainerCompatible` 需要遍历所有消息和工具调用，使用 useMemo 缓存
- **反向扫描**：`hasFrictionSignal` 从最新消息开始扫描，通常能快速匹配

---

## 风险、边界与改进建议

### 已知风险

1. **误报风险**
   - 正则表达式可能匹配到非摩擦语境的文本
   - 例如："No problem" 会被 `/^no[,!]\s/i` 匹配
   - 缓解：模式设计已考虑，如 "No," 或 "No!" 才匹配

2. **漏报风险**
   - 用户可能用非标准方式表达不满
   - 非英语摩擦信号无法检测

3. **隐私考虑**
   - 虽然仅限内部用户，但仍涉及用户输入分析
   - 需要确保符合隐私政策

### 边界情况

| 场景 | 行为 |
|-----|------|
| 非 ant 用户 | 始终返回 false |
| 消息为空 | 无摩擦信号，容器兼容 |
| 提交次数不足 | 不触发 |
| 冷却期内 | 不触发 |
| 同时满足多个条件 | 正常触发一次 |

### 改进建议

1. **多语言支持**
   - 添加其他语言的摩擦信号检测
   - 使用更智能的 NLP 模型替代正则

2. **上下文感知**
   - 考虑对话上下文，减少误报
   - 例如："No problem" vs "No, that's wrong"

3. **可配置性**
   - 允许用户关闭此功能（即使是 ant 用户）
   - 提供配置界面调整敏感度

4. **更精确的容器检测**
   - 当前仅检测 Bash 工具和 MCP 工具
   - 可以扩展到其他可能的外部调用

5. **遥测集成**
   - 触发时发送匿名遥测数据
   - 帮助改进模型和用户体验

### 测试建议

1. **正则测试**：
   - 每个 FRICTION_PATTERNS 的匹配/不匹配用例
   - 边界情况（大小写、标点）

2. **集成测试**：
   - 完整的消息流触发测试
   - 冷却期行为测试

3. **性能测试**：
   - 大量消息时的扫描性能
   - 确保不会阻塞渲染
