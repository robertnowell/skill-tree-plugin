---
name: skill-tree
description: Analyze your Claude collaboration style and generate a skill tree visualization with character archetype card. Use when the user says "skill tree", "show my skills", "analyze my style", "check my ai fluency", or wants to see their AI fluency profile.
---

# Skill Tree

Generate a personalized AI fluency profile by analyzing the user's conversation history **on the current surface**.

**CRITICAL: You MUST use the `analyze` MCP tool for classification. NEVER attempt to classify behaviors yourself — your classifications will be inconsistent and unvalidated. The remote server uses a calibrated classifier with cached results for consistency across runs.**

## Step 0: Detect surface and set up

### If running in Cowork (cloud sandbox):

**0a. Enable network egress** — The analyzer runs on a remote server. If you get a network error when calling the `analyze` tool, tell the user:

> "The Skill Tree analyzer needs network access. Please enable it:
> **Settings → Code execution and file creation → Allow network egress → toggle ON**
> Then say 'skill tree' again."

**Do NOT proceed without network access. Do NOT classify manually as a fallback.**

**0b. Mount Cowork sessions only:**

```
request_cowork_directory("~/Library/Application Support/Claude/local-agent-mode-sessions")
```

Only mount the Cowork directory — do NOT mount `~/.claude/projects`. Each surface analyzes its own sessions.

### If running in Claude Code:

No setup needed. You'll read from `~/.claude/projects/` only. Do NOT read from `~/Library/Application Support/Claude/`.

## Step 1: Find session files

**Cowork only:**
```bash
find ~/Library/Application\ Support/Claude/local-agent-mode-sessions -name "*.jsonl" -size +1k 2>/dev/null | head -30
```

**Claude Code only** (use Glob if available, otherwise find):
```bash
find ~/.claude/projects -name "*.jsonl" -size +1k ! -path "*/subagents/*" 2>/dev/null | head -30
```

Do NOT mix sessions from both sources.

## Step 2: Extract user messages

For each JSONL file, read it and extract user messages. Each line is JSON. Keep lines where:
- `type` is `"user"`
- `message.content` is a string (not an array/list)
- Content does NOT contain paste markers: `⏺`, `⎿`, `ctrl+o to expand`, `✻ Brewed`, `✻ Baked`
- Content is longer than 10 characters

Truncate each message to 2000 characters. Group by file (filename without `.jsonl` = session ID). Aim for 15-30 sessions.

## Step 3: Call the analyze tool

Format extracted sessions as a JSON string and call the `analyze` MCP tool:

```
analyze({ sessions_json: '[{"id":"uuid1","messages":["msg1","msg2"]},{"id":"uuid2","messages":["msg3"]}]' })
```

The parameter is a **JSON string** containing an array of `{id, messages}` objects.

**If the tool call fails with a network error:** Do NOT fall back to manual analysis. Guide the user to enable network egress (see Step 0a).

## Step 4: Save profile locally

After receiving the profile JSON from the `analyze` tool, write two files:

**4a. Save the growth quest** (enables the SessionStart hook to nudge you in future sessions):
```bash
mkdir -p ~/.skill-tree
```
Then write the `growth_quest` field from `profile.archetype` to `~/.skill-tree/growth-quest.txt`.

**4b. Save the profile** (for the visualization):
Write the full profile JSON to `~/.skill-tree/profile.json`.

## Step 5: Generate visualization

Call the `visualize` MCP tool with the profile JSON string:

```
visualize({ profile_json: '<the profile JSON string from step 3>' })
```

This returns self-contained HTML. Save it to `~/.skill-tree/report.html` and open it in the browser:
```bash
open ~/.skill-tree/report.html
```

## Step 6: Present results conversationally

Present the key findings:

1. **Surface context** — "Based on your [Claude Code / Cowork] sessions:"
2. **Archetype** — name and tagline
3. **Superpower** — their distinctive strength
4. **Axis scores** — Specification %, Evaluation %, Setup % (vs population averages of 28%, 15%, 30%)
5. **Growth quest** — one specific action for their next session
6. **Growth edge** — the behavior with the largest gap vs population average

## Archetype Reference

| Archetype | Pattern |
|-----------|---------|
| The Polymath | Shapes AND evaluates (rarest) |
| The Conductor | Plans AND shapes |
| The Architect | Plans AND evaluates |
| The Forgemaster | Shapes output precisely |
| The Illuminator | Questions and probes |
| The Compass | Sets clear direction |
| The Catalyst | Pure momentum |

## The 11 Behaviors (from AI Fluency Framework)

| Branch | Behavior | Population Avg |
|--------|----------|---------------|
| Planning | Clarifies goals upfront | 51% |
| Planning | Discusses approach first | 10% |
| Craft | Iterates on outputs | 86% |
| Craft | Provides examples | 41% |
| Craft | Specifies format | 30% |
| Craft | Sets interaction style | 30% |
| Craft | Expresses tone preferences | 23% |
| Craft | Defines audience | 18% |
| Judgment | Flags context gaps | 20% |
| Judgment | Questions Claude's logic | 16% |
| Rigor | Verifies facts | 9% |

Baselines from [Anthropic's AI Fluency Index](https://www.anthropic.com/research/AI-fluency-index) (Feb 2026, N=9,830 conversations).
