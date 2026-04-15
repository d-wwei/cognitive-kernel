[English](README.md) | [中文](README.zh.md)

# Cognitive Kernel

**Rules don't enforce themselves.**

One cognitive base, 30 lines of rules, injected into `CLAUDE.md` — it works. The agent's thinking changes. You can see it in the output.

Now install 19 bases. That's 720+ lines of rules, 100+ individual instructions, all competing for the agent's attention in the same context window. What happens? The same thing that happens when you tape 100 checklists to a wall — nobody reads any of them. The agent's attention gets diluted. Critical rules get skipped. The cognitive bases you installed are technically "active" but functionally invisible.

## Why Text Rules Fail

Writing "you must do X" in a config file is the weakest possible intervention. It depends entirely on the agent noticing the rule, at the right moment, under cognitive load. That's like depending on a factory worker reading a safety poster while operating heavy machinery.

The 6-layer model replaces hope with structure. Instead of one mechanism (text in a file), each rule is enforced through the strongest viable channel:

| Layer | Mechanism | Strength | How It Works |
|-------|-----------|----------|-------------|
| **L1** | Output templates | Strongest | Agent's response must include specific fields — structurally incomplete without them |
| **L2** | Hooks | Strong | Platform hooks auto-inject reminders before key actions — agent gets intercepted |
| **L3** | If-Then triggers | Medium | Pattern-matched rules fire automatically — no willpower needed |
| **L4** | External verification | Strong | A separate sub-agent reviews the work adversarially — self-checking doesn't work |
| **L5** | Core rules | Weak | A few critical rules stay in context — always visible, but still voluntary |
| **L6** | Skill reference | Weakest | Full framework loaded on demand — agent has to choose to read it |

**The key insight:** L1 and L2 are structural — the agent cannot produce valid output without complying. L5 and L6 are voluntary — the agent can ignore them under pressure. The kernel puts each rule at the strongest layer it can occupy.

## Three Jobs

Cognitive Kernel does exactly three things:

### 1. Conflict Analysis

When you install a new base, the kernel compares its manifest against every already-installed base:

- **L1 overlap** — Do two bases require similar output fields? (e.g., both want an "assumptions" section → merge or prioritize)
- **L3 tension** — Do triggers point in opposite directions? (e.g., "decompose to parts" vs "see the whole first" → define which fires when)
- **L5 conflict** — Do core rules contradict? (e.g., "act fast" vs "verify first" → dialectical resolution)

The kernel reports overlaps, tensions, and conflicts, then proposes resolutions. You decide.

### 2. Budget Management

The context window is finite. More rules ≠ better thinking. The kernel enforces budgets:

- **L5 core rules**: max 4 bases active simultaneously (~12 rules total). More than that → attention dilution.
- **L1 output fields**: per-context budgets (5 fields when proposing a solution, 3 when claiming completion). Prevents output template bloat.

When you try to install a 5th L5 base, the kernel warns you and asks which one to demote to L6 (on-demand only).

### 3. 6-Layer Assembly

Each cognitive base describes itself through `manifest.yml` — what it needs at each intervention layer. The kernel reads every installed base's manifest and assembles the runtime:

- L1 fields → written into `~/.cognitive-kernel/cognitive-kernel.md` as output template headings
- L2 hooks → registered with the platform's hook system (e.g., Claude Code's `ft-settings.json`)
- L3 triggers → assembled as if-then rules in the runtime config
- L4 prompts → stored as adversarial review briefs for sub-agents
- L5 rules → selected by budget, injected into always-on context
- L6 refs → registered as available skills for on-demand loading

Run `/cognitive-kernel regenerate` to rebuild the runtime from scratch based on current registry state.

## Quick Start

### Via meta-cogbase (Recommended)

If you installed [meta-cogbase](https://github.com/d-wwei/meta-cogbase), you already have the kernel. Nothing else to do.

### Manual Installation

1. Copy the skill files to your agent's skill directory:
   ```bash
   mkdir -p ~/.claude/skills/cognitive-kernel
   cp SKILL.md README.md ~/.claude/skills/cognitive-kernel/
   ```

2. Initialize the kernel:
   ```
   /cognitive-kernel setup
   ```

3. Install cognitive bases:
   ```
   /cognitive-kernel install /path/to/first-principles
   /cognitive-kernel install /path/to/results-driven
   ```

4. Verify:
   ```
   /cognitive-kernel status    # See what's installed and budget usage
   /cognitive-kernel check     # Run L1 compliance check
   ```

## Commands

| Command | What It Does |
|---------|-------------|
| `setup` | First-time initialization — creates `~/.cognitive-kernel/`, baseline config, hooks |
| `install <path>` | Install a base — reads manifest, runs conflict analysis, assembles into runtime |
| `uninstall <name>` | Remove a base — cleans up all layers, regenerates runtime |
| `status` | Installed bases, L5 budget usage, L1 field counts, tension mediations |
| `check` | L1 compliance linter — verifies agent output includes required fields |
| `review` | L4 adversarial review — spawns a sub-agent to challenge the current work |
| `monitor` | View hook trigger logs — which rules fired, how often, when |
| `optimize` | Data-driven suggestions — based on monitor data, recommends budget adjustments |
| `regenerate` | Rebuild runtime from registry — reprocesses all manifests, reassembles all layers |
| `reorder` | Adjust L1 field ordering — change which fields appear first in output templates |

## For Base Authors

Every cognitive base that works with the kernel needs a `manifest.yml`:

```yaml
schema_version: 1
name: my-framework
tier: 5                    # 5 = core (can contribute L5 rules), 6 = specialized (L6 only)

output_fields:             # L1: Required output fields
  - prompt: "Assumptions audited: [list givens vs conventions]"
    when: proposing_solution

hooks:                     # L2: Auto-injected reminders
  - event: before_task_complete
    inject: "Did I rebuild from fundamentals, or just reassemble the existing answer?"

triggers:                  # L3: Pattern-matched rules
  - if: "A problem statement contains an implicit 'should'"
    then: "Name the presupposition explicitly before proceeding"
    action_type: pause_and_check

core_rules:                # L5: Always-on rules (tier 5 only)
  - rule: "Separate givens from conventions before solving any problem"
    rank: 1

activation:                # When this base is most relevant
  - "Analyzing assumptions in any domain"
```

Use [Cognitive Base Creator](https://github.com/d-wwei/cognitive-base-creator) to generate complete bases with manifests automatically.

## Runtime Structure

```
~/.cognitive-kernel/
  ├── cognitive-kernel.md       # Assembled runtime (L1/L3/L4/L5/L6 content)
  ├── cognitive-registry.yml    # Installed bases + conflict resolutions
  ├── hooks/
  │   └── before-code-change.sh # L2 hook script
  └── logs/
      └── YYYY-MM-DD.jsonl      # Hook trigger event log
```

## Supported Platforms

| Platform | L2 Hook Support | How |
|----------|----------------|-----|
| Claude Code | Full | `ft-settings.json` PreToolUse hook |
| Gemini CLI | Degraded (L2→L3) | `GEMINI.md` @ reference |
| Cursor | Degraded (L2→L3) | `.cursorrules` / `.mdc` files |
| Codex CLI | Degraded (L2→L3) | `AGENTS.md` inline injection |
| OpenCode | Degraded (L2→L3) | `AGENTS.md` inline injection |
| OpenClaw | Degraded (L2→L3) | `AGENTS.md` inline injection |

Platforms without hook support automatically degrade L2 content to L3 triggers — rules still fire, just through pattern matching instead of event interception.

## License

MIT

## Links

- [meta-cogbase](https://github.com/d-wwei/meta-cogbase) — Package manager for cognitive bases (includes kernel)
- [Cognitive Base Creator](https://github.com/d-wwei/cognitive-base-creator) — Generate new bases from any thinking framework
- [Article: Every Agent Is Missing a Layer](https://github.com/d-wwei/cognitive-base-creator/blob/main/docs/article-zh.md) — Why cognitive bases matter
