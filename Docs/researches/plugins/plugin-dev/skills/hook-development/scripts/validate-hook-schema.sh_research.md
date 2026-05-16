# FILE 研究：plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh

## 场景与职责

`validate-hook-schema.sh` 是 hook 配置层面的校验器，目标是在 hook 注册到 Claude 会话前尽早发现结构错误、字段缺失和明显风险配置。

在工具链中的定位：

- 前置于运行时测试：先校验配置，再回放脚本
- 与 `test-hook.sh`、`hook-linter.sh` 形成“三段式”验证

典型调用来源：

- `plugins/plugin-dev/skills/hook-development/scripts/README.md:5-27,120-123`
- `plugins/plugin-dev/skills/hook-development/SKILL.md:707`
- `plugins/plugin-dev/commands/create-plugin.md:258`
- `plugins/plugin-dev/agents/plugin-validator.md:107-114`

## 功能点目的

按实现，脚本希望覆盖如下质量门禁：

1. JSON 语法正确（`30-37`）
2. 顶层事件名是否在允许集合（`41-56`）
3. 每个事件条目具备 `matcher/hooks`（`69-83`）
4. hook 元素具备 `type` 且仅 `command|prompt`（`89-101`）
5. `command` 或 `prompt` 类型字段存在（`103-128`）
6. `timeout` 为数字且范围合理（`130-142`）
7. 对硬编码绝对路径/事件兼容性给 warning（`111-113,124-126`）

目标是把“启动后才爆错”前移到“提交前可见”。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 主流程

1. 参数校验 + 文件存在性检查（`8-25`）
2. `jq empty` 语法校验（`30-35`）
3. 顶层 key 遍历与事件名检查（`43-55`）
4. 三层循环校验：`event -> item -> hooks[]`（`65-146`）
5. 根据 `error_count/warning_count` 输出总结（`148-159`）

### 2) 数据结构假设

当前实现假设输入是 direct 结构：

```json
{
  "PreToolUse": [
    {
      "matcher": "Write|Edit",
      "hooks": [
        { "type": "command", "command": "bash ...", "timeout": 30 }
      ]
    }
  ]
}
```

关键依据：

- 两次使用 `jq -r 'keys[]'` 直接枚举根键（`43,65`）
- 后续索引路径均为 `."$event"[...]`（`66,70,78,86,89,...`）

### 3) 实测行为

A. 对插件常见 wrapper 结构不兼容：

- 运行：

```bash
bash plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh \
  plugins/security-guidance/hooks/hooks.json
```

- 现象：先把 `description`、`hooks` 当作“未知事件”，随后在数组索引阶段触发：
`jq: Cannot index string with number`

B. warning/error 计数逻辑会被 `set -e` 提前中断：

- 当首次触发 `((warning_count++))` 或 `((error_count++))` 时，脚本直接提前退出，无法输出汇总段。
- 复现 1（warning）：构造绝对路径 command，脚本打印 warning 后退出码为 1，未进入 `148-159` 总结分支。
- 复现 2（error）：构造缺失 `matcher` 的配置，打印首个错误后直接退出。

## 关键代码路径与文件引用

核心实现：

- `plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh:1-159`

格式与调用上下文：

- `plugins/plugin-dev/skills/hook-development/scripts/README.md:5-27,120-123`
- `plugins/plugin-dev/skills/hook-development/SKILL.md:64-80`（wrapper 格式说明）
- `plugins/plugin-dev/skills/hook-development/SKILL.md:102-119`（settings direct 格式说明）
- `plugins/plugin-dev/commands/create-plugin.md:257-260`
- `plugins/plugin-dev/agents/plugin-validator.md:107-114`

真实插件样本：

- `plugins/security-guidance/hooks/hooks.json:1-16`
- `plugins/learning-output-style/hooks/hooks.json:1-15`
- `plugins/hookify/hooks/hooks.json:1-49`

这些样本均使用 wrapper 根结构（`description` + `hooks`）。

## 依赖与外部交互

运行依赖：

- `bash`
- `jq`

交互边界：

- 输入：本地 `hooks.json` 文件路径
- 输出：stdout 校验结果
- 无网络调用、无外部 API

副作用：

- 无写文件操作（纯读 + 输出）

## 风险、边界与改进建议

1. 配置格式兼容性风险（高）。
- 技能文档主推插件 wrapper 格式，但脚本按 direct 根结构解析；对真实插件文件会误报/异常退出。

2. 计数器与 `set -e` 组合导致流程中断（高）。
- 设计目标是“累计 error/warning 后统一汇总”，实际常在首个问题处退出，丢失完整诊断。

3. 错误处理与用户体验不足（中）。
- `jq` 索引异常直接暴露内部错误信息，缺少“输入格式不支持”的友好解释。

4. 校验规则与生态样本不一致（中）。
- `matcher` 被强制必填（`69-75`），但仓库中多个插件配置条目省略 matcher（依赖默认匹配行为）。

建议：

- 先统一输入根：
  - 若存在 `.hooks`，则下钻后校验；否则按 direct 根校验。
- 修复计数器：把 `((warning_count++))` 改为 `((warning_count+=1))` 或追加 `|| true`，避免 `set -e` 提前退出。
- 为结构不匹配增加显式错误：例如“detected wrapper/direct mismatch”。
- 兼容 `matcher` 可选场景，或在文档中明确“该脚本是严格模式校验器”。
