# verify.ts 深度研究文档

## 场景与职责

`verify.ts` 是 Claude Code CLI 的**内部验证技能模块**，负责实现 `/verify` 命令技能。该技能仅限内部员工（`USER_TYPE === 'ant'`）使用，用于验证代码变更是否按预期工作。

### 核心职责
1. **条件性 Skill 注册**：仅在内部员工环境中注册该技能
2. **动态描述生成**：从 SKILL.md 的 frontmatter 解析技能描述
3. **参考文件管理**：提供示例文档（CLI/Server）供模型按需读取

### 适用场景
- 内部员工需要验证代码变更时
- 运行应用程序测试修改效果
- 验证服务器端变更

---

## 功能点目的

### 1. 内部员工限制
```typescript
if (process.env.USER_TYPE !== 'ant') {
  return
}
```
**目的**：确保该技能仅对 Anthropic 内部员工可见和可用，避免外部用户访问内部验证流程。

### 2. Frontmatter 驱动的描述
```typescript
const { frontmatter, content: SKILL_BODY } = parseFrontmatter(SKILL_MD)
const DESCRIPTION = typeof frontmatter.description === 'string'
  ? frontmatter.description
  : 'Verify a code change does what it should by running the app.'
```
**目的**：允许通过 SKILL.md 的 frontmatter 动态配置技能描述，无需修改代码即可更新描述。

### 3. 参考文件提取
```typescript
files: SKILL_FILES,  // { 'examples/cli.md': cliMd, 'examples/server.md': serverMd }
```
**目的**：将示例文档作为参考文件提供给模型，模型可以按需 Read 这些文件获取详细验证指南。

---

## 具体技术实现

### 关键流程

#### 1. 模块加载与解析流程
```
import { parseFrontmatter } from '../../utils/frontmatterParser.js'
import { SKILL_FILES, SKILL_MD } from './verifyContent.js'

// 构建时：Bun text loader 将 .md 文件内联为字符串
// 运行时：解析 frontmatter 获取描述和正文
const { frontmatter, content: SKILL_BODY } = parseFrontmatter(SKILL_MD)
```

#### 2. Skill 注册流程
```typescript
export function registerVerifySkill(): void {
  // 1. 检查用户类型
  if (process.env.USER_TYPE !== 'ant') return
  
  // 2. 解析描述
  const DESCRIPTION = ...
  
  // 3. 注册到 bundled skills 系统
  registerBundledSkill({
    name: 'verify',
    description: DESCRIPTION,
    userInvocable: true,
    files: SKILL_FILES,
    getPromptForCommand: async (args) => {
      const parts: string[] = [SKILL_BODY.trimStart()]
      if (args) {
        parts.push(`## User Request\n\n${args}`)
      }
      return [{ type: 'text', text: parts.join('\n\n') }]
    },
  })
}
```

#### 3. 提示生成逻辑
```
prompt = SKILL_BODY.trimStart()
if (args provided):
  prompt += "\n\n## User Request\n\n" + args
```

### 数据结构

#### FrontmatterData（来自 frontmatterParser.ts）
```typescript
type FrontmatterData = {
  description?: string | null
  'allowed-tools'?: string | string[] | null
  model?: string | null
  'user-invocable'?: string | null
  // ... 其他字段
}
```

#### ParsedMarkdown（来自 frontmatterParser.ts）
```typescript
type ParsedMarkdown = {
  frontmatter: FrontmatterData
  content: string
}
```

#### SKILL_FILES（来自 verifyContent.ts）
```typescript
export const SKILL_FILES: Record<string, string> = {
  'examples/cli.md': cliMd,      // CLI 验证示例
  'examples/server.md': serverMd, // 服务器验证示例
}
```

### 关键代码路径

| 功能 | 文件路径 |
|------|----------|
| Skill 注册入口 | `src/skills/bundled/verify.ts:12-30` |
| Frontmatter 解析 | `src/skills/bundled/verify.ts:5` |
| 内容导入 | `src/skills/bundled/verify.ts:3` |
| 提示生成 | `src/skills/bundled/verify.ts:22-28` |
| 内容资产定义 | `src/skills/bundled/verifyContent.ts:1-13` |
| Frontmatter 解析器 | `src/utils/frontmatterParser.ts:130-175` |
| BundledSkill 注册器 | `src/skills/bundledSkills.ts:53-100` |

---

## 依赖与外部交互

### 直接依赖

| 模块 | 路径 | 用途 |
|------|------|------|
| `parseFrontmatter` | `../../utils/frontmatterParser.js` | 解析 SKILL.md 的 frontmatter |
| `registerBundledSkill` | `../bundledSkills.js` | 注册 bundled skill 到系统 |
| `SKILL_FILES` | `./verifyContent.js` | 参考文件内容映射 |
| `SKILL_MD` | `./verifyContent.js` | SKILL.md 文件内容 |

### 间接依赖

| 模块 | 路径 | 用途 |
|------|------|------|
| `parseYaml` | `src/utils/yaml.ts` | Frontmatter 中的 YAML 解析 |
| `FRONTMATTER_REGEX` | `src/utils/frontmatterParser.ts:123` | Frontmatter 分隔符匹配 |

### 外部交互
- **构建时**：Bun 的 text loader 将 `.md` 文件内联为字符串常量
- **运行时**：无网络调用，所有内容已内联在二进制中
- **文件提取**：首次调用时，`bundledSkills.ts` 将 `files` 中的内容提取到临时目录

---

## 风险、边界与改进建议

### 风险点

1. **构建时依赖**
   - `verifyContent.ts` 导入的 `.md` 文件必须在构建时存在
   - 如果 `src/skills/bundled/verify/` 目录不存在或缺少文件，构建会失败
   - **现状**：源代码仓库中该目录可能不存在（仅构建环境存在）

2. **硬编码的用户类型检查**
   ```typescript
   if (process.env.USER_TYPE !== 'ant')
   ```
   - 使用字符串字面量比较，无类型安全
   - 如果 `USER_TYPE` 值变更，需要同步修改多处代码

3. **Frontmatter 解析失败降级**
   ```typescript
   const DESCRIPTION = typeof frontmatter.description === 'string'
     ? frontmatter.description
     : 'Verify a code change does what it should by running the app.'
   ```
   - 当 frontmatter 解析失败或 description 不存在时，使用硬编码默认值
   - 无日志记录 frontmatter 解析问题

4. **参考文件路径硬编码**
   ```typescript
   'examples/cli.md': cliMd,
   'examples/server.md': serverMd,
   ```
   - 文件路径在 `verifyContent.ts` 中硬编码
   - 新增示例需要修改代码并重新构建

### 边界条件

1. **空参数处理**
   ```typescript
   if (args) {
     parts.push(`## User Request\n\n${args}`)
   }
   ```
   当 args 为空字符串时，不添加用户请求部分

2. **SKILL_BODY 处理**
   ```typescript
   parts: string[] = [SKILL_BODY.trimStart()]
   ```
   使用 `trimStart()` 移除前导空白，确保提示格式整洁

3. **files 字段可选性**
   - `files: SKILL_FILES` 是可选的
   - 如果 `SKILL_FILES` 为空对象，bundledSkills 系统会跳过文件提取

### 改进建议

1. **类型安全的用户类型检查**
   ```typescript
   // 建议添加常量定义
   const enum UserType {
     INTERNAL = 'ant',
     EXTERNAL = 'external',
   }
   
   if (process.env.USER_TYPE !== UserType.INTERNAL) {
     return
   }
   ```

2. **Frontmatter 解析错误日志**
   ```typescript
   // 建议添加错误处理
   const { frontmatter, content: SKILL_BODY } = parseFrontmatter(SKILL_MD)
   if (!frontmatter.description) {
     logForDebugging('verify skill: no description in frontmatter, using default')
   }
   ```

3. **动态示例文件加载**
   - 考虑使用 glob 模式动态加载 `verify/examples/` 目录下的所有 `.md` 文件
   - 避免每次新增示例都需要修改 `verifyContent.ts`

4. **Skill 可用性日志**
   ```typescript
   // 建议添加
   if (process.env.USER_TYPE !== 'ant') {
     logForDebugging('verify skill: skipped (not internal user)')
     return
   }
   logForDebugging('verify skill: registered for internal user')
   ```

5. **文档缺失处理**
   - 当前假设 `SKILL_MD` 和 `SKILL_FILES` 始终存在
   - 建议添加运行时检查，如果内容为空则跳过注册并记录警告
