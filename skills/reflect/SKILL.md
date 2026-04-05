---
name: reflect
description: Detect model corrections, delegate RCA to background subagent, async review with user approval
argument-hint: "[review <id>] [inspect <id>] [--deep] [correction context]"
level: 3
---

<Purpose>
When a user corrects a model error, launch a background subagent to perform root cause analysis and draft memory artifacts. The subagent runs as a persistent Claude session — the user can inspect progress at any time by resuming the session. Results are saved to disk for asynchronous user review; the main conversation is never blocked or polluted.

Reflect fills the gap between `learner` (extracts reusable skills) and `remember` (classifies knowledge): it detects corrections, analyzes WHY the error happened, and generates prevention-focused memory documents.
</Purpose>

<Use_When>
- User manually invokes `/claude-code-reflect:reflect`
- User wants to reflect on a correction that just happened
- User invokes `/claude-code-reflect:reflect review <id>` to review a completed reflection
- User invokes `/claude-code-reflect:reflect inspect <id>` to check background subagent progress
</Use_When>

<Do_Not_Use_When>
- Normal discussion disagreement (not a factual error)
- User is testing or probing
- No error has occurred
- User wants to extract a reusable skill (use `/learner` directly)
- User wants to organize existing knowledge (use `/remember` directly)
</Do_Not_Use_When>

<Why_This_Exists>
The existing `learner` skill detects problem-solution patterns but does not specifically recognize user corrections of model errors. It captures WHAT the solution was, but not WHY the error happened (assumption error, context gap, wrong abstraction, etc.). Reflect adds:
1. Correction-specific signal detection (multilingual)
2. Structured root cause analysis taxonomy
3. Async review loop — results saved to disk, user reviews on their own schedule
4. Memory artifact generation with user approval gate
5. Session-resumable subagent for full debuggability
</Why_This_Exists>

<Execution_Policy>
- Phase 1 (detection): Execute in main conversation, instant (<1 second)
- Phase 2 (reflection): Delegate to background Claude session (`--session-id`), user can inspect via `claude --resume`
- Phase 3 (review): User-initiated via `/reflect review <id>`, lightweight read + display
- Memory artifacts are written to final locations ONLY after user approval
- All drafts saved to `.omc/reflections/{id}/` until approved
</Execution_Policy>

<Steps>

## Step 1: Route — Determine Mode

Check the invocation arguments:

- If arguments contain `review` followed by a reflection ID: skip to **Step 4** (Review Mode)
- If arguments contain `inspect` followed by a reflection ID: skip to **Step 3.5** (Inspect Mode)
- If arguments contain `--deep`: set deep analysis flag
- Otherwise: proceed to **Step 2** (Detection + Reflection Mode)

## Step 2: Quick Detection (Main Conversation, Instant)

### 2.1 Scan for Correction Signals

Scan the last 3-5 conversation turns and match against the Correction Signal Taxonomy:

| Type | English | Chinese |
|------|---------|---------|
| Direct negation | "Wrong", "Incorrect", "That's not right", "No, that's wrong" | "不对", "错了", "搞错了", "不是这样的" |
| Redirection | "I said X, not Y", "I meant...", "What I actually wanted" | "我说的是X不是Y", "我的意思是...", "我要的其实是..." |
| Explicit correction | "You made a mistake", "You got this wrong" | "你说错了", "你理解错了", "不是这个问题" |
| Implicit correction | "Actually, it should be...", "The correct approach is..." | "实际上应该...", "正确做法是...", "重新看看..." |

### 2.2 Handle No Correction Found

If no correction signal is detected, use AskUserQuestion to ask the user what they want to reflect on. Do not fabricate corrections.

### 2.3 Extract Context

When a correction is detected, extract:

- `error_claim`: What Claude said that was wrong (the specific claim, code, or assumption)
- `correction`: What the user said is correct
- `context_excerpt`: Relevant conversation summary (keep under 500 characters)

### 2.4 Generate Reflection ID

Format: `ref-{YYYYMMDD}{sequence_letter}` (e.g., `ref-20260404a`)

### 2.5 Launch Background Subagent

Proceed to **Step 3** — fire the subagent immediately, do not wait.

## Step 3: Session-Resumable Background Subagent

Launch a background subagent as a persistent Claude session. The session is saved to disk so the user can inspect progress at any time by opening a new terminal and running `claude --resume <session_id>`.

### 3.1 Prepare

1. Generate a UUID for the subagent session. Use `uuidgen` via Bash.
2. Create the reflections directory: `mkdir -p {project_root}/.omc/reflections/{reflection_id}`
3. Initialize `state.json` with `status: "running"` and the session ID:

```json
{
  "id": "{reflection_id}",
  "status": "running",
  "session_id": "{generated_uuid}",
  "category": null,
  "severity": null,
  "created_at": "{ISO_timestamp}",
  "error_claim": "{error_claim_summary}",
  "correction": "{correction_summary}"
}
```

### 3.2 Write Subagent Prompt to Temp File

Write the filled-in SUBAGENT_PROMPT (with all `{variables}` replaced) to a temp file. Use Bash heredoc to avoid path mapping issues:

```bash
cat > /c/tmp/reflect-{reflection_id}-prompt.txt << 'PROMPT_EOF'
{filled_subagent_prompt}
PROMPT_EOF
```

Note: On Windows with Git Bash, use `/c/tmp/` not `/tmp/` to ensure both Write tool and Bash reference the same path.

### 3.3 Launch Background Claude Process

Execute the subagent in background:

```bash
claude -p "$(cat /c/tmp/reflect-{reflection_id}-prompt.txt)" \
  --session-id {session_uuid} \
  --permission-mode bypassPermissions \
  --output-format json \
  2>/c/tmp/reflect-{reflection_id}-stderr.log
```

Launch via Bash with `run_in_background=true`.

Key flags:
- `--session-id {uuid}`: Creates a persistent, resumable session
- `--permission-mode bypassPermissions`: Bypasses safety classifier (subagent prompt has self-imposed path restrictions)
- `--output-format json`: Returns structured result (one-line summary from subagent)
- `2>stderr.log`: Captures errors for debugging

### 3.4 After Launch

After launching the subagent:
1. Save the session UUID in `state.json` (already done in 3.1)
2. Inform the user: "Reflection `{reflection_id}` launched in background. Inspect progress: `claude --resume {session_uuid}`"
3. Respond to the user normally — the subagent runs independently

When the subagent completes, a `<task-notification>` will appear automatically.

### 3.5 Inspect Mode (User-Initiated)

When the user invokes `/reflect inspect <id>`:

1. Read `state.json` for the session UUID and status
2. If `status: "running"`, show the user:
   - Session UUID
   - How to resume: `claude --resume {session_uuid}`
   - How to check logs: `cat /c/tmp/reflect-{id}-stderr.log`
   - Whether partial files exist (check if `report.md` exists)
3. If `status: "pending_review"`, suggest the user run `/reflect review <id>`

### 3.6 Error Handling

If the background task fails (detected via `<task-notification>` with `status: failed`):

1. Read the stderr log: `/c/tmp/reflect-{id}-stderr.log`
2. Read the task output file for the error message
3. **Classify the error**:
   - **Classifier unavailability**: Message contains "temporarily unavailable" + "auto mode cannot"
     → Should not happen with `bypassPermissions`. If it does, report to user as a platform issue.
   - **API connectivity**: Message contains "ECONNRESET", "ECONNREFUSED", "timeout", or "Unable to connect"
     → Transient network issue. Offer retry immediately with a new session UUID.
   - **Rate limit**: Message contains "rate limit", "quota", or "429"
     → Suggest waiting and retrying. Do not auto-retry rate limits.
   - **Other errors**: Report full error to user with stderr log contents.
4. Check if partial files were written (report.md may exist even if subagent crashed)
5. Report the failure details to the user
6. Offer options: retry (with new session UUID), fall back to inline execution, or discard

<SUBAGENT_PROMPT>
You are a reflection agent. Your task is to analyze a model error, perform root cause analysis, and draft memory artifacts. Save all drafts to disk — do NOT write to final memory locations.

IMPORTANT: After saving files, return ONLY a one-line summary in this exact format:
"Reflection {id} complete. Root cause: {category}. {N} drafts saved."
Do NOT include the full report content, Do NOT include draft artifact text. Just the one-line summary.

## Path Restrictions (MANDATORY)

You MUST ONLY write files to these locations:
1. `{project_root}/.omc/reflections/{reflection_id}/` — reflection reports and state
2. Memory directories under `~/.claude/projects/` — feedback memory files

You MUST NOT:
- Write to any path outside the project root or user memory directories
- Execute destructive commands (rm -rf, git push --force, git reset --hard)
- Modify `.git/`, `CLAUDE.md`, or any configuration file
- Install packages or make network requests

## Error Context
- Reflection ID: {reflection_id}
- Error claim: {error_claim}
- User correction: {correction}
- Conversation excerpt: {context_excerpt}
- Project root: {project_root}
- Deep analysis mode: {deep_flag}

## Task A: Root Cause Analysis

Classify the error into exactly one of these categories:

1. **Assumption Error** — Claude assumed facts not in evidence (e.g., assumed library version, assumed file structure)
2. **Context Gap** — Claude missed or forgot relevant context from earlier in the conversation
3. **Wrong Abstraction** — Applied a correct principle at the wrong level of abstraction
4. **Scope Mismatch** — Solved a different problem than what was asked
5. **Stale Knowledge** — Applied outdated information (API changed, deprecated pattern)
6. **Over-Engineering** — Added unnecessary complexity where simple code would do
7. **Under-Specification** — Answer was correct but incomplete

For deep analysis mode, classify into a primary category AND identify contributing secondary categories.

Write out:
- Error category (primary + secondary if deep)
- Root cause description (one sentence)
- Reasoning chain (2-3 steps that led to the error)
- Prevention recommendation (specific, actionable)

## Task B: Severity Classification

Rate the error severity:

- **Critical**: Would cause bugs, deployment failures, data loss, or security issues
- **Important**: Wrong direction that wastes significant time
- **Minor**: Style, preference, naming, or formatting issues
- **Informational**: User preference update or minor clarification

## Task C: Draft Memory Artifacts

Draft artifacts based on severity. Do NOT write to final locations. Save everything to the report.

### C1: Reflection Report (required for all severities)

Save to: `{project_root}/.omc/reflections/{reflection_id}/report.md`

Use this exact format:

```markdown
# Reflection Report {reflection_id}

## Error
{error_claim}

## Correction
{correction}

## Root Cause Analysis
- Category: {category}
- Root cause: {root_cause_one_liner}
- Reasoning chain:
  1. {step_1}
  2. {step_2}
  3. {step_3}

## Prevention
{prevention_recommendation}

## Severity: {severity}

---

## Drafted Memory Artifacts

### Draft 1: Claude Code Feedback Memory
- Target: ~/.claude/projects/{project_hash}/memory/feedback_{topic}.md
- Content:
---
name: {memory_name}
description: {one_line_description}
type: feedback
---

{rule_statement}

**Why:** {why_this_error_happened_and_matters}

**How to apply:** {when_and_where_to_apply_this_lesson}

### Draft 2: ... (additional drafts based on severity)
```

### C2: Additional Drafts by Severity

**All severities** get at least Draft 1 (Claude Code feedback memory).

**Important+ severity** also drafts:
- A `project_memory_add_note` entry (project-level factual note)

**Critical severity** also drafts:
- A `notepad_write_priority` entry (high-priority session note)

**If the correction reveals a reusable, non-Googleable, codebase-specific pattern** (test against learner's quality gate: Could someone Google this in 5 minutes? No. Is this specific to THIS codebase? Yes. Did this take real debugging effort? Yes.), also draft:
- An OMC learned skill file for `.omc/skills/{skill-name}.md`

## Task D: Save State and Inject Notification

1. Write state file to `{project_root}/.omc/reflections/{reflection_id}/state.json`:
```json
{
  "id": "{reflection_id}",
  "status": "pending_review",
  "session_id": "{session_uuid}",
  "category": "{category}",
  "severity": "{severity}",
  "created_at": "{ISO_timestamp}",
  "error_claim": "{error_claim_summary}",
  "correction": "{correction_summary}"
}
```

2. Use the `notepad_write_priority` MCP tool to inject a persistent notification:
```
[Reflection complete ID:{reflection_id}] Root cause: {category} | Severity: {severity} | Enter /claude-code-reflect:reflect review {reflection_id} to review drafts and approve memory artifacts
```

3. Create the reflections directory if it does not exist.

4. IMPORTANT: Return ONLY a one-line summary: "Reflection {reflection_id} complete. Root cause: {category}. {N} memory drafts saved." Do NOT include full report or draft content in your return value — all content is already saved to files.

</SUBAGENT_PROMPT>

After launching the subagent, respond to the user normally in the main conversation. The subagent runs independently. When it completes, a `<task-notification>` will appear automatically. The notepad priority notification ensures the review prompt survives context compaction.

## Step 4: Review Mode (User-Initiated)

When the user invokes `/claude-code-reflect:reflect review <id>` or mentions reviewing a reflection ID:

### 4.1 Load Reflection

Read `{project_root}/.omc/reflections/{id}/report.md`

If the file does not exist, inform the user and suggest checking the ID. List available reflections by scanning `.omc/reflections/*/state.json`.

### 4.2 Display Results

Present the reflection report to the user in a clear format:

1. **Root Cause Analysis** — category, root cause, reasoning chain
2. **Prevention recommendation**
3. **Severity rating**
4. **Drafted memory artifacts** — list each draft with its target location and content

Keep the display concise. Show full artifact content only for artifacts the user asks about.

### 4.3 Collect User Decision

Use AskUserQuestion with these options:

- **Approve all** — Write all drafted artifacts to final locations (proceed to Step 5)
- **Approve with modifications** — User describes what to change, then write modified versions (proceed to Step 5)
- **Re-analyze** — User provides additional context, launch new subagent with `--deep` flag (loop to Step 3)
- **Discard** — Delete the reflection drafts and state file, no artifacts written

## Step 5: Commit Artifacts (Write to Final Locations)

After user approval, write each approved draft to its final location:

| Draft Type | Write Method | Final Location |
|------------|-------------|----------------|
| Claude Code feedback memory | Write tool | `~/.claude/projects/{project}/memory/feedback_{topic}.md` |
| Project memory note | `project_memory_add_note` MCP | `.omc/project-memory.json` |
| Project memory directive | `project_memory_add_directive` MCP | `.omc/project-memory.json` |
| High-priority notepad entry | `notepad_write_priority` MCP | `.omc/notepad.md` |
| OMC learned skill | Write tool | `.omc/skills/{skill-name}.md` |

If a Claude Code feedback memory was written, also update `MEMORY.md` index in the same memory directory with a pointer to the new file.

### Cleanup

1. Update `state.json` status to `"approved"` or `"approved_with_modifications"`
2. Use `notepad_write_priority` to remove or update the review notification (replace with a brief "Reflection {id} artifacts committed" note)
3. Do NOT delete the report.md — it serves as a historical record

</Steps>

<Tool_Usage>
```bash
# Step 3.1: Generate session UUID and initialize state
SESSION_ID=$(uuidgen)
mkdir -p {project_root}/.omc/reflections/{reflection_id}
# Write state.json with status: "running" and session_id

# Step 3.2: Write prompt to temp file (use /c/tmp/ on Windows)
cat > /c/tmp/reflect-{reflection_id}-prompt.txt << 'PROMPT_EOF'
{filled_subagent_prompt}
PROMPT_EOF

# Step 3.3: Launch background Claude session
claude -p "$(cat /c/tmp/reflect-{reflection_id}-prompt.txt)" \
  --session-id {session_uuid} \
  --permission-mode bypassPermissions \
  --output-format json \
  2>/c/tmp/reflect-{reflection_id}-stderr.log

# Step 3.5: User inspects subagent progress
claude --resume {session_uuid}
cat /c/tmp/reflect-{reflection_id}-stderr.log
```

```python
# Step 4: Read reflection report
Read(file_path=".omc/reflections/{id}/report.md")
Read(file_path=".omc/reflections/{id}/state.json")

# Step 5: Write approved artifacts
Write(file_path="~/.claude/projects/{project}/memory/feedback_{topic}.md", content=...)
Edit(file_path="~/.claude/projects/{project}/memory/MEMORY.md", ...)
project_memory_add_note(content=...)
project_memory_add_directive(directive=...)
notepad_write_priority(content=...)
```
</Tool_Usage>

<Examples>
<Good>
Complete async review flow with session inspection:
```
[Turn 1] User: 不对，这个 API 的参数应该是 callback 而不是 promise
[Turn 1] Claude: 感谢纠正！[detects correction → launches background session]
[Turn 1] Claude: Reflection ref-20260404a launched. Inspect: claude --resume {uuid}
[Turn 1] Claude: [main conversation continues normally]

[User opens new terminal]
$ claude --resume {uuid}
[Sees subagent's full RCA work in progress]

[Background completes] Notepad notification: "[Reflection complete ID:ref-20260404a]..."

[Turn 3] User: /claude-code-reflect:reflect review ref-20260404a
[Turn 3] Claude: [reads report, displays RCA + drafted artifacts]
[Turn 3] User: 确认，全部写入
[Turn 3] Claude: [writes all artifacts to final locations]
```
</Good>

<Good>
User inspects stuck subagent:
```
[Turn 2] User: /claude-code-reflect:reflect inspect ref-20260404a
[Turn 2] Claude: [reads state.json]
  Status: running
  Session UUID: {uuid}
  Resume: claude --resume {uuid}
  Error log: [shows stderr contents]
  Partial files: report.md exists (subagent started writing)
[Turn 2] Claude: Subagent appears to be running. Check logs or wait for completion.
```
</Good>

<Good>
User modifies before approving:
```
[Turn 5] User: /claude-code-reflect:reflect review ref-20260404a
[Turn 5] Claude: [displays results]
[Turn 5] User: 根因分析没问题，但记忆文档要加一条：这个项目用的是 v2.x 不是 v3.x
[Turn 5] Claude: [modifies drafts based on feedback → writes modified versions to final locations]
```
</Good>

<Good>
Background task fails with clear error:
```
[task-notification: status=failed]
[Turn 2] Claude: Reflection ref-20260404a failed.
  Error: API Error: Unable to connect to API (ECONNRESET)
  Stderr log: /c/tmp/reflect-ref-20260404a-stderr.log
  Partial files: none
  Options: [Retry] [Inline fallback] [Discard]
```
</Good>

<Bad>
Running full RCA in main conversation:
```
[Turn 1] User: 不对
[Turn 1] Claude: [500 lines of RCA analysis polluting the main context window...]
```
Why bad: Reflection content bloats the main conversation, wastes context window, and prevents the user from continuing their work.
</Bad>

<Bad>
Writing artifacts without user approval:
```
[Background subagent completes]
[Immediately writes all drafts to final memory locations]
```
Why bad: User has no opportunity to review, correct, or reject the reflection. Artifacts may be inaccurate or unwanted.
</Bad>

<Bad>
No way to debug a stuck subagent:
```
[Background task hangs for 10 minutes]
[Main conversation has no idea what happened]
[No logs, no session to resume, no error output]
```
Why bad: User and main agent are blind to subagent state. Cannot distinguish "still working" from "stuck" from "crashed".
</Bad>
</Examples>

<Escalation_And_Stop_Conditions>
- If the reflection report file does not exist when reviewing, list available reflections and ask the user to pick one
- If the user provides contradictory feedback across multiple review rounds, ask for clarification before proceeding
- If the subagent fails 3 times in a row, report the issue to the user and suggest running `/reflect` manually with explicit context
- If `bypassPermissions` mode also fails, offer inline execution as fallback (run RCA directly in main conversation instead of background subagent)
- If `.omc/reflections/` accumulates more than 20 pending reflections, suggest the user review or clean up old ones
- If the subagent appears stuck, the user can inspect via `/reflect inspect <id>` or `claude --resume <session_uuid>`
</Escalation_And_Stop_Conditions>

<Final_Checklist>
- [ ] Correction signal detected from conversation (or user provided explicit context)
- [ ] Session UUID generated and saved to state.json
- [ ] Background Claude session launched with --session-id and --permission-mode bypassPermissions
- [ ] Reflection report saved to `.omc/reflections/{id}/report.md`
- [ ] State file updated to `pending_review`
- [ ] Notepad priority notification injected with review instructions
- [ ] User reviewed reflection report
- [ ] User approved artifacts (or approved with modifications)
- [ ] All approved artifacts written to final locations
- [ ] MEMORY.md index updated if feedback memory was written
- [ ] State file updated to `approved`
- [ ] Review notification cleaned up from notepad
</Final_Checklist>

<Advanced>
## Integration with Other Skills

| Scenario | Delegate to | Reason |
|----------|------------|--------|
| Correction reveals a reusable non-Googleable pattern | `/learner` | Learner has quality gate validation |
| Multiple artifacts need classification | `/remember` | Remember classifies by memory surface |
| Correction reveals a repeatable workflow | `/skillify` | Skillify creates formal skill drafts |
| Simple single artifact write | Direct MCP tool call | Faster path, no delegation overhead |

## Debugging Guide

When a background subagent fails or appears stuck:

1. **Check state**: Read `.omc/reflections/{id}/state.json` — is status "running"?
2. **Resume session**: `claude --resume {session_uuid}` — see the subagent's full conversation
3. **Check stderr**: `cat /c/tmp/reflect-{id}-stderr.log` — API errors, permission issues
4. **Check partial output**: Does `report.md` exist? Is it complete or truncated?
5. **Check task output**: Read the Bash task output file for the subagent's return value
6. **Classifier unavailability**: If errors mention "temporarily unavailable", the internal safety classifier was down. This should be resolved by `bypassPermissions` mode — verify the launch command uses it.
7. **Fallback**: If subagent keeps failing, offer inline execution as fallback

## Future Enhancements

1. **Auto-trigger via hook**: Add a `model-correction` pattern type to `detector.ts` with correction-specific regex patterns
2. **Reflection history dashboard**: Add a `/reflect history` command to browse past reflections
3. **Pattern aggregation**: Detect when the same root cause category appears repeatedly and generate a meta-reflection
</Advanced>
