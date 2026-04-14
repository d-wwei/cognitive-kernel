# Cognitive Kernel

认知底座的安装器、组装器和运行时工具。6 层干预框架让认知底座从"请记得遵守"升级为"结构性强制"。

## 问题

19 个认知底座全文注入 CLAUDE.md（720+ 行 / 100+ 规则），导致注意力稀释。Agent 在关键决策时刻无法激活正确的规则。

## 解决方案：6 层干预模型

| Layer | 机制 | 强度 |
|-------|------|------|
| L1 | 输出模板：## heading 必填字段 | 最强 |
| L2 | Hooks：平台 hook 自动注入提醒 | 强 |
| L3 | If-Then 触发器：模式→动作规则 | 中强 |
| L4 | 外部验证：子 Agent 对抗性审查 | 强 |
| L5 | 核心透镜：少量规则常驻 context | 弱 |
| L6 | Skill 参考：按需加载完整底座 | 弱 |

## 安装

### 前提

- Claude Code（或其他支持的 AI 编程工具）
- 认知底座仓库（包含 manifest.yml 的底座目录）

### 步骤

1. **安装内核 skill**：
   将 `cognitive-kernel/` 目录放到 `~/.claude/skills/cognitive-kernel/`

2. **初始化**：
   在 AI agent 会话中执行 `/cognitive-kernel setup`

3. **安装底座**：
   ```
   /cognitive-kernel install /path/to/first-principles
   /cognitive-kernel install /path/to/systems-thinking
   ```

4. **验证**：
   ```
   /cognitive-kernel status    # 查看安装状态
   /cognitive-kernel check     # 合规验证
   ```

## 命令参考

| 命令 | 用途 |
|------|------|
| `/cognitive-kernel setup` | 首次初始化 |
| `/cognitive-kernel install <path>` | 安装底座（含冲突分析） |
| `/cognitive-kernel uninstall <name>` | 卸载底座 |
| `/cognitive-kernel status` | 查看安装状态 + 预算使用 |
| `/cognitive-kernel check` | L1 合规验证（linter） |
| `/cognitive-kernel review` | L4 对抗性审查（子 Agent） |
| `/cognitive-kernel monitor` | 查看运行监控数据 |
| `/cognitive-kernel optimize` | 基于数据的优化建议 |
| `/cognitive-kernel regenerate` | 从 registry 重新生成产物 |
| `/cognitive-kernel reorder` | 调整 L1 字段排序 |

## 支持的平台

| 平台 | L2 Hook 支持 | 配置方式 |
|------|-------------|---------|
| Claude Code | 完整支持 | ft-settings.json PreToolUse hook |
| Cursor | L2→L3 降级 | .cursorrules / .mdc 文件 |
| Gemini CLI | L2→L3 降级 | GEMINI.md @引用 |

不支持 hook 的平台自动将 L2 内容降级为 L3 触发器。

## 底座作者指南

### 使用 cognitive-kernel 的底座

在底座目录中包含 `manifest.yml`：

```yaml
schema_version: 1
name: my-framework
tier: 6                    # 5=核心透镜, 6=按需加载

triggers:
  - if: "触发条件"
    then: "执行动作"

activation:
  - "激活场景描述"
```

使用 `/cognitive-base-creator` 生成底座时，Phase 2.5 会自动从 cognitive-protocol.md 提取候选内容生成 manifest.yml。

### install.sh 双模式

底座的 install.sh 支持双模式：
- 检测到 cognitive-kernel → 建议使用 `/cognitive-kernel install`
- 未检测到 → 走传统独立安装流程

## 产物结构

```
~/.cognitive-kernel/
  ├── cognitive-kernel.md       # 运行时产物（L1/L3/L4/L5/L6）
  ├── cognitive-registry.yml    # 已安装底座注册表
  ├── hooks/
  │   └── before-code-change.sh # L2 hook 脚本
  └── logs/
      └── YYYY-MM-DD.jsonl      # Hook 触发日志
```

## License

MIT
