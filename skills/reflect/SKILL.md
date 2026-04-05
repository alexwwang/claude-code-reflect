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
- User wants to extract a reusable skill (write directly to `.reflect/skills/`)
- User wants to organize existing knowledge (write directly to `.reflect/project-memory.json`)
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
- All drafts saved to `.reflect/reflections/{id}/` until approved
</Execution_Policy>

<Steps>

## Step 1: Route — Determine Mode

Check the invocation arguments:

- If arguments contain `review` followed by a reflection ID: skip to **Step 4** (Review Mode)
- If arguments contain `inspect` followed by a reflection ID: skip to **Step 3.3** (Inspect Mode)
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

### 3.1 Atomic Preparation + Launch

All setup steps are merged into a **single Bash call** to eliminate the interruption window where user messages could interleave and cause the subagent to never launch. `bypassPermissions` is applied to this single Bash call for atomicity — not for permission scope.

```bash
SESSION_ID=$(uuidgen) && \
mkdir -p {project_root}/.reflect/reflections/{reflection_id} && \
cat > {project_root}/.reflect/reflections/{reflection_id}/state.json << EOF
{
  "id": "{reflection_id}",
  "status": "running",
  "session_id": "$SESSION_ID",
  "created_at": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
  "error_claim": "{error_claim_summary}",
  "correction": "{correction_summary}"
}
EOF
cat > {project_root}/.reflect/reflections/{reflection_id}/prompt.txt << 'PROMPT_EOF'
{filled_subagent_prompt}
PROMPT_EOF
claude -p "$(cat {project_root}/.reflect/reflections/{reflection_id}/prompt.txt)" \
  --session-id $SESSION_ID \
  --permission-mode bypassPermissions \
  --output-format json \
  2>{project_root}/.reflect/reflections/{reflection_id}/stderr.log &
echo $SESSION_ID
```

Launch via Bash with `run_in_background=true`.

Key design points:
- **Single Bash call**: produces at most one permission confirmation in `ask` mode; `bypassPermissions` on this call eliminates even that, ensuring atomic execution
- **`--session-id`**: Creates a persistent, resumable session
- **`--permission-mode bypassPermissions`**: Required for the subagent to write output files (report.md, state.json, notifications.md) without interactive permission prompts. Safe because the subagent is restricted to project root only. Note: Claude Code has a confirmed bug where `bypassPermissions` on subagents silently denies writes outside the project root — this is acceptable since our subagent only writes within `.reflect/reflections/`
- **`--output-format json`**: Returns structured result (one-line summary from subagent)
- **`2>stderr.log`**: Captures errors for debugging
- **Background (`&`)**: Subagent runs independently; main conversation continues immediately
- **All paths in project root**: prompt.txt and stderr.log stored in the reflection staging directory, avoiding cross-platform temp path issues (no /tmp/ or /c/tmp/ dependency)

### 3.2 After Launch

After launching the subagent:
1. The session UUID is echoed by the atomic Bash command — capture it
2. Inform the user: "Reflection `{reflection_id}` launched in background. Inspect progress: `claude --resume {session_uuid}`"
3. Respond to the user normally — the subagent runs independently

When the subagent completes, a `<task-notification>` will appear automatically.

### 3.3 Inspect Mode (User-Initiated)

When the user invokes `/reflect inspect <id>`:

1. Read `state.json` for the session UUID and status
2. If `status: "running"`, show the user:
   - Session UUID
   - How to resume: `claude --resume {session_uuid}`
   - How to check logs: `cat {project_root}/.reflect/reflections/{id}/stderr.log`
   - Whether partial files exist (check if `report.md` exists)
3. If `status: "pending_review"`, suggest the user run `/reflect review <id>`

### 3.4 Error Handling

If the background task fails (detected via `<task-notification>` with `status: failed`):

1. Read the stderr log: `{project_root}/.reflect/reflections/{id}/stderr.log`
2. Read the task output file for the error message
3. **Classify the error**:
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

## First Step: Preserve Session State

Before writing anything, READ the existing `{project_root}/.reflect/reflections/{reflection_id}/state.json` file. This file was created by the preparation step and contains the real `session_id` and `created_at`. You MUST preserve these values exactly as-is when you update state.json in Task D. Do NOT use placeholders like "PLACEHOLDER" — use the actual values from the existing file.

## Path Restrictions (MANDATORY)

You MUST ONLY write files to:
1. `{project_root}/.reflect/reflections/{reflection_id}/` — ALL output goes here, including drafts

You MUST NOT write to:
- `~/.claude/` or any path outside the project root
- `.git/`, `CLAUDE.md`, or any configuration file
- Destructive commands: `rm -rf`, `git push --force`, `git reset --hard`
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

## Task A2: Scope Judgment

For each artifact you will draft in Task C, determine its scope:

- **user-level**: error reflects a general knowledge gap, language/API misunderstanding, or conceptual error that applies regardless of project
- **project-level**: error reflects missing project-specific context, codebase conventions, or domain constraints specific to this repository

Default scope by root cause category:
- Stale Knowledge → user-level
- Wrong Abstraction → user-level
- Assumption Error → evaluate: is the assumption project-specific? If yes → project-level
- Context Gap → project-level
- Scope Mismatch → project-level
- Under-Specification → project-level
- Over-Engineering → evaluate: is this a general tendency? If yes → user-level

You MUST provide for each artifact:
- `scope`: "user-level" or "project-level"
- `scope_reasoning`: one sentence explaining why

Record scope and scope_reasoning in both report.md and state.json.

## Task B: Severity Classification

Rate the error severity:

- **Critical**: Would cause bugs, deployment failures, data loss, or security issues
- **Important**: Wrong direction that wastes significant time
- **Minor**: Style, preference, naming, or formatting issues
- **Informational**: User preference update or minor clarification

## Task C: Draft Memory Artifacts

Draft artifacts based on severity. Save all draft content to `report.md` ONLY. Do NOT write artifact files to their final locations — target paths listed in each draft are for display at review time only. Actual writes happen after user approval in the interactive review session.

### C1: Reflection Report (required for all severities)

Save to: `{project_root}/.reflect/reflections/{reflection_id}/report.md`

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
- Scope: USER-LEVEL (or PROJECT-LEVEL)
- Scope reasoning: {one sentence explaining why this scope}
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

### Draft 2: ... (additional drafts based on severity, each with Scope + Scope reasoning)
```

### C2: Additional Drafts by Severity

**All severities** get at least Draft 1 (Claude Code feedback memory).

**Important+ severity** also drafts:
- A project-level factual note (Scope: PROJECT-LEVEL) to `.reflect/project-memory.json`

**Critical severity** also drafts:
- A high-priority session note (Scope: PROJECT-LEVEL) to `.reflect/notifications.md`

**If the correction reveals a reusable, non-Googleable, codebase-specific pattern** (quality gate: Could someone Google this in 5 minutes? No. Is this specific to THIS codebase? Yes. Did this take real debugging effort? Yes.), also draft:
- A learned skill file (Scope: PROJECT-LEVEL) for `.reflect/skills/{skill-name}.md`



## Task D: Save State and Inject Notification

1. Write state file to `{project_root}/.reflect/reflections/{reflection_id}/state.json`:
```json
{
  "id": "{reflection_id}",
  "status": "pending_review",
  "session_id": "{session_uuid}",
  "created_at": "{ISO_timestamp}",
  "category": "{category}",
  "severity": "{severity}",
  "error_claim": "{error_claim_summary}",
  "correction": "{correction_summary}",
  "artifacts": [
    {
      "id": "artifact-001",
      "type": "feedback_memory",
      "scope": "user-level",
      "scope_reasoning": "{one_sentence_explanation}",
      "name": "{artifact_name}",
      "target_path": "~/.claude/projects/{project_hash}/memory/feedback_{topic}.md",
      "content": "{artifact_content}"
    }
  ]
}
```

2. Use the Write tool to write a persistent notification to `{project_root}/.reflect/notifications.md`:
```
[Reflection complete ID:{reflection_id}] Root cause: {category} | Severity: {severity} | Enter /claude-code-reflect:reflect review {reflection_id} to review drafts and approve memory artifacts
```

3. IMPORTANT: Return ONLY a one-line summary: "Reflection {reflection_id} complete. Root cause: {category}. {N} memory drafts saved." Do NOT include full report or draft content in your return value — all content is already saved to files.

</SUBAGENT_PROMPT>

After launching the subagent, respond to the user normally in the main conversation. The subagent runs independently. When it completes, a `<task-notification>` will appear automatically. The notification file in `.reflect/notifications.md` ensures the review prompt can be found later.

## Step 4: Review Mode (User-Initiated)

When the user invokes `/claude-code-reflect:reflect review <id>` or mentions reviewing a reflection ID:

### 4.1 Load Reflection

Read `{project_root}/.reflect/reflections/{id}/report.md`

If the file does not exist, inform the user and suggest checking the ID. List available reflections by scanning `.reflect/reflections/*/state.json`.

### 4.2 Display Results

Present the reflection report to the user in a clear format:

1. **Root Cause Analysis** — category, root cause, reasoning chain
2. **Prevention recommendation**
3. **Severity rating**
4. **Drafted memory artifacts** — for each artifact, display:

```
[artifact-001] {artifact_name}
Scope:         {USER-LEVEL or PROJECT-LEVEL}
Reason:        {scope_reasoning}
Target:        {target_path}
```

Keep the display concise. Show full artifact content only for artifacts the user asks about.

### 4.3 Collect User Decision

Use AskUserQuestion with these options:

- **Approve all** — Write all drafted artifacts to their target locations (proceed to Step 5)
- **Approve with modifications** — User describes what to change, then write modified versions (proceed to Step 5)
- **Change scope** — User changes one or more artifacts between user-level and project-level. Update target path accordingly:
  - user-level → project-level: target changes from `~/.claude/projects/.../memory/` to `.reflect/project-memory.json`
  - project-level → user-level: target changes from `.reflect/...` to `~/.claude/projects/.../memory/`
  - Update `state.json` artifact entries with new scope before writing
- **Re-analyze** — User provides additional context, launch new subagent with `--deep` flag (loop to Step 3)
- **Discard** — Delete the reflection drafts and state file, no artifacts written

## Step 5: Commit Artifacts (Write to Final Locations)

After user approval, write each approved draft to its final location. **All writes execute in the interactive (resumed) session** — this is the natural execution boundary where `~/.claude/` writes work without special permissions.

Read target paths from `state.json` artifacts array (not hardcoded paths), then write:

| Draft Type | Write Method | Final Location |
|------------|-------------|----------------|
| Claude Code feedback memory | Write tool | `~/.claude/projects/{project}/memory/feedback_{topic}.md` |
| Project memory note | Write tool | `.reflect/project-memory.json` |
| Project memory directive | Write tool | `.reflect/project-memory.json` |
| High-priority notification | Write tool | `.reflect/notifications.md` |
| Learned skill | Write tool | `.reflect/skills/{skill-name}.md` |

If a Claude Code feedback memory was written, also update `MEMORY.md` index in the same memory directory with a pointer to the new file.

### Cleanup

1. Update `state.json`: set `status` to `"approved"` (or `"approved_with_modifications"`), add `approved_at` timestamp, set each artifact's `status` to `"written"` and record `final_path`
2. Overwrite `.reflect/notifications.md` with a brief "Reflection {id} artifacts committed." message
3. Do NOT delete the report.md — it serves as a historical record

</Steps>

<Tool_Usage>
```bash
# Step 3.1: Atomic preparation + launch (single Bash call)
SESSION_ID=$(uuidgen) && \
mkdir -p {project_root}/.reflect/reflections/{reflection_id} && \
cat > {project_root}/.reflect/reflections/{reflection_id}/state.json << EOF
{state_json_content}
EOF
cat > {project_root}/.reflect/reflections/{reflection_id}/prompt.txt << 'PROMPT_EOF'
{filled_subagent_prompt}
PROMPT_EOF
claude -p "$(cat {project_root}/.reflect/reflections/{reflection_id}/prompt.txt)" \
  --session-id $SESSION_ID \
  --permission-mode bypassPermissions \
  --output-format json \
  2>{project_root}/.reflect/reflections/{reflection_id}/stderr.log &
echo $SESSION_ID

# Step 3.3: User inspects subagent progress
claude --resume {session_uuid}
cat {project_root}/.reflect/reflections/{id}/stderr.log
```

```python
# Step 4: Read reflection report
Read(file_path=".reflect/reflections/{id}/report.md")
Read(file_path=".reflect/reflections/{id}/state.json")

# Step 5: Write approved artifacts
Write(file_path="~/.claude/projects/{project}/memory/feedback_{topic}.md", content=...)
Edit(file_path="~/.claude/projects/{project}/memory/MEMORY.md", ...)
Write(file_path=".reflect/project-memory.json", content=...)
Write(file_path=".reflect/project-memory.json", content=...)
Write(file_path=".reflect/notifications.md", content=...)
```
</Tool_Usage>

<Examples>
<Good>
Complete async review flow with scope display:
```
[Turn 1] User: 不对，这个 API 的参数应该是 callback 而不是 promise
[Turn 1] Claude: 感谢纠正！[detects correction → atomic preparation → launches background session]
[Turn 1] Claude: Reflection ref-20260404a launched. Inspect: claude --resume {uuid}
[Turn 1] Claude: [main conversation continues normally]

[User opens new terminal]
$ claude --resume {uuid}
[Sees subagent's full RCA work in progress]

[Background completes] Notification: "[Reflection complete ID:ref-20260404a]..."

[Turn 3] User: /claude-code-reflect:reflect review ref-20260404a
[Turn 3] Claude: [reads report + state.json, displays]
  Root Cause: Wrong Abstraction
  Severity: Important

  [artifact-001] callback-vs-promise
  Scope:   USER-LEVEL
  Reason:  Language-level misunderstanding, not project-specific
  Target:  ~/.claude/projects/.../memory/feedback_callback-vs-promise.md

[Turn 3] User: Approve all
[Turn 3] Claude: [writes all artifacts to final locations in interactive session]
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
  Error log: [shows stderr from .reflect/reflections/{id}/stderr.log]
  Partial files: report.md exists (subagent started writing)
[Turn 2] Claude: Subagent appears to be running. Check logs or wait for completion.
```
</Good>

<Good>
User changes scope before approving:
```
[Turn 5] User: /claude-code-reflect:reflect review ref-20260404a
[Turn 5] Claude: [displays results with scope per artifact]
  [artifact-001] go-append-isolation
  Scope:   USER-LEVEL
  Reason:  Go language semantics, applies globally
  Target:  ~/.claude/projects/.../memory/feedback_go-append.md
[Turn 5] User: Change scope — this project has a custom slice wrapper, make it project-level
[Turn 5] Claude: [updates scope to project-level, changes target to .reflect/project-memory.json,
                  updates state.json artifact entry, writes modified version]
```
</Good>

<Good>
Background task fails with clear error:
```
[task-notification: status=failed]
[Turn 2] Claude: Reflection ref-20260404a failed.
  Error: API Error: Unable to connect to API (ECONNRESET)
  Stderr log: {project_root}/.reflect/reflections/ref-20260404a/stderr.log
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
- If the subagent keeps failing, offer inline execution as fallback (run RCA directly in main conversation instead of background subagent)
- If `.reflect/reflections/` accumulates more than 20 pending reflections, suggest the user review or clean up old ones
- If the subagent appears stuck, the user can inspect via `/reflect inspect <id>` or `claude --resume <session_uuid>`
</Escalation_And_Stop_Conditions>

<Final_Checklist>
- [ ] Correction signal detected from conversation (or user provided explicit context)
- [ ] Preparation completed atomically (single Bash call with bypassPermissions for atomicity)
- [ ] Session UUID generated and saved to state.json
- [ ] Background Claude session launched with --session-id and --permission-mode bypassPermissions
- [ ] Reflection report saved to `.reflect/reflections/{id}/report.md` with scope judgments per artifact
- [ ] State file updated to `pending_review` with artifacts array (scope/scope_reasoning per artifact)
- [ ] Notification file written to `.reflect/notifications.md` with review instructions
- [ ] User reviewed reflection report with scope display per artifact
- [ ] User approved artifacts (or approved with modifications or scope override)
- [ ] All approved artifacts written to final locations in interactive (resumed) session
- [ ] MEMORY.md index updated if feedback memory was written
- [ ] State file updated to `approved`
- [ ] Review notification cleaned up from `.reflect/notifications.md`
</Final_Checklist>

<Advanced>
## Integration with Other Skills

This standalone version does not delegate to OMC skills. All artifact writing is done directly with the Write tool. Compatible with OMC — each system manages its own data directory (`.reflect/` vs `.omc/`).

| Scenario | Action | Reason |
|----------|--------|--------|
| Correction reveals a reusable pattern | Write to `.reflect/skills/` | Direct file write, no quality gate |
| Multiple artifacts needed | Write each to its target path | No classification, all direct writes |
| Simple single artifact write | Direct Write tool call | Faster path, no delegation overhead |

## Debugging Guide

When a background subagent fails or appears stuck:

1. **Check state**: Read `.reflect/reflections/{id}/state.json` — is status "running"?
2. **Resume session**: `claude --resume {session_uuid}` — see the subagent's full conversation
3. **Check stderr**: `cat {project_root}/.reflect/reflections/{id}/stderr.log` — API errors, permission issues
4. **Check partial output**: Does `report.md` exist? Is it complete or truncated?
5. **Check task output**: Read the Bash task output file for the subagent's return value
6. **Classifier unavailability**: If errors mention "temporarily unavailable", the internal safety classifier was down. This should be resolved by `bypassPermissions` mode — verify the launch command uses it.
7. **Fallback**: If subagent keeps failing, offer inline execution as fallback

## Future Enhancements

1. **Auto-trigger via hook**: Add a `model-correction` pattern type to `detector.ts` with correction-specific regex patterns
2. **Reflection history dashboard**: Add a `/reflect history` command to browse past reflections
3. **Pattern aggregation**: Detect when the same root cause category appears repeatedly and generate a meta-reflection
</Advanced>
