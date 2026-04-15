<!-- Synced with README.md as of 2026-04-15 -->

[English](README.md) | [中文](README.zh.md)

# Cognitive Kernel

**规则不会自己执行。**

一个认知底座，30 行规则，写入 `CLAUDE.md`——能工作。Agent 的思维方式变了，你从输出里就能看出来。

现在装 19 个。720 多行规则，100 多条指令，全部挤在同一个上下文窗口里争夺 Agent 的注意力。结果？跟你在墙上贴 100 张检查清单一样——没人看任何一张。Agent 的注意力被稀释，关键规则被跳过。认知底座技术上"已加载"，实际上形同虚设。

## 为什么文字规则不管用

在配置文件里写"你必须做 X"是最弱的干预方式。完全依赖 Agent 在正确的时刻、在认知负荷下注意到这条规则。这就像指望工人在操作重型机械时阅读安全标语。

6 层干预模型用结构替代祈祷。不是靠一种机制（文件里的文字），而是让每条规则通过最强的可用渠道生效：

| 层级 | 机制 | 强度 | 怎么工作 |
|------|------|------|---------|
| **L1** | 输出模板 | 最强 | Agent 的回复必须包含特定字段——不填就不完整 |
| **L2** | Hook | 强 | 平台 hook 在关键动作前自动注入提醒——Agent 被拦截 |
| **L3** | If-Then 触发器 | 中 | 模式匹配规则自动触发——不靠自觉 |
| **L4** | 外部验证 | 强 | 另一个子 Agent 做对抗性审查——自己查自己靠不住 |
| **L5** | 核心规则 | 弱 | 少量关键规则常驻上下文——始终可见，但仍是自愿 |
| **L6** | Skill 参考 | 最弱 | 完整框架按需加载——Agent 得自己选择去读 |

**核心洞察：** L1 和 L2 是结构性的——Agent 不合规就无法产出有效输出。L5 和 L6 是自愿性的——Agent 在压力下可以忽略。Kernel 把每条规则放到它能占据的最强层级。

## 三件事

Cognitive Kernel 做且只做三件事：

### 1. 冲突分析

安装新底座时，kernel 拿它的 manifest 跟所有已安装底座逐一比对：

- **L1 重叠**——两个底座要求类似的输出字段？（比如都要"假设审计"字段 → 合并或排优先级）
- **L3 张力**——触发器指向相反方向？（比如"拆到部件"vs"先看整体" → 定义触发顺序）
- **L5 冲突**——核心规则矛盾？（比如"快速行动"vs"先验证" → 辩证调和）

Kernel 报告重叠、张力和冲突，提出解决方案。你来决定。

### 2. 预算管理

上下文窗口是有限的。规则越多 ≠ 思考越好。Kernel 执行预算：

- **L5 核心规则**：最多 4 个底座同时激活（约 12 条规则）。再多 → 注意力稀释。
- **L1 输出字段**：按场景预算（提出方案时 5 个字段，声称完成时 3 个）。防止输出模板膨胀。

当你试图安装第 5 个 L5 底座时，kernel 会警告你，问你把哪个降级到 L6（按需加载）。

### 3. 6 层装配

每个认知底座通过 `manifest.yml` 描述自己——它在每个干预层级需要什么。Kernel 读取所有已安装底座的 manifest，组装运行时：

- L1 字段 → 写入 `~/.cognitive-kernel/cognitive-kernel.md` 作为输出模板标题
- L2 hook → 注册到平台的 hook 系统（如 Claude Code 的 `ft-settings.json`）
- L3 触发器 → 组装为运行时配置中的 if-then 规则
- L4 提示 → 存储为子 Agent 对抗性审查的 brief
- L5 规则 → 按预算选择，注入到常驻上下文
- L6 引用 → 注册为可按需加载的 skill

运行 `/cognitive-kernel regenerate` 可以基于当前注册表状态从头重建运行时。

## 快速开始

### 通过 meta-cogbase（推荐）

如果你安装了 [meta-cogbase](https://github.com/d-wwei/meta-cogbase)，kernel 已经装好了。不需要额外操作。

### 手动安装

1. 将 skill 文件复制到 agent 的 skill 目录：
   ```bash
   mkdir -p ~/.claude/skills/cognitive-kernel
   cp SKILL.md README.md ~/.claude/skills/cognitive-kernel/
   ```

2. 初始化 kernel：
   ```
   /cognitive-kernel setup
   ```

3. 安装认知底座：
   ```
   /cognitive-kernel install /path/to/first-principles
   /cognitive-kernel install /path/to/results-driven
   ```

4. 验证：
   ```
   /cognitive-kernel status    # 查看安装状态和预算使用
   /cognitive-kernel check     # 运行 L1 合规检查
   ```

## 命令

| 命令 | 做什么 |
|------|-------|
| `setup` | 首次初始化——创建 `~/.cognitive-kernel/`、基线配置、hook |
| `install <path>` | 安装底座——读 manifest，跑冲突分析，组装到运行时 |
| `uninstall <name>` | 卸载底座——清理所有层级，重新生成运行时 |
| `status` | 已装底座、L5 预算使用、L1 字段数、张力调解记录 |
| `check` | L1 合规检查——验证 Agent 输出是否包含必填字段 |
| `review` | L4 对抗性审查——派一个子 Agent 挑战当前工作 |
| `monitor` | 查看 hook 触发日志——哪些规则触发了、频率、时间 |
| `optimize` | 数据驱动建议——基于 monitor 数据，推荐预算调整 |
| `regenerate` | 从注册表重建运行时——重新处理所有 manifest，重新组装所有层级 |
| `reorder` | 调整 L1 字段排序——改变输出模板中字段的出现顺序 |

## 给底座作者

每个要和 kernel 协作的认知底座都需要一个 `manifest.yml`：

```yaml
schema_version: 1
name: my-framework
tier: 5                    # 5 = 核心（可贡献 L5 规则），6 = 专用（仅 L6）

output_fields:             # L1: 必填输出字段
  - prompt: "假设审计：[区分已验证事实 vs 未验证惯例]"
    when: proposing_solution

hooks:                     # L2: 自动注入提醒
  - event: before_task_complete
    inject: "我是从基本面重建的，还是只是重新组装了现有答案？"

triggers:                  # L3: 模式匹配规则
  - if: "问题陈述中包含隐含的'应该'"
    then: "在继续之前先指出这个预设"
    action_type: pause_and_check

core_rules:                # L5: 常驻规则（仅 tier 5）
  - rule: "解决任何问题前，先区分已验证事实和未验证惯例"
    rank: 1

activation:                # 最相关的使用场景
  - "在任何领域分析假设"
```

用 [Cognitive Base Creator](https://github.com/d-wwei/cognitive-base-creator) 可以自动生成包含 manifest 的完整底座包。

## 运行时目录

```
~/.cognitive-kernel/
  ├── cognitive-kernel.md       # 组装后的运行时（L1/L3/L4/L5/L6 内容）
  ├── cognitive-registry.yml    # 已安装底座注册表 + 冲突解决记录
  ├── hooks/
  │   └── before-code-change.sh # L2 hook 脚本
  └── logs/
      └── YYYY-MM-DD.jsonl      # Hook 触发事件日志
```

## 支持的平台

| 平台 | L2 Hook 支持 | 方式 |
|------|-------------|------|
| Claude Code | 完整支持 | `ft-settings.json` PreToolUse hook |
| Gemini CLI | 降级（L2→L3） | `GEMINI.md` @ 引用 |
| Cursor | 降级（L2→L3） | `.cursorrules` / `.mdc` 文件 |
| Codex CLI | 降级（L2→L3） | `AGENTS.md` inline 注入 |
| OpenCode | 降级（L2→L3） | `AGENTS.md` inline 注入 |
| OpenClaw | 降级（L2→L3） | `AGENTS.md` inline 注入 |

不支持 hook 的平台自动将 L2 内容降级为 L3 触发器——规则仍然触发，只是通过模式匹配而非事件拦截。

## 许可证

MIT

## 链接

- [meta-cogbase](https://github.com/d-wwei/meta-cogbase) — 认知底座包管理器（包含 kernel）
- [Cognitive Base Creator](https://github.com/d-wwei/cognitive-base-creator) — 从任意思维框架生成新底座
- [文章：所有 Agent 都缺了一层](https://github.com/d-wwei/cognitive-base-creator/blob/main/docs/article-zh.md) — 深度解析认知底座为什么重要
