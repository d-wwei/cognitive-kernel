---
name: cognitive-kernel
description: "认知底座的安装器、组装器和运行时工具。6 层干预框架保障认知底座真正发挥作用。"
---

# Cognitive Kernel

认知底座的安装、组装与运行时框架。

**命令**：
- `/cognitive-kernel setup` — 首次初始化
- `/cognitive-kernel install <path>` — 安装认知底座
- `/cognitive-kernel uninstall <name>` — 卸载认知底座
- `/cognitive-kernel status` — 查看安装状态
- `/cognitive-kernel regenerate` — 重新组装运行时产物
- `/cognitive-kernel check` — 合规验证（linter）
- `/cognitive-kernel review` — 对抗性审查（L4 子 Agent）
- `/cognitive-kernel reorder` — 调整 L1 字段排序
- `/cognitive-kernel monitor` — 查看运行监控数据（Phase 3）
- `/cognitive-kernel optimize` — 分析报告 + 优化建议（Phase 3）
- `/cognitive-kernel all-in` — 全底座分析：调动所有认知底座多维分析一个问题（别名：`/Cognitive-All-In`、`/认知全开`）

---

## 1. Setup（首次初始化）

当用户执行 `/cognitive-kernel setup` 或首次 install 检测到未初始化时执行。

**步骤**：
1. 创建目录结构：
   ```
   ~/.cognitive-kernel/
     hooks/
     logs/
     cognitive-kernel.md
     cognitive-registry.yml
   ```
2. 将 §A（基线内容）写入 `~/.cognitive-kernel/cognitive-kernel.md`
3. 创建空的 `~/.cognitive-kernel/cognitive-registry.yml`：
   ```yaml
   version: 1
   bases: []
   config:
     l5_budget:
       recommended: 4
       hard_max: 6
       max_rules_per_base: 3
     output_budget:
       proposing_solution: 5
       proposing_change: 5
       claiming_done: 3
       always: 3
   ```
4. 将 hook 脚本复制到 `~/.cognitive-kernel/hooks/`（如果尚未存在）
5. 在 `~/.claude/ft-settings.json` 中添加 hook 配置（见 §B）（注意：Claude Code 用户级配置是 ft-settings.json，不是 settings.json）
6. 确保 `~/.claude/CLAUDE.md` 中包含 `@~/.cognitive-kernel/cognitive-kernel.md`
7. 报告完成状态

---

## 2. Install（安装底座）

当用户执行 `/cognitive-kernel install <path>` 时执行。

**输入**：底座目录路径（包含 manifest.yml）

**步骤**：

### 2.1 读取 manifest
1. 读取 `<path>/manifest.yml`
2. 验证 schema_version 兼容（当前支持 v1）
3. 验证必填字段存在（schema_version, name, tier）

### 2.2 检查是否已安装
1. 读取 `~/.cognitive-kernel/cognitive-registry.yml`
2. 如果同名底座已存在 → 提示："已安装，要更新还是跳过？"

### 2.3 冲突分析

将新底座的 manifest 与所有已安装底座的 manifest 进行语义比对。产出三类结果：重叠、张力、冲突。

**执行步骤**：

1. 读取新底座的 manifest.yml
2. 读取 registry 中每个已安装底座的 manifest 路径，逐个读取其 manifest.yml
3. 读取基线内容（§A），作为"内核基线"参与比对
4. 按以下三个维度逐一比对，收集所有发现

**维度 1: L1 output_fields 重叠检查**

对新底座的每个 output_field，与同 `when` 分组下的所有现有字段（含基线）做语义比对：
- 判断标准：两个 prompt 是否在要求 Agent 思考**同一件事**（即使措辞不同）
- 示例重叠："这会破坏什么？"（基线）vs "列出所有受影响的系统"（某底座）→ 语义重叠
- 示例非重叠："这会破坏什么？" vs "给出置信度"→ 不同维度

对每对重叠，记录：`{new_base, new_field, existing_base, existing_field, similarity: "高/中"}`

**维度 2: L3 triggers 冲突检查**

对新底座的每个 trigger，与所有现有 triggers 比对：
- 重叠：`if` 条件相似且 `then` 动作相似 → 纯重复，标记跳过
- 张力：`if` 条件相似但 `then` 动作不同但互补 → 辩证张力，生成调解规则
- 冲突：`if` 条件相似但 `then` 动作矛盾 → 真正冲突，需用户选择

**维度 3: L5 core_rules 张力检查**（仅 tier-5 底座）

将新底座的 core_rules 与所有已装 tier-5 底座的 core_rules 比对：
- 检查是否存在辩证关系（如：bayesian 的概率更新 vs conviction-override 的信念驱动）
- 张力 ≠ 冲突：辩证张力是有价值的，需要调解规则而非二选一

**调解规则生成**（张力发现时）：

对每对张力，生成一条调解规则，格式：
```
当 [底座A 的规则] 与 [底座B 的规则] 同时适用时：
默认走 [底座A]（因为 [适用场景]），
仅当 [具体条件] 满足时走 [底座B]。
```
写入 registry 的 `tension_mediations` 数组，regenerate 时合并到 kernel.md 的"已知张力"section。

**输出报告**：

比对完成后，向用户展示完整报告：

```
冲突分析报告（新底座: <name>）

[重叠] L1: "<new_field>" 与 <existing_base> 的 "<existing_field>" 语义相近
  → 选择：(1) 保留两个  (2) 合并为一个  (3) 跳过新的

[张力] L5: "<new_rule>" 与 <existing_base> 的 "<existing_rule>" 存在辩证关系
  → 已生成调解规则: "<mediation_text>"
  → 确认：(1) 接受  (2) 修改调解规则  (3) 忽略张力

[冲突] L3: trigger "<new_if> → <new_then>" 与 <existing_base> 的 "<existing_if> → <existing_then>" 矛盾
  → 选择：(1) 保留新的  (2) 保留旧的  (3) 同时保留（由用户自行管理）

无发现 → "未检测到冲突或张力，继续安装。"
```

等待用户对每项做出选择，将决策记录到 registry 的 `conflict_resolutions`：
```yaml
conflict_resolutions:
  - type: overlap | tension | conflict
    new_base: <name>
    existing_base: <name>
    layer: L1 | L3 | L5
    resolution: skip | merge | keep_both | keep_new | keep_existing | mediation
    detail: "<具体决策描述>"
    mediation_rule: "<调解规则文本>"  # 仅张力类型
```

### 2.4 预算检查（仅 tier-5）
1. 统计当前 L5 已有底座数量
2. 如果 < recommended → 直接纳入
3. 如果 = recommended 且 < hard_max → 警告 "超出推荐预算，可能注意力稀释"，提供选项：
   a. 将新底座降为 tier 6
   b. 选择一个现有 L5 底座降级
   c. 强制加入（接受风险）
4. 如果 = hard_max → 拒绝，必须先降级一个现有底座

### 2.5 L1 预算检查
1. 统计各 when 分组的现有字段数（含基线）
2. 新底座的 output_fields 加入后是否超出 output_budget
3. 超出 → 提示用户选择保留哪些字段

### 2.6 注册
将底座信息写入 registry：
```yaml
bases:
  - name: <name>
    tier: <final tier after budget negotiation>
    tier_mode: full | partial    # tier-5 only
    path: <manifest path>
    installed_at: <ISO timestamp>
    manifest: <complete manifest content>
    conflict_resolutions: [...]   # from 2.3
```

### 2.7 安装为 Skill
如果底座目录包含 SKILL.md：
```bash
mkdir -p ~/.claude/skills/<name>
cp <path>/SKILL.md ~/.claude/skills/<name>/
cp <path>/cognitive-protocol.md ~/.claude/skills/<name>/
```

### 2.8 重新组装
执行 §3（Regenerate）生成新的 cognitive-kernel.md。

### 2.9 报告
```
安装完成：<name>
  Tier: <tier> (<full/partial>)
  L1 贡献: <N> 个输出字段
  L3 贡献: <N> 个触发器
  L5 贡献: <N> 条核心规则（仅 tier-5）
  冲突: <N> 个已解决
  张力调解: <N> 条已生成
```

---

## 3. Regenerate（重新组装）

从 registry 重新生成 `~/.cognitive-kernel/cognitive-kernel.md`。

**步骤**：
1. 读取 `~/.cognitive-kernel/cognitive-registry.yml`
2. 按以下模板生成 cognitive-kernel.md：

```markdown
# Cognitive Kernel — 运行时配置
# 此文件由 cognitive-kernel 自动生成，请勿手动编辑
# 最后生成时间: <ISO timestamp>

## 核心原则

- 实事求是：一切表述必须与实际情况相符，不夸大、不模糊、不隐瞒
- 没有调研就没有发言权：回答问题基于实际调查，不进行猜测。如果没有验证过，明确说明"我没有验证"

## Level 1: 输出协议

提出方案或建议时，回复中必须包含以下部分（不可省略，空着不写视为违规）：

### 提出方案时（proposing_solution）

{基线 L1 字段}
{已安装底座在 proposing_solution 下的 output_fields，按安装顺序}

### 提出修改时（proposing_change）

{基线 + 底座 output_fields for proposing_change}

### 声称完成时（claiming_done）

{基线 + 底座 output_fields for claiming_done}

### 始终（always）

{底座 output_fields for always，基线无此分组}

## Level 3: 行为触发器

以下规则在匹配到对应场景时必须执行：

### 基线触发器
{基线 L3 triggers，逐条列出}

### 底座触发器
{每个已安装底座的 triggers，标注来源底座名}

## Level 4: 外部验证协议

### 自动触发条件
方案涉及以下情况时，必须 spawn 子 Agent 做对抗性审查：
- 修改涉及 5 个以上文件
- 变更公共接口或 API
- 架构级改动（新建子系统、改变数据流）
{已安装底座的 verification.condition，列出触发条件}

### 审查 brief 模板
审查子 Agent 收到以下指令：
"你是对抗性审查员。唯一职责是找出方案中的问题。"

基线审查项（始终执行）：
1. 这是选了"正确的路"还是"容易的路"？有没有更难但更对的替代方案？
2. 这会破坏什么？对当前视野之外的部分有什么影响？
3. 从一年后回看，这是最佳选择吗？有没有在积累技术债？

领域审查项（按条件触发）：
{匹配的底座 verification.review_prompt}

## Level 5: 核心透镜

以下规则常驻，始终影响思维过程：

{每个 tier-5(full) 底座的全部 core_rules}
{每个 tier-5(partial) 底座的 rank=1 core_rule}

## 已知张力

以下底座之间存在辩证张力，按此调解：

{registry 中的 tension_mediations}

## Level 6: 可用认知技能

以下底座可通过 skill 按需调用：

| 底座 | 激活场景 |
|------|---------|
{每个 tier-6 底座的 name 和 activation}
{每个 tier-5(partial) 底座也列出，标注"core rule 常驻，完整内容可通过 skill 调用"}
```

3. 写入 `~/.cognitive-kernel/cognitive-kernel.md`
4. 更新 hooks 配置（如果底座声明了新的 hook events）

---

## 4. Uninstall（卸载底座）

当用户执行 `/cognitive-kernel uninstall <name>` 时执行。

**步骤**：
1. 读取 registry，查找该底座
2. 展示将要移除的内容：
   ```
   将要移除 <name>：
     L1: <N> 个输出字段
     L3: <N> 个触发器
     L5: <N> 条核心规则（如果 tier-5）
     L6: 从技能参考表移除
   ```
3. 检查 recommends 关系：
   - 有其他底座 recommends 这个底座 → 提示："以下底座推荐与 <name> 配合使用: [list]。卸载可能降低它们的效果。"
4. 用户确认 → 从 registry 移除 → 重新组装（§3）
5. 可选：移除 skill 文件
   ```
   是否同时移除 skill 文件？
   ~/.claude/skills/<name>/
   [Y/n]
   ```

---

## 5. Status（状态查看）

当用户执行 `/cognitive-kernel status` 时执行。

**输出**：
```
Cognitive Kernel 状态

已安装底座: <N> 个
  Tier 5 (full):    <list> (<N>/<recommended> 推荐上限)
  Tier 5 (partial): <list>
  Tier 6:           <list>

L5 预算: <used>/<recommended> (硬上限 <hard_max>)

L1 字段统计:
  proposing_solution: <N>/<budget>
  proposing_change:   <N>/<budget>
  claiming_done:      <N>/<budget>

L2 Hooks: <N> 个已配置
L3 Triggers: <N> 条（基线 <N> + 底座 <N>）
L4 审查条件: <N> 个

已知张力调解: <N> 条

最后组装时间: <timestamp>
```

---

## 6. Check（合规验证 — Linter）

当用户执行 `/cognitive-kernel check` 时执行。

**定位**：客观合规验证，不做主观质量判断。检查 L1 heading 存在性和内容非空性，类似代码 linter。

**执行步骤**：

### 6.1 确定检查范围

回顾当前对话中 agent（自己）最近的回复。确定回复属于哪个场景：

| 场景 | 判定条件 |
|------|---------|
| proposing（提出方案/修改） | 回复中包含方案、建议、设计决策、代码修改 |
| claiming_done（声称完成） | 回复中声称任务完成、已修复、已实现 |
| 两者皆是 | 回复中既有方案又声称完成 → 两组都检查 |
| 都不是 | 回复是纯问答/信息查找/探索 → 报告"当前回复不需要 L1 合规" |

### 6.2 检查必填 heading

**proposing 场景下必须出现的 `##` heading**（从 cognitive-kernel.md Level 1 的输出模板读取，包含基线字段和结构性 heading）：

1. `## 方案` — 方案内容本体（结构性 heading，非 registry output_field）
2. `## 为什么选这个而不是更难但可能更对的路？` — 基线 output_field，痛点1覆盖
3. `## 这会破坏什么？` — 基线 output_field，痛点2覆盖
4. `## 一年后回看` — 基线 output_field，痛点3覆盖
5. `## 假设审计` — 底座贡献 output_field（来自已安装底座）

**claiming_done 场景下必须出现的 `##` heading**：

6. `## 完成证据` — 逐项核对原始需求

对每个必填 heading，检查：
- **存在性**：该 heading（或语义等价的变体）是否出现在回复中
- **非空性**：heading 下方是否有实质内容（≥ 15 字，排除以下敷衍模式）

**敷衍模式检测**（内容匹配以下模式视为不合规）：
- 纯占位符："无"、"N/A"、"同上"、"不适用"、"暂无"、"None"
- 单句重复 heading：内容只是把 heading 问题重述一遍
- 过于笼统："没有影响"、"不会破坏任何东西"（没有说明检查了什么）

### 6.3 输出合规报告

```
L1 合规检查

场景: proposing（提出方案/修改）
  ✓ "方案" — 存在，内容 <N> 字
  ✓ "为什么选这个而不是更难的路？" — 存在，内容实质
  ✗ "这会破坏什么？" — 未出现
  ✓ "一年后回看" — 存在，内容实质
  ✗ "假设审计" — 存在但内容为"无"（敷衍）

合规率: 3/5 (60%)
缺失: "这会破坏什么？"（建议补充受影响范围分析）
敷衍: "假设审计"（建议列出具体假设而非写"无"）
```

### 6.4 L3 触发器检查（可选扩展）

如果用户执行 `/cognitive-kernel check --full`，额外执行 L3 触发器回溯检测：

**Step 1: 扫描回复中的 L3 匹配信号**

读取 cognitive-kernel.md 的 L3 基线触发器和底座触发器，逐条检查：
- 该触发器的 `if` 条件是否在回复中匹配？
- 如果匹配，对应的 `then` 行为是否在回复中体现？

**Step 2: 报告应触发但未执行的触发器**

```
L3 触发器回溯检测：

  [匹配] 基线 #2: "有两个方案且一个更简单"
    条件匹配: 回复中比较了方案 A 和 B，其中 B 更简单
    行为检测: ✗ 未解释"为什么简单方案不是捷径"
    → 触发器应激活但未执行

  [匹配] 基线 #3: "写了'应该可以'"
    条件匹配: 回复中出现 "应该可以直接用"
    行为检测: ✗ 未替换为具体验证步骤
    → 触发器应激活但未执行

  [未匹配] 基线 #1: "声称完成" — 回复未声称完成，不适用
  ...

L3 回溯得分: 1/3 应触发的触发器被正确执行
```

**重点检测的基线触发器**（最常匹配的 5 条）：
1. 声称完成但无验证证据 → 检查是否有具体的运行输出/测试结果
2. 有多个方案时未解释为什么不选更简单的 → 检查是否有方案比较和选择理由
3. "应该可以"/"should work" 模糊措辞 → 文本扫描
4. 陈述事实但未标注来源（调查 vs 推测）→ 检查结论性陈述是否有出处
5. 模糊措辞（"大概"/"一般来说"）→ 文本扫描

### 6.5 不做的事

- 不判断回答质量好坏（那是 review 的职责）
- 不判断方案是否正确（那是 review 的职责）
- 不做主观评价
- 不修改任何文件
- 不 spawn 子 Agent（check 由主 Agent 自身执行，成本低）

---

## 7. Review（对抗性审查 — L4）

当用户执行 `/cognitive-kernel review` 或自动触发条件满足时执行。

**触发方式**：
- **手动**：用户执行 `/cognitive-kernel review`
- **自动**：方案涉及 5+ 文件修改、变更公共接口、架构级改动
- **建议**：方案涉及 3-4 文件修改或核心业务逻辑 → 提示用户："建议执行对抗性审查，是否继续？"

**执行步骤**：

### 7.1 收集审查上下文

收集以下信息作为审查输入：
- 当前方案/回复的完整内容
- 涉及修改的文件列表（如果是代码修改）
- 原始需求/任务描述（如果可获取）

### 7.2 构造领域审查项

读取 `~/.cognitive-kernel/cognitive-registry.yml`，对每个已安装底座：
1. 读取其 manifest 中的 `verification` 字段
2. 检查每个 `verification.condition` 是否匹配当前方案特征
3. 匹配的 → 将其 `review_prompt` 加入领域审查项列表

匹配规则（Agent 语义判断）：
- condition 描述的场景是否在当前方案中出现
- 例："当方案涉及概率判断或不确定性估计时"→ 如果方案确实涉及不确定性 → 匹配

### 7.3 Spawn 审查子 Agent

使用 Agent 工具 spawn 一个子 Agent，prompt 如下：

```
你是一个对抗性审查员。你的唯一职责是找出方案中的问题。
不要确认方案是好的——你的工作是找问题。如果你找不到问题，
说明你审查得不够深入。

## 被审查方案

{当前方案/回复的完整内容}

## 基线审查项（必须逐项回答）

1. **正确 vs 容易**：这是选了"正确的路"还是"容易的路"？有没有更难但更对的替代方案被忽略了？
2. **破坏分析**：这会破坏什么？列出所有受影响的模块、文件、功能。特别关注不在方案视野内的部分。
3. **长期视角**：从一年后回看，这是最佳选择吗？是否在积累技术债或创造未来的维护负担？

## 领域审查项（按匹配底座附加）

{每个匹配的底座 verification.review_prompt，标注来源底座名}

## 输出格式

对每个审查项，输出：
- **通过** — 该维度没有问题（简述原因）
- **有风险** — 存在可接受的风险（说明具体风险和建议的缓解措施）
- **不通过** — 存在严重问题（说明问题、影响范围、建议的替代方向）

最后给出总体评估：通过 / 有条件通过 / 建议修改
```

### 7.4 展示审查结果

子 Agent 返回结果后，原样展示给用户，格式：

```
═══ 对抗性审查结果 ═══

基线审查:
  1. 正确 vs 容易: [通过/有风险/不通过] — [具体说明]
  2. 破坏分析: [通过/有风险/不通过] — [具体说明]
  3. 长期视角: [通过/有风险/不通过] — [具体说明]

领域审查:
  [底座名]: [通过/有风险/不通过] — [具体说明]

总体评估: [通过 / 有条件通过 / 建议修改]
═══════════════════════
```

### 7.5 主 Agent 逐项回应

审查结果展示后，主 Agent **必须**对每个"有风险"和"不通过"项逐项回应：
- **接受风险**：说明为什么风险可接受（附理由）
- **修改方案**：根据审查建议调整方案
- **不同意**：说明为什么审查结论不成立（附证据）

**不允许**忽略审查发现或用"已知晓"敷衍过去。

---

## 8. Reorder（调整 L1 字段排序）

当用户执行 `/cognitive-kernel reorder` 时执行。

**步骤**：
1. 展示当前各 when 分组的字段列表及顺序
2. 用户指定新顺序
3. 更新 registry 中的排序信息
4. 重新组装（§3）

---

## 9. Monitor（运行监控）

当用户执行 `/cognitive-kernel monitor` 时执行。读取 hook 日志 + 分析运行状态。

**执行步骤**：

### 9.1 读取 Hook 日志

1. 扫描 `~/.cognitive-kernel/logs/` 目录
2. 默认读取最近 7 天的 `.jsonl` 文件（用户可指定 `/cognitive-kernel monitor --days 30`）
3. 解析每行 JSON Lines：`{"ts", "event", "tool", "file"}`
4. 如果存在旧格式 `.json` 文件，读取但标记为"旧格式日志"

### 9.2 统计分析

从日志中提取以下指标：

**L2 触发统计**：
- 总触发次数
- 日均触发次数
- 按工具分布（Edit vs Write）
- 按时段分布（上午 6-12 / 下午 12-18 / 晚上 18-6）
- 触发趋势（最近 7 天 vs 之前 7 天，如果数据充足）

**文件热点**：
- 被修改最多的前 5 个文件路径
- 提示：频繁修改的文件可能需要更强的审查机制

### 9.3 L1 合规率估算

提示用户："是否提供最近 3-5 个包含方案的回复片段用于 L1 合规分析？"
- 如果用户提供 → 对每个片段执行 §6（check）的检查逻辑，汇总合规率
- 如果用户跳过 → 标注"L1 合规率：未检测（需要回复样本）"

### 9.4 输出报告

```
═══ Cognitive Kernel 运行监控 ═══
时间范围: <start_date> ~ <end_date> (<N> 天)

L2 Hook 触发统计:
  总触发: <N> 次 (日均 <N> 次)
  工具分布: Edit <N> (<percent>%) | Write <N> (<percent>%)
  时段分布: 上午 <percent>% | 下午 <percent>% | 晚上 <percent>%
  趋势: <up/down/stable> (对比上一周期)

文件修改热点:
  1. <path> — <N> 次
  2. <path> — <N> 次
  ...

L1 合规率: <percent>% (基于 <N> 个样本) 或 "未检测"

日志存储: <N> 个文件, 占用 <size>
保留策略: 30 天（超过自动提示清理）
═════════════════════════════════
```

---

## 10. Optimize（优化建议）

当用户执行 `/cognitive-kernel optimize` 时执行。基于 monitor 数据生成分析报告和建议。**不自动执行任何调整**。

**执行步骤**：

### 10.1 数据收集

1. 执行 §9（monitor）的数据读取（如果尚未运行）
2. 读取 registry 获取当前底座配置和预算使用
3. 收集用户反馈信号（如果用户提供）：
   - "最近有没有对 Agent 的回复不满意？哪些方面？"
   - 将用户描述的不满映射到可能相关的底座

### 10.2 生成分析报告

**预算使用分析**：
- L5 当前使用 <N>/<recommended>，是否接近上限
- L1 各 when 组字段数 vs 预算，是否过多导致敷衍
- 建议：如果 L1 字段接近上限且合规率下降，考虑精简

**底座激活频率**（基于间接信号）：
- L6 底座在最近对话中被 skill 调用的频率（如果平台有记录）
- 从未被激活的底座 → 建议检查激活条件是否匹配实际使用场景
- 频繁激活的 L6 底座 → 建议考虑升级为 L5

**张力调解有效性**：
- registry 中的 tension_mediations 条目
- 如果用户反馈某对底座频繁冲突 → 建议修改调解规则

**Hook 效果推断**：
- L2 触发频率高但 L1 合规率低 → Hook 提醒可能被忽视，建议强化提示文本
- L2 触发频率低 → 可能修改代码频率低，正常现象

### 10.3 输出建议报告

```
═══ Cognitive Kernel 优化建议 ═══

预算状态:
  L5: <N>/<recommended> [健康/接近上限/已满]
  L1 (proposing): <N>/<budget> 字段 [建议: 精简/保持/有余量]

信号分析:
  [信号] L2 日均 <N> 次触发，L1 合规率 <percent>%
  [解读A] Hook 工作正常，Agent 在响应提醒
  [解读B] 如果合规率 < 80%，考虑 hook 提示文本是否需要更具体

建议操作 (由用户决定是否执行):
  [低风险] 清理 30 天以上的旧日志文件
  [中风险] 将 <base> 从 L6 升级为 L5（频繁激活）
  [中风险] 精简 L1 proposing 组字段（当前 <N> 个，建议 <M> 个）
  [高风险] 修改张力调解规则: <mediation_text>

注意: 以上建议基于有限信号，需要用户判断。
执行任何调整请使用对应命令（如 /cognitive-kernel reorder）。
═════════════════════════════════
```

### 10.4 不做的事

- 不自动修改 registry 或 kernel.md
- 不自动调整预算数值
- 不自动升降级底座
- 所有调整必须由用户明确执行对应命令

---

## 11. All-In（全底座分析）

当用户执行 `/cognitive-kernel all-in`、`/Cognitive-All-In` 或 `/认知全开` 时执行。

**用途**：调动所有已安装认知底座，从每个维度分析同一个问题，最终合成多视角洞察。

**输入**：用户提供的问题、决策、方案或困境。如未提供，提示用户输入。

---

### 11.1 收集底座清单

1. 读取 `~/.cognitive-kernel/cognitive-registry.yml`
2. 收集所有已安装底座（Tier 5 + Tier 6），记录 name 和 path
3. 对每个底座，读取其 manifest.yml，提取 `triggers`（`if` → `then`）和 `core_rules`
4. 向用户报告："已收集 <N> 个认知底座，开始全维度分析。"

### 11.2 分组

将底座按认知功能分为 4 组，每组 spawn 一个子 Agent 并行分析：

| 组 | 功能 | 底座 |
|----|------|------|
| 解构组 | 拆解问题、审计框架、识别焦点 | first-principles, frame-auditing, attention-allocation, inversion-thinking |
| 系统组 | 看见结构、循环、矛盾、跨域模式 | systems-thinking, dialectical-thinking, cross-domain-connector, second-order-thinking |
| 行动组 | 导向结果、校准概率、转化约束 | results-driven, principled-action, constraint-as-catalyst, bayesian-reasoning |
| 元认知组 | 审查动机、检测盲点、战略时机、博弈建模 | double-loop-learning, non-attachment, motivation-audit, temporal-wisdom, interactive-cognition, tacit-knowledge, conviction-override |

**动态调整**：
- 未安装的底座跳过
- 新增底座若不在预设分组中，归入元认知组
- 如果总底座数 ≤ 8，合并为 2 组而非 4 组

### 11.3 Spawn 并行分析 Agents

对每组 spawn 一个子 Agent（使用 Agent 工具，4 组并行），prompt 模板：

```
你是认知分析师，负责从以下视角深度分析一个问题。

## 待分析问题

{用户提供的问题}

## 你的分析视角

{对该组每个底座，列出：}
### {底座名称}
核心规则：
{core_rules 逐条列出}
行为触发器：
{triggers 逐条列出，格式：IF ... THEN ...}

## 要求

对每个视角，输出：
1. **[底座名] 看到了什么**：这个视角揭示了问题的哪个侧面？（2-3 句）
2. **关键洞察**：该视角独有的、其他视角不容易看到的发现（1-2 句）
3. **行动指引**：基于这个视角，应该做什么或避免什么？（1 句）

最后，给出本组视角的 **组内合成**：这些视角合在一起，揭示了什么共同模式或互补洞察？
```

### 11.4 合成

4 个子 Agent 返回后，主 Agent 执行合成：

**Step 1: 汇总**
将 4 组分析结果按组排列展示。

**Step 2: 交叉分析**
- **收敛点**：多个底座独立得出相同结论 → 标记为高置信度信号
- **张力点**：不同底座得出矛盾结论 → 辩证处理（不是简单取舍，而是找出各自成立的条件）
- **盲区检测**：有没有哪个底座完全找不到切入角度？→ 可能暗示问题被错误框定

**Step 3: 主要矛盾定位**（引用 dialectical-thinking）
在所有张力中，哪一个是主要矛盾——解决它会改变其他矛盾的格局？

**Step 4: 最终输出**

```
═══ Cognitive All-In 分析 ═══

问题: {问题}

底座数: {N} 个（Tier-5: {n1}, Tier-6: {n2}）

─── 解构组 ───
{每个底座的 [看到了什么] + [关键洞察] + [行动指引]}
组内合成: {summary}

─── 系统组 ───
{同上}

─── 行动组 ───
{同上}

─── 元认知组 ───
{同上}

─── 交叉分析 ───

收敛点（高置信度）:
- {多个底座独立确认的结论，标注来源底座}

张力点（需辩证处理）:
- {底座 A 认为 X，底座 B 认为 Y}
  → 调解：{在什么条件下 X 优先，什么条件下 Y 优先}

盲区:
- {无底座覆盖的维度，或所有底座都回避的角度}

─── 主要矛盾 ───
{这个问题的核心矛盾是什么？解决它如何改变全局？}

─── 建议行动 ───
{基于全部视角的综合建议，按优先级排列，每条标注支撑的底座}

═════════════════════════════
```

### 11.5 注意事项

- **不修改任何文件**：All-In 是纯分析命令，不改 registry、不改 kernel.md
- **L1 输出协议豁免**：All-In 的输出使用自己的格式模板，不需要遵循常规的 proposing 模板（因为它本身就是一个远比常规模板更全面的分析）
- **耗时预期**：4 个并行子 Agent + 合成，预计 1-2 分钟。提前告知用户
- **可选深度**：用户可追加 `--deep`，此时子 Agent 除读 manifest 外，还 invoke 每个底座的 skill 获取完整框架后再分析（更慢但更深）

---

## 12. 平台适配器（Platform Adapters）

内核不绑定特定平台。manifest 中的抽象事件通过适配器映射到具体平台实现。

### 12.1 抽象事件定义

| 抽象事件 | 语义 | 影响层级 |
|----------|------|---------|
| before_code_change | 即将修改代码文件 | L2 |
| before_file_create | 即将创建新文件 | L2 |
| before_command_run | 即将执行 shell 命令 | L2 |
| before_task_complete | 即将声称任务完成 | L2（多数平台降级为 L3） |
| after_plan_proposed | 提出了方案后 | L2（自定义逻辑） |

### 12.2 Claude Code 适配器（已实现）

| 抽象事件 | 平台映射 | 实现方式 |
|----------|---------|---------|
| before_code_change | PreToolUse: Edit\|Write | ft-settings.json hook → shell script |
| before_file_create | PreToolUse: Write | 合并到 before_code_change hook |
| before_command_run | PreToolUse: Bash | 可选，默认不启用（太频繁） |
| before_task_complete | — | 降级为 L3 基线触发器（无原生事件） |
| after_plan_proposed | — | 降级为 L3（自定义逻辑不可靠） |

配置文件：`~/.claude/ft-settings.json`
Hook 脚本：`~/.cognitive-kernel/hooks/before-code-change.sh`

### 12.3 Cursor 适配器

Cursor 不支持 PreToolUse hook，L2 全部降级为 L3。

| 抽象事件 | 平台映射 | 实现方式 |
|----------|---------|---------|
| before_code_change | — | 降级 L3：注入 .cursorrules |
| before_file_create | — | 降级 L3：注入 .cursorrules |
| before_command_run | — | 降级 L3：注入 .cursorrules |
| before_task_complete | — | 降级 L3：注入 .cursorrules |
| after_plan_proposed | — | 降级 L3：注入 .cursorrules |

**安装方式**：
1. 生成 `.cursorrules` 文件或 `.cursor/rules/cognitive-kernel.mdc`
2. 内容 = cognitive-kernel.md 全文（L1+L3+L4+L5+L6）
3. L2 hook 提示降级为 L3 trigger：原 hook inject 文本追加到 L3 triggers 列表
4. regenerate 时自动检测 Cursor 适配器 → 生成 .cursorrules

### 12.4 Gemini CLI 适配器

Gemini CLI 支持 GEMINI.md（类似 CLAUDE.md），但不支持 hook 机制。

| 抽象事件 | 平台映射 | 实现方式 |
|----------|---------|---------|
| before_code_change | — | 降级 L3：注入 GEMINI.md |
| before_file_create | — | 降级 L3：注入 GEMINI.md |
| before_command_run | — | 降级 L3：注入 GEMINI.md |
| before_task_complete | — | 降级 L3：注入 GEMINI.md |
| after_plan_proposed | — | 降级 L3：注入 GEMINI.md |

**安装方式**：
1. 在 `~/.gemini/GEMINI.md` 中添加 `@~/.cognitive-kernel/cognitive-kernel.md` 引用
2. L2 内容全部并入 L3 triggers（降级）
3. cognitive-kernel.md 本身不需要改动（L3 部分已包含降级的 L2 内容）

### 12.5 降级规则（L2 → L3）

当平台不支持 hook 时，L2 内容自动降级为 L3 trigger：

```
原 L2 hook:
  event: before_code_change
  inject: "这个修改对全局有什么影响？"

降级后 L3 trigger:
  if: "即将修改代码文件（Edit/Write 操作）"
  then: "暂停——这个修改对全局有什么影响？想清楚再动。"
```

**regenerate 时的适配器检测**：
1. 读取 registry 中的 `platform` 字段（setup 时记录）
2. 如果 platform 不支持 hook → 自动将 L2 hook 内容合并到 L3 section
3. 生成的 cognitive-kernel.md 在 L3 部分包含降级标注：`[L2→L3 降级] <trigger>`

### 12.6 多平台同时使用

用户可能在多个平台上使用同一套认知底座。registry 支持多平台注册：

```yaml
platforms:
  - name: claude-code
    adapter: claude-code
    config_path: ~/.claude/ft-settings.json
    l2_support: true
  - name: cursor
    adapter: cursor
    config_path: .cursor/rules/cognitive-kernel.mdc
    l2_support: false
  - name: gemini
    adapter: gemini
    config_path: ~/.gemini/GEMINI.md
    l2_support: false
```

setup 时询问："你使用哪些 AI 编程工具？" → 注册对应适配器 → regenerate 时按平台生成配置。

---

## A. 基线内容（Baseline）

以下内容在 setup 时写入初始 cognitive-kernel.md，构成不依赖任何底座的最低保障。

### A.1 核心原则

- 实事求是：一切表述必须与实际情况相符，不夸大、不模糊、不隐瞒
- 没有调研就没有发言权：回答问题基于实际调查，不进行猜测。如果没有验证过，明确说明"我没有验证"

### A.2 L1 基线：场景分类门控 + 输出字段

**Step 0: 场景分类门控**（每次回复前强制执行）：
- 在写任何回复前，先判断场景：proposing / claiming_done / 都不是
- 防绕过规则：用户说"先帮我想方案"/"讨论一下" = 仍是 proposing；多方案比较后推荐 = proposing；代码实现中选了实现方式 = proposing
- 豁免条件（满足任一即可）：(1) 回复完全不涉及选择、判断或建议；(2) Pipeline 协议优先——当 pipeline 执行协议（如 apex-forge）已通过 Skill tool 加载但未初始化时，必须先完成 pipeline 初始化再应用 proposing 模板，pipeline 阶段内的输出仍需满足 L1 格式

**proposing 模板字段**:
1. "## 方案"（结构性 heading）
2. "## 为什么选这个而不是更难但可能更对的路？"
3. "## 这会破坏什么？"
4. "## 一年后回看"
5. "## 假设审计"

**claiming_done 模板字段**:
1. "## 完成证据"

### A.3 L3 基线触发器

1. IF 即将声称"完成"/"搞定"/"done"/"fixed" → THEN 停下来，重读原始需求，逐项核对每一点是否满足。未满足的明确列出。
2. IF 有两个方案且其中一个明显更简单 → THEN 在选择前先写出：为什么简单方案不是在回避真正困难？如果确实简单方案更好，说明原因。
3. IF 发现自己在写"应该可以"/"should work"/"这样就行了" → THEN 将推测替换为具体的验证步骤，并实际执行验证。
4. IF 即将陈述一个事实性结论 → THEN 确认：这是基于实际调查（读了文件/跑了命令/查了文档）还是推测？如果是推测，明确标注"我没有验证这一点"。
5. IF 发现自己使用模糊措辞（"应该是"/"大概"/"一般来说"/"通常"） → THEN 要么去调查获取确切信息，要么将措辞改为明确的不确定表达："我不确定，需要验证"。

### A.4 L4 基线审查协议

**自动触发条件**（满足任一即 spawn 子 Agent）：
- 修改涉及 5 个以上文件
- 变更公共接口、API 签名或数据结构
- 架构级改动（新增子系统、改变数据流向、引入新依赖）

**建议触发条件**（提示用户确认）：
- 修改涉及 3-4 个文件
- 修改核心业务逻辑

**子 Agent 审查 brief（基线）**：
```
你是一个对抗性审查员。你的唯一职责是找出方案中的问题。
不要确认方案是好的——你的工作是找问题。

审查以下方案：
{方案内容}

基线审查项：
1. 这是选了"正确的路"还是"容易的路"？有没有更难但更对的替代方案被忽略了？
2. 这会破坏什么？列出所有受影响的模块、文件、功能。特别关注不在方案视野内的部分。
3. 从一年后回看，这是最佳选择吗？是否在积累技术债或创造未来的维护负担？

对每项输出：通过 / 有风险（说明具体风险）/ 不通过（说明问题和建议方向）
```

---

## B. Claude Code Hook 配置

**重要**：Claude Code 用户级配置文件是 `~/.claude/ft-settings.json`（不是 `settings.json`）。

setup 时在 `~/.claude/ft-settings.json` 的 hooks 段添加以下配置：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "bash ~/.cognitive-kernel/hooks/before-code-change.sh",
            "timeout": 5
          }
        ]
      }
    ]
  }
}
```

**设计决策：before_task_complete hook 降级为 L3**
Claude Code 没有原生的"任务完成"事件。before_task_complete 抽象事件无法直接映射到 PreToolUse/PostToolUse。
因此该检查降级为 L3 基线触发器（§A.3 第1条："IF 即将声称完成 → THEN 逐项核对"）。
未来如平台支持 task-complete 事件，可在此处添加对应 hook。

**注意**：
- 添加时检查已有的 hooks 配置，做合并而非覆盖
- 如果已存在相同 matcher 的 hook，追加到 hooks 数组中
- setup 前备份 ft-settings.json

---

## C. Manifest Schema Reference（v1）

供 install 命令解析时参考：

```yaml
# 必填
schema_version: 1          # 固定值
name: string               # 底座名称（kebab-case）
tier: 5 | 6               # 期望层级

# 可选
output_fields:
  - prompt: string         # 提示文本
    when: proposing_solution | proposing_change | claiming_done | always

hooks:
  - event: before_code_change | before_file_create | before_command_run | before_task_complete | after_plan_proposed
    inject: string         # 注入文本

triggers:
  - if: string             # 触发条件
    then: string           # 执行动作
    action_type: require_justification | require_evidence | require_verification | pause_and_check | search_before_answer  # 可选

core_rules:                # 仅 tier: 5
  - rule: string
    rank: 1 | 2 | 3       # 1=最核心

activation:                # 激活场景
  - string

recommends:                # 软依赖
  - string
```
