---
name: sdd-review
description: Goldfish review gate for an OpenSpec change. Spec mode reviews proposal.md, the spec deltas and design.md in four fresh-context stages (Zoom Out, Comprehension, Critic, Intermediate Zoom Out) before tasks are written; tasks mode runs the Implementation Readiness review on tasks.md and task-details/ before implementation. Findings are resolved with the author. Use after sdd-create-spec or sdd-spec-to-tasks, or when the user asks to review a change.
---

# SDD Review: Goldfish Review Gates

Reviews and validates an OpenSpec change through specialized reviewer personas that read
the artifacts without the conversation that produced them. Two modes, matching the two
halves of a change:

| Mode | Target | Stages | Runs after |
| --- | --- | --- | --- |
| `spec` | `proposal.md`, `specs/**/spec.md`, `design.md` | 5 → 6 → 7 → 8 | `sdd-create-spec` |
| `tasks` | `tasks.md`, `task-details/*.md` | 9 | `sdd-spec-to-tasks` |

Reviewer personas live in [`reviewers/`](reviewers/), standards in
[`standards/`](standards/). Pass every path to reviewers as an absolute path.

## Delegation

A Goldfish runs in a fresh context with no conversation history. Spawn each
reviewer with your harness's subagent tool (GitHub Copilot: its native subagent
tool, `agent` / `runSubagent` in VS Code — inspect its exposed schema; Claude
Code: `Agent`; Codex: `spawn_agent`; Antigravity: `invoke_subagent`). Spawn a
general-purpose subagent and tell it to read the reviewer file first. Pass the
prompt verbatim and wait for the report. If your harness has no subagent tool,
stop and tell the user: a review in the same context is not a Goldfish review.

- Start a new child for every reviewer invocation, including repeat reviews. Never fork or resume this conversation.
- Use a task that contains only the requested mode, target, persona, standards, and explicit reading list. Do not forward the conversation, private rationale, memory, or answers the target should explain. Never pass `brainstorm.md`.
- Read prior reviewer reports only when explicitly supplied in the reading list and required by the stage (Stage 8). Do not forward other previous findings.
- Wait for each completion before synthesis. A missing child, read failure, timeout, or denied tool makes that reviewer incomplete. Never simulate a child or treat missing findings as a passing result.
- Fresh conversation context does not mean that the host omits its instruction files. State review assumptions that the Goldfish must challenge explicitly in the artifacts.

Task for each reviewer:

```text
Read <reviewer file> before work and follow it.
Mode: REVIEW
Target: <absolute paths of the stage's target files>
Target Standard: <absolute standard path, or "none">
Reading list: <GoldfishContext paths, plus any report the stage requires>
Repository root: <absolute path>; search source files there when your instructions call for code search.
Use only this task and its explicit reading list. Do not read other files in the change directory.
Do not edit files, run commands, delegate, or use MCP tools.
Return findings and the paths you read.
```

## Execution Constraints (CRITICAL)

1.  **Artifact-First Delivery (MANDATORY)**: You are **STRICTLY FORBIDDEN** from
    presenting large reports or summaries directly in the chat pane. You **MUST**
    write them to the review directory using your file-writing tool and then
    tell the user in chat the file path and ask for review.
2.  **Plain-English Standard**: You **MUST** write all reports, summaries, and
    communications in simple, direct, jargon-free plain English. Explain
    technical mechanisms directly. Do not summarize away substance or rely on
    buzzwords.
3.  **No Glazing**: Eliminate sycophancy. Do not call ideas "great,"
    "brilliant," or "smart." Compliments without substance are noise.
4.  **Banned Apologies**: You MUST NOT output polite apologies (e.g., "I
    apologize," "I am sorry," "Forgive me") when challenged or corrected.
    Instead, outline the corrected technical logic or trade-offs directly.
5.  **Interaction Invariants**: All interactive gates (structured question tools,
    if your harness has one, or plain chat) MUST contain strictly one question,
    be the sole tool call in the turn, and provide clear, action-oriented
    options without `(Recommended)` prefixes.
6.  **Preserve unauthored lines**: when resolving findings, keep lines and sections
    of the artifacts that the review did not target (for example tracking sections
    or marker lines added by other tools).

## Workflow

### Step 1: Intake & Setup

1.  **Change and mode**: Resolve the change id (from the caller, the conversation, or
    `openspec list --json`). Use the mode the caller gives; otherwise pick `spec` when
    `tasks.md` does not exist yet and `tasks` when it does, and confirm with the user.
2.  **Target files**: Run `openspec status --change "<change-id>" --json` and take the
    target files for the mode from `artifactPaths.<id>.existingOutputPaths`. For
    `tasks` mode, add every file in `task-details/`. Stop if a target is missing.
3.  **GoldfishContext**: Ask once, in plain chat, whether the user wants to add context
    files (a reading list that establishes baseline knowledge for reviewers). An empty
    answer counts as none. Add linked files from the targets whose real path stays
    inside the repository; skip `brainstorm.md`, anchor-only links and files that do
    not exist. List links outside the repository or URLs in chat and ask which to
    include. Some reviewers cannot open URLs: save the raw page of each approved URL as
    a file in the review directory and pass its path instead.
4.  **Review directory**: Write all reports to `<change root>/review/`. OpenSpec ignores
    this folder; it moves with the change when the change is archived.

### Step 2: Run the stages

Run the stages of the mode in order. Parallelize only reviewers within the current
stage. Wait for every reviewer in that stage, write its report, and finish its gate
before dispatching the next stage.

**Failed Reviewers**: A reviewer that times out, fails, or returns no report did not
review. Retry it once. If it fails again, its status is `not run`. Never count it as a
reviewer with zero findings.

#### Stage 5: Initial Zoom Out Review (`spec`)

- Reviewers, in parallel: `reviewers/zoom_out.md` with `standards/zoom-out-standard.md`, and `reviewers/inventive_catalyst.md` with no standard. Pass `GoldfishContext`.
- Write both into `zoom_out_report.md` (report format below).
- Gate: Stages 5, 6, 8 checkpoint.

#### Stage 6: Comprehension Test (`spec`)

- Reviewer: `reviewers/comprehension.md`, no standard, no `GoldfishContext`: it reads the target alone. The test is self-containment, not detail.
- Write the complete reviewer result to `comprehension_report.md`. Start with the two paraphrase headings verbatim, then the findings. Keep the paraphrase when there are no findings. The author compares the paraphrase with the intent; a wrong paraphrase is a finding.
- Gate: Stages 5, 6, 8 checkpoint.

#### Stage 7: Critic Review (`spec`)

- Reviewers, in parallel: `reviewers/design_critic.md` and `reviewers/structural.md` with `standards/feature-spec-standard.md`, and `reviewers/style.md` with no standard. Pass `GoldfishContext`.
- Write `critic_report.md`.
- Gate: Stages 7, 9 checkpoint.

#### Stage 8: Intermediate Zoom Out (`spec`)

- Reviewer: `reviewers/zoom_out.md` with `standards/zoom-out-standard.md`. Pass `GoldfishContext` and add `zoom_out_report.md` to the reading list; tell it to focus on additions and mitigations made since Stage 5.
- Write `intermediate_zoom_out_report.md`.
- Gate: Stages 5, 6, 8 checkpoint; Option 0 is `Conclude review and begin detailed implementation planning`.

#### Stage 9: Implementation Readiness Test (`tasks`)

- Reviewers, in parallel: `reviewers/implementation_readiness.md` and `reviewers/codebase_details.md` with `standards/implementation-readiness-standard.md`, and `reviewers/style.md` with no standard. Pass `GoldfishContext`.
- Write `implementation_readiness_report.md`.
- Gate: Stages 7, 9 checkpoint; Option 0 is `Conclude review and finalize tasks`.

#### Report format

Group findings into 🔴 **Blocker**, 🟡 **Major**, and 🟢 **Nit**. Deduplicate overlapping
findings and normalize file references.

```markdown
# Goldfish Review Report

**Target:** `[target files]`

## Reviewer Status

| Reviewer | Status |
| --- | --- |
| `[reviewer file]` | [completed \| not run] |

## Prioritized Findings (Blockers -> Major -> Nits)

### [Issue Name] - Severity: [Blocker | Major | Nit]
* **Reviewer Lens:** `[reviewer file]`
* **Affected Scope:** `[file:lines]` (or section header)
* **Discrepancy Description**: [Precise technical, structural, or clarity gap.]
* **Rationale & Recommendation**: [Concise, objective reason and guidance.]
```

### Step 3: Checkpoint Gates

Notify the user that findings are written to the report (give its path), then present
one single-select gate:

- **Stages 5, 6, 8**: Question: "Stage [N] ([Name]) completed. Findings written to [report]. How would you like to proceed?"
  - Option 0: `Proceed to next stage: Stage [N+1] ([Next Stage Name])` (Stage 8: `Conclude review and begin detailed implementation planning`)
  - Option 1: `I will update the change first (pause session; notify me in chat when edits are complete to re-run)`
  - Option 2: `Pause Review (save progress and exit)`
- **Stages 7, 9**: Question: "Stage [N] ([Name]) completed. Findings written to [report]. How would you like to proceed?"
  - Option 0: `Proceed to next stage: Stage [N+1] ([Next Stage Name])` (Stage 9: `Conclude review and finalize tasks`)
  - Option 1: `Resolve findings with agent (evaluate findings, propose fixes, and apply approved edits)`
  - Option 2: `I will update the change manually (pause session; notify me in chat when edits are complete to re-run)`
  - Option 3: `Pause Review (save progress and exit)`

**Pause & resume**: On a manual-update option, confirm in chat that the review is paused
and yield. When the author says edits are complete, re-read the targets and re-run the
current stage. **1-Audit Convergence Cap**: each stage allows a maximum of 1 re-audit. If
updates are requested again after a re-audit, tell the author the cap is reached, record
residual findings as accepted trade-offs, and advance to the next stage.

**Pause Review**: halt immediately, keeping all reports.

### Step 4: Resolve Findings (Option 1 of Stages 7 and 9)

Run in this conversation — the Elephant — which holds the reasoning behind the change.

1.  **Evaluate & Categorize**: Give each finding exactly one category:
    *   **CONTEST**: The critique is inaccurate or relies on a misunderstanding of
        the codebase. Cite evidence that the reviewer could read: a file and
        symbol, or a decision recorded in the artifacts.
    *   **ESCALATE**: The critique is inaccurate, but the evidence is outside every
        file the reviewer could read (for example, a team convention or a verbal
        decision). State that fact. Do not fix the finding. Give it to the user.
    *   **PROPOSE FIX**: The critique is valid. Define precise textual replacements
        or structural formatting adjustments to resolve the gap.
    *   **DECLINE**: The finding is a Nit. Record it. Do not fix it. You can use
        PROPOSE FIX for a Nit if the fix is worth it.

    Apply these rules:

    *   **A contest must be checkable.** "This was intentional" is not evidence.
        If you cannot cite evidence the reviewer could read, use ESCALATE or
        PROPOSE FIX.
    *   **Do not add context only to satisfy a reviewer.** Add context when a human
        reader of the artifacts needs it. If only a zero-context subagent needs it,
        use ESCALATE.
    *   **Do not contest a comprehension gap.** If a fresh reader could not follow
        the artifacts, they are missing context.
    *   **Do not change a severity.** If you think a Blocker is a Nit, CONTEST it
        and state the reason.
    *   You wrote the artifacts under review. A wish to contest is not evidence
        that the reviewer is wrong.

2.  **Report**: Save `goldfish_review_report.md` in the review directory:

    ```markdown
    # Goldfish Annotated Review Report

    **Target:** `[target files]` — Stage [N]

    ## Prioritized Findings & AI Suggestions

    ### [Issue 1] [`[reviewer file]`] - Severity: [Blocker | Major | Nit]
    *   **Affected Scope:** [file:lines] ([Sub-header])
    *   **Discrepancy Description:** [copied from the stage report]
    *   **Rationale:** [copied from the stage report]
    *   ***AI Suggestion:*** [CONTEST | ESCALATE | PROPOSE FIX | DECLINE]
    *   ***Justification:*** [Detailed reason]
    *   ***Proposed Edit:***
        *   Location: [file, line/header]
        *   Change: [Precise replacement string or deletion note]
    ```

3.  **Resolution Gate**: List ESCALATE findings first in chat — they need the user's
    knowledge, not only approval. Then tell the user the report path and the choices,
    and wait without calling tools:
    -   **Comment directly on the file**: leave comments/edits in
        `goldfish_review_report.md` to approve, reject, or contest specific items.
    -   **Auto accept**: apply all PROPOSE FIX edits in one batch; CONTEST,
        ESCALATE and DECLINE items are skipped.
    -   **Continue**: proceed without applying fixes.
    -   **Interactive breakdown**: walk through every item except DECLINE, one per
        turn. PROPOSE FIX → **Accept proposed fix**, **Skip / ignore**, or
        **Discuss**. CONTEST → **Skip / agree with contest**, **Overrule contest
        (propose fix)**, or **Discuss**. ESCALATE → **Confirm (no change)**, **Add
        the fact to the artifacts**, or **Discuss**. Record decisions; edit nothing
        until all items are decided, then apply the accepted edits.
    -   **Custom instruction**: apply it and proceed.

4.  **Post-Resolution**: Apply edits to `proposal.md`, spec deltas and `design.md` (spec
    mode) or `tasks.md` and `task-details/` (tasks mode); the resolution gate is the
    approval. Run `openspec validate <change-id> --strict`. Add a `Status: resolved`
    line at the top of the report. Then advance to the next stage, or re-run the
    stage (counts toward the 1-audit cap).

## Handoff

- `spec` mode concluded → offer [`sdd-spec-to-tasks`](../sdd-spec-to-tasks/SKILL.md). Invoked from `sdd-spec-to-tasks` intake → resume it instead.
- `tasks` mode concluded → offer [`sdd-orchestrate`](../sdd-orchestrate/SKILL.md). Invoked from `sdd-orchestrate` → resume it instead.
- Do not hand off automatically. Name any open Blocker or Major finding and any reviewer recorded as `not run` before the user decides.
- Also offer any follow-up steps the project's agent instructions define for this stage.
