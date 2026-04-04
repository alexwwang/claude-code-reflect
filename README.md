# claude-code-reflect

> *"Knowing yourself is the beginning of all wisdom." — Aristotle*

[中文文档](README_zh.md)

A Claude Code plugin that detects when you correct the model, performs root cause analysis in the background, and drafts memory artifacts for your approval.

## What It Does

When you correct Claude's mistake (e.g., "That's wrong, the API uses callback not promise"), reflect:

1. **Detects** the correction signal (supports English and Chinese)
2. **Launches** a background Claude session to analyze WHY the error happened
3. **Drafts** memory artifacts (feedback memories, project notes) for your review
4. **Waits** for your approval before writing anything

The analysis uses a structured root cause taxonomy: Assumption Error, Context Gap, Wrong Abstraction, Scope Mismatch, Stale Knowledge, Over-Engineering, Under-Specification.

## Installation

This plugin has two branches with different dependency requirements:

| Branch | OMC Required | Install Command |
|--------|-------------|-----------------|
| `main` | Yes | `claude plugin add github:alexwwang/claude-code-reflect` |
| `standalone` | No | `claude plugin add github:alexwwang/claude-code-reflect --branch standalone` |

### Option A: With oh-my-claudecode (`main` branch, recommended)

Uses OMC's MCP tools (`notepad_write_priority`, `project_memory_add_note`) for cross-compaction notifications and project memory integration.

```bash
# 1. Install oh-my-claudecode first
claude plugin add github:Yeachan-Heo/oh-my-claudecode

# 2. Install reflect (main branch, default)
claude plugin add github:alexwwang/claude-code-reflect
```

### Option B: Standalone (`standalone` branch)

Replaces all OMC dependencies with direct file operations. No OMC needed, but loses cross-compaction notification reliability and cross-skill delegation.

```bash
claude plugin add github:alexwwang/claude-code-reflect --branch standalone
```

See [Branch Comparison](#branch-comparison) for detailed differences.

## Usage

### Trigger a Reflection

When you correct Claude, invoke the skill:

```
/oh-my-claudecode:reflect
```

Or just say something corrective and then invoke it. The skill scans recent turns for correction signals.

### Review Results

When the background analysis completes, review the drafts:

```
/oh-my-claudecode:reflect review ref-20260404a
```

You can approve all, approve with modifications, re-analyze with more context, or discard.

### Inspect Progress

Check on a running background analysis:

```
/oh-my-claudecode:reflect inspect ref-20260404a
```

This shows session status, logs, and how to resume the subagent session directly.

## Quick Start: Verify Installation

After installing and restarting Claude Code, run this smoke test to confirm everything works:

**Test 1: Skill is discovered**
```
# In Claude Code, type:
/oh-my-claudecode:reflect

# Expected: Claude asks what you want to reflect on (AskUserQuestion appears)
# If you see "skill not found", restart Claude Code completely (not just /reload-plugins)
```

**Test 2: End-to-end flow (Chinese correction)**
```
# In Claude Code, have this conversation:

You: Tell me about Go's append() function.
Claude: [gives some explanation, e.g. "append() always creates a new slice"]
You: That's wrong. append() reuses the underlying array when capacity is sufficient.
You: /oh-my-claudecode:reflect

# Expected flow:
# 1. Claude detects "That's wrong" as a direct negation signal
# 2. Claude launches a background session and gives you the session UUID
# 3. Main conversation continues normally
# 4. When background completes, you get a notification
# 5. Run: /oh-my-claudecode:reflect review ref-xxxxxxxx
# 6. You see the RCA report with drafted memory artifacts
```

**Test 3: End-to-end flow (English correction)**
```
You: What does Python's sorted() return?
Claude: [says something wrong, e.g. "sorted() sorts the list in place"]
You: Incorrect, sorted() returns a new list, it doesn't modify the original.
You: /oh-my-claudecode:reflect

# Expected: same flow as Test 2
```

**Test 4: Review flow**
```
# After the background task completes:
/oh-my-claudecode:reflect review ref-xxxxxxxx

# Expected: Claude displays:
#   - Root Cause Analysis (category, reasoning chain)
#   - Severity rating
#   - Drafted memory artifact(s) with target file path
#   - Options: Approve all / Approve with modifications / Re-analyze / Discard
```

**What to check in the output:**
- `.omc/reflections/ref-xxxxxxxx/report.md` exists and contains the full RCA
- `.omc/reflections/ref-xxxxxxxx/state.json` shows `status: "pending_review"`
- After approving, feedback memory file exists in `~/.claude/projects/*/memory/`

## What You'll See

### The RCA Report

When you run `/reflect review`, you'll see output like this:

```
## Root Cause Analysis
- Category: Wrong Abstraction
- Root cause: The model abstracted Go's append() as a reallocating operation,
  conflating the guarantee of reallocation with the common case.
- Reasoning chain:
  1. Correctly understood append() can trigger reallocation when capacity exceeded
  2. Incorrectly elevated this to a guarantee about old-slice isolation
  3. Failed to account for the dual behavior when capacity is sufficient

## Prevention
When reasoning about conditional behavior, always identify the complementary
case and state both branches explicitly.

## Severity: Important

## Drafted Artifacts
1. [feedback] go-append-slice-isolation
   Target: ~/.claude/projects/.../memory/feedback_go-append-isolation.md
   Rule: In Go, append() may reuse the existing underlying array.
   Only copy() or explicit allocation guarantees isolation.
```

### The Feedback Memory (after approval)

Approved artifacts are written as Claude Code feedback memories. Future conversations
will automatically load this memory:

```markdown
---
name: go-append-slice-isolation
description: Go append() does not guarantee slice isolation
type: feedback
---

In Go, append() may reuse the existing underlying array if capacity is sufficient.

**Why:** append() only allocates a new backing array when capacity is exceeded.
If the existing array has room, the append writes into shared memory.

**How to apply:** Whenever code relies on an old slice remaining unchanged after
an append() to a derived slice, enforce isolation with copy() or three-index slice.
```

This memory is loaded into context automatically in future sessions, preventing
the same error from recurring.

### File Layout After Usage

```
.omc/reflections/
  ref-20260404a/
    report.md       <- Full RCA report + drafted artifacts
    state.json      <- Status: pending_review -> approved
  ref-20260404b/
    report.md
    state.json

~/.claude/projects/{project}/memory/
  MEMORY.md                     <- Index (updated automatically)
  feedback_go-append-isolation.md  <- Approved feedback memory
```

## How It Works

```
Turn 1: User corrects Claude -> /reflect -> detection + background launch
Turn 1: Main conversation continues normally
   ...background subagent runs RCA, writes report.md + state.json...
Turn 2+: User runs /reflect review ref-xxxx -> sees RCA + drafts -> approves -> memory written
```

The background subagent runs as a persistent Claude session (`claude --session-id`). You can inspect progress at any time by opening a new terminal and running `claude --resume <session_uuid>`.

### Permission & Safety Model

The background subagent runs with `--permission-mode bypassPermissions` instead of `auto`. This is necessary because:

- `auto` mode depends on an internal safety classifier model that can be temporarily unavailable, which would block ALL tool calls and leave the subagent stuck
- Background subagents cannot prompt the user interactively for permissions
- The subagent runs a bounded, user-initiated task with a tightly-scoped prompt

To compensate for bypassing platform-level safety checks, the subagent prompt includes **mandatory path restrictions**:

- Files may ONLY be written to `.reflect/reflections/{id}/` and user memory directories
- Destructive commands (`rm -rf`, `git push --force`, etc.) are forbidden
- Modifying `.git/`, `CLAUDE.md`, or config files is forbidden

### Error Handling

If the background subagent fails, the skill classifies the error and offers appropriate recovery options:

| Error Type | Detection Pattern | Recovery |
|------------|-------------------|----------|
| API connectivity | `ECONNRESET`, `timeout` | Retry with new session |
| Rate limit | `rate limit`, `429` | Wait and retry |
| Platform issue | Any other failure | Inline fallback or discard |

### The Three Modes

| Command | When to Use | What Happens |
|---------|------------|--------------|
| `/reflect` | After you correct Claude | Scans for correction signals, launches background RCA |
| `/reflect review ref-xxxx` | After background completes | Shows RCA + drafted artifacts, asks for approval |
| `/reflect inspect ref-xxxx` | While background is running | Shows session status, logs, how to debug |

## Branch Comparison

| Feature | `main` (OMC) | `standalone` |
|---------|-------------|-------------|
| OMC required | Yes | No |
| Cross-compaction notifications | `notepad_write_priority` MCP | File-based (`.reflect/notifications.md`) |
| Project memory | `project_memory_add_note` MCP | File-based (`.reflect/project-memory.json`) |
| Cross-skill delegation | `/learner`, `/remember` | Removed |
| Data directory | `.omc/reflections/` | `.reflect/reflections/` |
| Claude Code native memory | Yes | Yes |

> **Note:** Examples in this README use `main` branch paths (`.omc/reflections/`). On the `standalone` branch, all data is stored under `.reflect/` instead (e.g., `.reflect/reflections/`, `.reflect/notifications.md`, `.reflect/project-memory.json`).

## Iterative Development

This skill is designed to be iteratively improved through usage. The recommended workflow:

1. **Use the skill** in real conversations to catch model errors
2. **Review reflections** to identify patterns in the root cause taxonomy
3. **Tune the SKILL.md** based on real-world corrections that weren't caught or were misclassified
4. **Update the correction signal taxonomy** when you find new patterns (e.g., domain-specific corrections)
5. **Adjust the subagent prompt** if RCA quality is unsatisfactory

### What to iterate on

- **Correction Signal Taxonomy** (Step 2.1): Add new signal patterns as you encounter them
- **Root Cause Categories**: The 7-category taxonomy can be refined for your domain
- **Subagent prompt** (inside `<SUBAGENT_PROMPT>`): Adjust RCA depth, add domain-specific guidance
- **Draft artifact templates**: Customize memory artifact format for your workflow
- **Severity thresholds**: Tune what counts as Critical vs Important vs Minor

### Testing changes

```bash
# After editing SKILL.md, restart Claude Code to reload the plugin
# Then test with a known correction pattern

# Check test plan for covered scenarios
cat skills/reflect/tests/TEST_PLAN.md
```

### Contributing changes back

If you improve the correction detection or RCA quality, consider opening a PR. The most valuable contributions are:
- New correction signal patterns (especially non-English)
- Adjusted root cause taxonomy with real-world examples
- Subagent prompt improvements that produce better RCA

## Development

```bash
git clone https://github.com/alexwwang/claude-code-reflect.git
cd claude-code-reflect

# Main branch (with OMC)
git checkout main

# Standalone branch (no OMC)
git checkout standalone
```

## License

MIT
