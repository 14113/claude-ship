---
name: ship
description: Feature workflow from description to PR — autonomous after discovery
argument-hint: "[feature description in free text]"
---

# Ship Feature

Complete autonomous workflow: idea → spec → implement → verify → PR.
User interaction only during discovery (clarifying questions). Everything else runs autonomously.

## Arguments

- `$ARGUMENTS` - Feature description (free text, Czech or English)

## Design Principles

1. **Main window = orchestrator only**. Never read code files, never run tests, never see verbose output.
2. **All heavy work in subagents**
3. **1-line protocol**. Subagents return structured 1-line responses. Exception: spec-shaper returns JSON (parsed by orchestrator).
4. **Parallel by default**. Independent work launches in a single message.
5. **Auto-retry up to 3x** on verification failures.
6. **File-based IPC**. Subagents communicate through spec folder files, not Task API.
7. **Swarm for research, single agent for implementation**. Parallel Explore agents gather codebase context fast. One feature-builder implements everything for consistency.

---

## PHASE 0: CONTEXT

Determine starting point. Delegate to spec-initializer agent (handles folder creation, resume detection, git branching).

### Step 0.1: Initialize

Extract `SESSION_ID` from the scratchpad path in your system prompt (the UUID segment, e.g. `ada759e9-fcd4-4d17-baad-7856d45c5e3f`).

```
Task — subagent_type: "spec-initializer"
  model: "haiku"
  prompt: "Feature: $ARGUMENTS | session: SESSION_ID"
```

Parse the 1-line response:

- `INIT: NEW | spec: X | branch: Y` → Store `SPEC_PATH` and `BRANCH_NAME` → proceed to Phase 1
- `INIT: EXISTING | specs: [list]` → go to Step 0.2
- `INIT: RESUME | spec: X | branch: Y | resume_from: Z` → Store and skip to indicated phase

### Step 0.2: Handle existing specs (only if INIT: EXISTING)

Ask user: continue one of the existing specs, or start new?

If user chooses existing:
```
Resume spec-initializer [agent ID]:
  prompt: "Resume spec: [chosen spec path] | session: SESSION_ID"
```

If user chooses new:
```
Resume spec-initializer [agent ID]:
  prompt: "New spec: $ARGUMENTS | session: SESSION_ID"
```

Parse 1-line response → store `SPEC_PATH` and `BRANCH_NAME`.

---

## PHASE 1: DISCOVER

**Goal**: Understand the feature. Only phase with user interaction.

### Step 1.0: Swarm codebase research (parallel research agents)

Launch 3 parallel general-purpose agents (sonnet) to pre-gather codebase context and write findings to spec folder files. File-based IPC keeps orchestrator context clean — orchestrator only sees 1-line responses, never research content.

Extract 3-5 keywords from the feature description (model names, domain terms, action verbs).

```
Task A — subagent_type: "general-purpose"
  model: "sonnet"
  prompt: "Search the codebase for data layer context related to: $ARGUMENTS
  Focus on: models, migrations, database schema, associations, validations.
  Search in: app/models/, db/migrate/, db/schema.rb
  Keywords: [extracted keywords]
  Use Glob and Grep to find relevant files, Read to examine them.
  Write findings to: SPEC_PATH/planning/research-data.md (use Write tool)
  Format: markdown list of files with 1-line descriptions. Include key model names, column types, and associations found.
  CRITICAL: Return EXACTLY 1 line: RESEARCH: DATA | files: N"

Task B — subagent_type: "general-purpose"
  model: "sonnet"
  prompt: "Search the codebase for business logic context related to: $ARGUMENTS
  Focus on: services, queries, jobs, mailers, policies.
  Search in: app/services/, app/queries/, app/jobs/, app/mailers/, app/policies/
  Keywords: [extracted keywords]
  Use Glob and Grep to find relevant files, Read to examine them.
  Write findings to: SPEC_PATH/planning/research-logic.md (use Write tool)
  Format: markdown list of files with 1-line descriptions. Include key method signatures and patterns found.
  CRITICAL: Return EXACTLY 1 line: RESEARCH: LOGIC | files: N"

Task C — subagent_type: "general-purpose"
  model: "sonnet"
  prompt: "Search the codebase for presentation layer context related to: $ARGUMENTS
  Focus on: controllers, views, routes, Stimulus controllers, components.
  Search in: app/controllers/, app/presenters/, app/forms/, app/decorators/, app/helpers/, app/views/, config/routes.rb, app/javascript/
  Keywords: [extracted keywords]
  Use Glob and Grep to find relevant files, Read to examine them.
  Write findings to: SPEC_PATH/planning/research-presentation.md (use Write tool)
  Format: markdown list of files with 1-line descriptions. Include route patterns and view structure found.
  CRITICAL: Return EXACTLY 1 line: RESEARCH: PRESENTATION | files: N"
```

Parse 1-line responses. Do NOT read the research files — they stay in the spec folder for spec-shaper.

### Step 1.1: Launch spec-shaper with file references only

```
Task — subagent_type: "spec-shaper"
  model: "opus"
  prompt: "Feature: $ARGUMENTS
  Spec folder: SPEC_PATH

  Pre-gathered codebase context is in these files (read them first):
  - SPEC_PATH/planning/research-data.md
  - SPEC_PATH/planning/research-logic.md
  - SPEC_PATH/planning/research-presentation.md

  Use this context as a starting point, then do your own targeted supplemental search
  to fill gaps — especially cross-cutting concerns, system-level patterns, and
  architectural context that per-layer searches may miss."
```

Agent has its own output format (see `.claude/agents/spec-shaper.md`).
Keep spec-shaper **agent ID** for resuming.

### Step 1.2: Ask user questions

Parse JSON from spec-shaper. Present questions using AskUserQuestion with options from the agent's response.
**Wait for user response.**

### Step 1.3: Complete discovery

**Resume** spec-shaper with user's answers:

```
Resume agent [ID]:
  prompt: "User answers: [answers]"
```

Agent returns next round of questions (`"complete": false`) or signals done (`"complete": true` with summary).

- If more questions → repeat Step 1.2. **Max 3 rounds total**, then force save.
- If `"complete": true` → resume agent one more time to save requirements:

```
Resume agent [ID]:
  prompt: "Save requirements to SPEC_PATH/planning/requirements.md"
```

### Step 1.4: Phase tracking

Create TodoWrite for high-level progress:

```
TodoWrite:
  1. "Write and verify specification" — pending
  2. "Implement feature" — pending
  3. "Run verification checks" — pending
  4. "Fix issues (if any)" — pending
  5. "Ship: commit, push, create PR" — pending
```

---

## PHASE 2: SPEC

**Goal**: Formal specification + ordered task breakdown. Fully autonomous.

Mark TodoWrite item 1 as in_progress.

### Step 2.1: Write spec

```
Task — subagent_type: "spec-writer"
  model: "sonnet"
  prompt: "Spec folder: SPEC_PATH

  CRITICAL: Return EXACTLY 1 line to stdout, nothing else:
  SPEC_COMPLETE: [path] | sections: N"
```

Expected: `SPEC_COMPLETE: [path] | sections: N`

### Step 2.2 + 2.3: Verify spec AND create task breakdown (PARALLEL)

Launch BOTH in a single message. The task-list-creator reads spec.md independently — it does not need the verifier to finish first.

```
Task A — subagent_type: "spec-verifier"
  model: "sonnet"
  prompt: "Spec folder: SPEC_PATH

  CRITICAL: Return EXACTLY 1 line to stdout, nothing else:
  VERIFY_SPEC: PASSED | sections: N | issues: 0
  or: VERIFY_SPEC: FAILED | issues: [comma-separated list]"

Task B — subagent_type: "task-list-creator"
  model: "sonnet"
  prompt: "Spec folder: SPEC_PATH

  CRITICAL INSTRUCTIONS:
  1. Create an ORDERED list of implementation steps (not parallel waves).
     Order: migrations → models → services/queries → controllers → views → tests.
  2. Each task description must be detailed enough for a single agent to implement it
     without needing to ask questions.
  3. For EACH task, find 1-2 existing files in the codebase that serve as a PATTERN
     to follow. Include these file paths in the task description so the implementing
     agent knows what to read before writing code.

  Save the JSON to SPEC_PATH/tasks.json using the Write tool. Format:
  {
    \"tasks\": [
      {
        \"id\": \"t1\",
        \"subject\": \"Create renter_bank_accounts table\",
        \"description\": \"Migration + model with validations...\"
      }
    ]
  }

  CRITICAL: Return EXACTLY 1 line to stdout, nothing else:
  TASKS: CREATED | path: SPEC_PATH/tasks.json | count: N"
```

**Handle verifier result:**
- `VERIFY_SPEC: PASSED` → continue
- `VERIFY_SPEC: FAILED` → Re-run Step 2.1 → 2.2 only (verifier). Max 2 attempts total. Task breakdown from 2.3 is still valid — task-list-creator reads spec.md which didn't change if verifier passes on retry.

Mark TodoWrite item 1 as completed.

---

## PHASE 3: BUILD

**Goal**: Implement the entire feature with ONE agent. Fully autonomous.

One agent has full context — no cross-agent inconsistencies, no convention mismatches, fewer test runs. Swarm is for research, not implementation.

Mark TodoWrite item 2 as in_progress.

### Step 3.1: Launch feature-builder

```
Task — subagent_type: "feature-builder"
  run_in_background: true
  prompt: "Spec folder: SPEC_PATH"
```

All instructions built into the agent (`.claude/agents/feature-builder.md`).

### Step 3.2: Wait for completion

**IMMEDIATELY** call TaskOutput to wait:

```
TaskOutput — task_id: [agent_task_id], block: true, timeout: 600000
```

10 minute timeout — single agent implementing full feature needs more time.

Parse 1-line status. If BUILD: FAILED, issues will be caught in Phase 4.

Mark TodoWrite item 2 as completed.

---

## PHASE 4: CHECK

**Goal**: Multi-layer verification. Parallel where possible.

Mark TodoWrite item 3 as in_progress.

### Step 4.1: Launch ALL checks in parallel (single message, all background)

All three checks are independent — run them simultaneously.

```
Task A — subagent_type: "quality-gate"
  run_in_background: true
  prompt: "spec-path: SPEC_PATH
  Save review outputs to: SPEC_PATH/verifications/"

Task B — subagent_type: "general-purpose"
  model: "sonnet"
  run_in_background: true
  prompt: "Verify all original requirements from the spec are implemented.

  Steps:
  1. Read SPEC_PATH/planning/requirements.md
  2. Read SPEC_PATH/spec.md
  3. Read SPEC_PATH/planning/architecture-critique.md (if exists)
  4. Run: git diff master...HEAD --stat
  5. For each requirement, verify code changes address it
  6. Save detailed report to SPEC_PATH/verifications/requirements-check.md

  CRITICAL: Return EXACTLY 1 line to stdout, nothing else:
  REQUIREMENTS: PASSED | coverage: N/N
  or: REQUIREMENTS: FAILED | missing: [comma-separated list]"

Task C — subagent_type: "implementation-verifier"
  run_in_background: true
  prompt: "Spec folder: SPEC_PATH
  Verify the implementation matches the spec visually and functionally.
  Return exactly 1 line:
  VERIFICATION: PASSED | ...
  or: VERIFICATION: FAILED | issues: ..."
```

### Step 4.2: Wait for results

**IMMEDIATELY** call TaskOutput for all three agents in a single message (parallel):

```
TaskOutput — task_id: [task_a_id], block: true, timeout: 300000
TaskOutput — task_id: [task_b_id], block: true, timeout: 300000
TaskOutput — task_id: [task_c_id], block: true, timeout: 300000
```

### Step 4.3: Aggregate results

Build status map:
```
quality_gate:  PASSED/FAILED
requirements:  PASSED/FAILED
visual:        PASSED/FAILED
```

- ALL passed → mark TodoWrite item 3 completed → skip Phase 5 → go to Phase 6
- ANY failed → go to Phase 5

---

## PHASE 5: FIX (Auto-retry, max 3 cycles)

Mark TodoWrite item 4 as in_progress.

Initialize: `fix_cycle = 1`

### Step 5.1: Launch parallel fix agents (single message, all background)

One agent per failed check category. Agents fix code but do NOT commit.

For EACH failed check from Phase 4, launch:

```
Task — subagent_type: "general-purpose"
  model: "sonnet"
  run_in_background: true
  prompt: "Fix [CHECK_NAME] issues for this feature.

  Read the report: SPEC_PATH/verifications/[report-file].md
  Also read spec: SPEC_PATH/spec.md

  Instructions:
  1. Read the report, understand ALL issues
  2. Check actual code before fixing — review scripts hallucinate line numbers
  3. Fix each issue
  4. Do NOT run tests, do NOT commit — other agents are fixing in parallel

  CRITICAL: Return EXACTLY 1 line:
  FIX: [CHECK_NAME] | fixed: N issues
  or: FIX: [CHECK_NAME] | fixed: N/M | skipped: [brief list]"
```

Report file mapping:
- quality-gate → `review-summary.md`
- requirements → `requirements-check.md`
- codex → `codex-review.md`
- security → `security-review.md`
- performance → `performance-review.md`
- quality → `quality-review.md`
- multitenancy → `multitenancy-review.md`
- visual → `final-verification.md`

### Step 5.2: Wait + finalize

**IMMEDIATELY** call TaskOutput for ALL fix agents in parallel.

After all agents complete:

```bash
bundle exec rubocop -a
bin/rails test
git add -A && git commit -m 'fix: address verification findings'
```

Run rubocop/tests/commit via a single general-purpose agent (sonnet, background) to keep orchestrator clean.

### Step 5.3: Re-verify (only failed checks)

Re-run ONLY the checks that failed in Phase 4 (same subagent patterns as Phase 4).

### Step 5.4: Evaluate

- All checks pass → mark TodoWrite item 4 completed → go to Phase 6
- Still failing AND `fix_cycle < 3` → increment `fix_cycle` → go to Phase 5 again
- `fix_cycle >= 3` → mark TodoWrite item 4 completed (with note) → go to Phase 6 with warnings

Store: `remaining_issues = [list of still-failing checks]`

---

## PHASE 6: SHIP

Mark TodoWrite item 5 as in_progress.

### Step 6.1: Final commit

```bash
git add -A && git status --short
```

If uncommitted changes exist:

```bash
git commit -m "$(cat <<'EOF'
feat: [feature name derived from spec]

[1-2 line summary of what was implemented]
EOF
)"
```

### Step 6.2: Push

```bash
git push -u origin HEAD
```

On network error: retry up to 4x with exponential backoff (2s, 4s, 8s, 16s).

### Step 6.3: Create PR

```bash
gh pr create --title "feat: [short feature name, max 70 chars]" --body "$(cat <<'EOF'
## Summary

[2-3 bullet points from spec — what this feature does for the user]

## Verification

| Check | Result |
|-------|--------|
| Tests + Linting | ✅/❌ |
| Security (Brakeman) | ✅/❌ |
| Code Review | ✅/❌ |
| Requirements Coverage | ✅/❌ (N/N) |
| Visual Verification | ✅/❌ |

Fix cycles used: N/3
[If remaining_issues: "⚠️ Known issues: [list]"]

## Test Plan

[Steps for manual QA verification — from spec]

---
Spec: SPEC_PATH/spec.md
EOF
)"
```

### Step 6.4: Summary

Mark TodoWrite item 5 as completed.

Output:

```
Feature shipped!

Spec:    SPEC_PATH
Branch:  BRANCH_NAME
PR:      [PR URL]

Verification:
  Quality Gate:  ✅/❌
  Requirements:  ✅/❌ (N/N)
  Visual:        ✅/❌
  Fix cycles:    N/3
```

### Step 6.5: Learning Summary

After shipping, reflect on the implementation and share insights with the user. Write in natural language — this is for the developer, not a machine. Be specific to THIS feature, not generic.

#### Always include:

```markdown
---

## Learning Summary

#### Key Insight
What's the **one thing** worth remembering from this implementation?
Frame as a reusable principle or mental model.
- "When X happens, always consider Y because Z"
- "This is a classic example of [pattern name]"

#### Architecture Decisions
What non-obvious decisions were made and why?
- Why this approach over alternatives
- What tradeoffs were considered
- What constraints shaped the design

#### Connections
How does this connect to broader SW engineering?
- Design patterns or principles applied
- Similar challenges you'd encounter in other domains
```

#### Only if genuinely valuable:

```markdown
#### Further Learning
One specific resource (article, docs section, book chapter) that deepens
understanding of a key concept from this implementation.
```

---

## Error Handling

| Error | Action |
|-------|--------|
| Subagent timeout | Retry once. If still fails, skip and note in PR |
| Git conflict | Report to user. Do NOT force resolve |
| Network error on push | Retry 4x with exponential backoff (2s, 4s, 8s, 16s) |
| All 3 fix cycles exhausted | Create PR with ⚠️ warning and list remaining issues |
| Spec verification fails 2x | Proceed with best-effort spec, note in PR |
| JSON parse error from subagent | Ask subagent to retry with stricter format instructions |

## Resume

Running `/ship` without arguments:
1. spec-initializer finds recent spec folders and detects resume point (Phase 0)
2. Returns `INIT: EXISTING` or `INIT: RESUME` with the right phase
3. Orchestrator skips to indicated phase

No user confirmation needed for resume — it picks up where it left off.
