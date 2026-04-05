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

```bash
# Step 1: Register marketplace
/plugin marketplace add alexwwang/claude-code-reflect

# Step 2: Install plugin
/plugin install claude-code-reflect@alexwwang/claude-code-reflect

# Step 3: Restart Claude Code completely (not just /reload-plugins)
```

> **Note:** Only the `standalone` branch is actively maintained. The `main` (OMC-dependent) branch is deprecated.
> This plugin is fully compatible with oh-my-claudecode — each system manages its own data directory (`.reflect/` vs `.omc/`) and feedback memories in `~/.claude/projects/*/memory/` use Claude Code's native format recognized by both systems.

## Usage

### Trigger a Reflection

When you correct Claude, invoke the skill:

```
/claude-code-reflect:reflect
```

Or just say something corrective and then invoke it. The skill scans recent turns for correction signals.

### Review Results

When the background analysis completes, review the drafts:

```
/claude-code-reflect:reflect review ref-20260404a
```

You can approve all, approve with modifications, change scope, re-analyze with more context, or discard.

### Inspect Progress

Check on a running background analysis:

```
/claude-code-reflect:reflect inspect ref-20260404a
```

This shows session status, logs, and how to resume the subagent session directly.

## Quick Start: Verify Installation

After installing and restarting Claude Code, run this smoke test to confirm everything works:

**Test 1: Skill is discovered**
```
# In Claude Code, type:
/claude-code-reflect:reflect

# Expected: Claude asks what you want to reflect on (AskUserQuestion appears)
# If you see "skill not found", restart Claude Code completely (not just /reload-plugins)
```

**Test 2: End-to-end flow (Chinese correction)**
```
# In Claude Code, have this conversation:

You: Tell me about Go's append() function.
Claude: [gives some explanation, e.g. "append() always creates a new slice"]
You: That's wrong. append() reuses the underlying array when capacity is sufficient.
You: /claude-code-reflect:reflect

# Expected flow:
# 1. Claude detects "That's wrong" as a direct negation signal
# 2. Claude launches a background session (atomic preparation) and gives you the session UUID
# 3. Main conversation continues normally
# 4. When background completes, you get a notification
# 5. Run: /claude-code-reflect:reflect review ref-xxxxxxxx
# 6. You see the RCA report with scope-judged drafted artifacts
```

**Test 3: End-to-end flow (English correction)**
```
You: What does Python's sorted() return?
Claude: [says something wrong, e.g. "sorted() sorts the list in place"]
You: Incorrect, sorted() returns a new list, it doesn't modify the original.
You: /claude-code-reflect:reflect

# Expected: same flow as Test 2
```

**Test 4: Review flow**
```
# After the background task completes:
/claude-code-reflect:reflect review ref-xxxxxxxx

# Expected: Claude displays:
#   - Root Cause Analysis (category, reasoning chain)
#   - Severity rating
#   - Drafted memory artifact(s) with scope (USER-LEVEL or PROJECT-LEVEL), reasoning, and target path
#   - Options: Approve all / Approve with modifications / Change scope / Re-analyze / Discard
```

**What to check in the output:**
- `.reflect/reflections/ref-xxxxxxxx/report.md` exists and contains the full RCA
- `.reflect/reflections/ref-xxxxxxxx/state.json` shows `status: "pending_review"` with `artifacts` array containing `scope` and `scope_reasoning` per artifact
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
1. [artifact-001] go-append-slice-isolation
   Scope:   USER-LEVEL
   Reason:  Go language semantics error, applies globally
   Target:  ~/.claude/projects/.../memory/feedback_go-append-isolation.md
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
.reflect/reflections/
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
Turn 1: User corrects Claude -> /reflect -> detection + atomic preparation + background launch
Turn 1: Main conversation continues normally
   ...background subagent runs RCA, writes ONLY to staging (.reflect/reflections/{id}/)...
Turn 2+: User runs /reflect review ref-xxxx -> sees RCA + drafts with scope -> approves -> memory written
```

The background subagent runs as a persistent Claude session (`claude --session-id`). You can inspect progress at any time by opening a new terminal and running `claude --resume <session_uuid>`.

### Permission & Safety Model

The skill uses a three-phase permission design:

| Phase | Session type | `bypassPermissions` | Writes to |
|-------|-------------|---------------------|-----------|
| Preparation | main session (1 atomic Bash call) | Yes — atomicity | `.reflect/reflections/` only |
| Background analysis | background subagent | Yes — required for file writes in non-interactive mode | `.reflect/reflections/{id}/` only |
| Review + write | interactive (resumed) session | No | `.reflect/` + `~/.claude/` |

**Why `bypassPermissions` in preparation:** The preparation phase (uuidgen, mkdir, write state.json, write prompt, launch subagent) is merged into a single Bash call. Without `bypassPermissions`, this call produces a confirmation prompt in `ask` mode, creating an interruption window where the user's next message could interleave with the preparation flow, causing the subagent to never launch. Since all operations are within the project root (mkdir, file write) and `/tmp`-equivalent, there is no meaningful security risk.

**Why `bypassPermissions` on the subagent:** The subagent runs via `claude -p` in non-interactive mode — without `bypassPermissions`, all tool calls (including Write) are denied because there's no way to interactively approve permissions. This is safe because the subagent prompt restricts writes to `.reflect/reflections/{id}/` within the project root. Note: Claude Code has a confirmed bug where `bypassPermissions` silently denies writes *outside* the project root — this is acceptable since our subagent only writes within the project.

**Scope judgment:** During analysis, the model determines whether each artifact is **user-level** (general knowledge, applies across projects) or **project-level** (codebase-specific context). You see the scope reasoning at review time and can override before any write occurs.

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

## Branch Status

> **`standalone` is the only actively maintained branch.** The `main` (OMC-dependent) branch is deprecated and will not receive further updates.
>
> This plugin coexists with oh-my-claudecode without conflicts. Each system manages its own data directory (`.reflect/` vs `.omc/`). Feedback memories in `~/.claude/projects/*/memory/` use Claude Code's native format, recognized by both systems.

## Known Issues

These issues were identified during real-world testing and are candidates for future improvement.

1. **Subagent model configuration missing**

   The `claude -p` command that launches the subagent does not specify a model parameter. This may cause the subagent to use a default model instead of the main session's current model.

   *Fix direction:* SKILL.md should guide extracting the current model ID from the main session and passing it via `--model` to the subagent.

2. **Session ID collision on retry**

   After a failed subagent launch, a retry may reuse the already-registered session UUID, causing `claude --resume` behavior to be unpredictable.

   *Fix direction:* Every retry (including `--deep` re-analysis) must generate a new UUID and update `state.json`.

3. **Read tool rendering vs. actual file content**

   When verifying files containing markdown syntax (especially backtick code blocks), the Read tool may render content incorrectly due to nested markdown parsing. The actual file on disk is correct — this is a display issue, not data corruption.

   *Fix direction:* SKILL.md should remind the subagent to use `Bash cat` instead of the Read tool when verifying files with markdown syntax.

4. **Insufficient error recovery options**

   When the subagent launch fails, the current options are "retry / inline fallback / discard". This doesn't cover intermediate states where partial files exist (e.g., `report.md` written but `state.json` not updated).

   *Fix direction:* Add a "check partial results" option that detects and reuses existing partial files to continue the operation.

5. **Cross-compaction notification reliability**

   The standalone branch uses file-based notifications (`.reflect/notifications.md`). File-based notifications cannot be automatically injected into context after a compaction event.

   *Fix direction:* Consider adding a post-compaction check mechanism in SKILL.md (e.g., reading `state.json` to scan for `pending_review` status).

### Resolved Issues

- ~~**Background subagent write path bug**~~ — Resolved by write path redesign (v3). Background subagent now writes ONLY to project-local staging directory. All final writes occur in the interactive review session where `~/.claude/` access works normally.

- ~~**Multi-step process lacks atomicity**~~ — Resolved by merging all preparation steps into a single atomic Bash command. No interruption window exists between steps.

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
# standalone is the active branch (default)
```

## License

MIT
