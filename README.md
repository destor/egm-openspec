# Spec-driven development skills

Agent skills for spec-driven development with [OpenSpec](https://openspec.dev/): an idea is
brainstormed, specified, reviewed by fresh-context reviewers, broken into tasks, and
implemented task by task through a coder → reviewer → gate cycle, with the developer
approving each step. Small tasks and bug fixes have a shorter flow.

The skills live in [`.agents/skills/`](.agents/skills/). They are written for any agent
harness that loads skills and can start subagents; GitHub Copilot is the primary target.

## Contents

- [Setup](#setup)
- [Skills](#skills)
- [Feature flow](#feature-flow)
- [Small task and bug flow](#small-task-and-bug-flow)
- [Testing](#testing)
- [Keeping specs and code in step](#keeping-specs-and-code-in-step)
- [Maintaining the skillset](#maintaining-the-skillset)
- [Credits](#credits)
- [License](#license)

## Setup

1. **OpenSpec CLI** (global):

   ```sh
   npm install -g @fission-ai/openspec@latest
   openspec --version    # 1.17.0 or later
   ```

2. **OpenSpec in the project:** the project needs an `openspec/` root (`openspec init` in the
   project, or an existing setup). The skills never create it on their own.
3. **Skills:** copy `.agents/skills/` into the project. GitHub Copilot discovers skills in
   `.agents/skills/`.
4. **Subagents:** the review, orchestration and quick flows start subagents. In a harness
   without a subagent tool they stop and say so; they never fake a review in the same
   context.
5. **Project instructions** (`AGENTS.md`, Copilot instructions or equivalent), as needed:

   ```text
   Seam registry: docs/testing/seams/index.md
   ```

   The seam registry is recommended for large repositories (see [Testing](#testing)).

## Skills

Skills prefixed `sdd-` are the ones a developer runs. The others are run by those skills, or
are OpenSpec mechanics.

| Skill | Run by | Purpose |
| --- | --- | --- |
| `sdd-brainstorm` | developer | Stress-test an idea, write the OpenSpec proposal, review it with fresh-context reviewers |
| `sdd-create-spec` | developer | Write the spec deltas and design, one approved artifact at a time |
| `sdd-review` | developer | Fresh-context review gate: `spec` mode before tasks, `tasks` mode before implementation |
| `sdd-spec-to-tasks` | developer | Break the approved spec into phased tasks with per-task definitions and test plans |
| `sdd-orchestrate` | developer | Implement the change task by task, verify, run the manual checks gate, archive |
| `sdd-quick` | developer | Small task or bug fix through the same cycle, without an OpenSpec change |
| `sdd-drift-report` | developer | Compare `openspec/specs/` (or other docs) with the current code |
| `tdd` | `sdd-` skills | Test rules: seams, good tests, anti-patterns, red → green |
| `roles` | `sdd-orchestrate`, `sdd-quick` | Prompts for the coder, reviewer, security auditor and style auditor subagents |
| `openspec-*` | `sdd-` skills | OpenSpec mechanics: new, continue, update, verify, sync specs, archive |

## Feature flow

```text
 idea
        │
        ▼
 sdd-brainstorm ──► proposal.md + brainstorm.md      (Goldfish review of the proposal)
        │
        ▼
 sdd-create-spec ──► specs/**/spec.md + design.md    (one approved artifact at a time)
        │
        ▼
 sdd-review spec ──► review/*.md                     (offered by default; 4 stages)
        │
        ▼
 sdd-spec-to-tasks ──► tasks.md + task-details/      (phases, files, test plans)
        │
        ▼
 sdd-review tasks ──► review/implementation_readiness_report.md   (offered by default)
        │
        ▼
 sdd-orchestrate ──► code, tests, commits, progress.md
        │            (per task: coder → reviewer → gates; then verify,
        │             manual checks gate, archive)
        ▼
 archived change, main specs updated
```

Each step ends with a gate: the skill shows what it wrote and waits for the developer.
Nothing moves to the next step automatically. After `sdd-create-spec` and after
`sdd-spec-to-tasks`, the first option of the gate is the matching review; skipping it is a
separate choice. If a review was skipped, `sdd-spec-to-tasks` and `sdd-orchestrate` offer
it again before they start.

### Elephant and Goldfish

The flow uses the Elephant–Goldfish model. The **Elephant** is the main conversation: it
holds the project context and the decisions made with the developer, and writes the
artifacts. A **Goldfish** is a subagent started in a fresh context with only the document
under review and the files it is told to read — never the conversation. It sees the
artifact the way a new team member would, so it finds assumptions the Elephant shares with
the developer. The **developer** resolves the trade-offs and approves each step.

A fresh context hides the conversation, not necessarily the project's instruction files.
Assumptions that reviewers must challenge belong in the artifacts themselves.

### 1. Brainstorm — `sdd-brainstorm`

- Asks open-ended questions, one per turn, about the problem, consumers, constraints, edge
  cases, current workaround and success measure. It stress-tests rather than agrees.
- At the readiness gate it summarises the idea in 2–3 sentences and proposes a change id.
- Creates the OpenSpec change and writes `proposal.md`.
- Runs Goldfish reviewers on the proposal in parallel: Contrarian/YAGNI, UX & developer
  ergonomics, migration & compatibility, tech debt, and inventive catalyst. Cost and
  observability reviewers run only when the developer accepts them; the skill suggests them
  when the proposal raises their questions.
- Writes findings, non-goals, rejected alternatives and key decisions to `brainstorm.md`,
  points out the 2–3 strongest objections, and iterates until the developer is satisfied.
- Hands off to `sdd-create-spec`, or keeps the change for later.

### 2. Specify — `sdd-create-spec`

- Synthesises the scope from `proposal.md` and `brainstorm.md`, and asks only what remains
  open.
- Researches the codebase: existing capabilities, reuse candidates (reuse as-is, extract then
  share, new), test seams, and the manual checks the developer will run.
- For every decision with more than one defensible answer, presents up to three unranked
  options and lets the developer choose.
- Writes the spec deltas and `design.md`, one artifact at a time, each followed by an
  approve/revise gate. Spec scenarios use GIVEN/WHEN/THEN and are the source of truth for
  expected values in tests. `design.md` records Reuse, Testing (seams), Manual checks,
  alternatives, risks and open questions.
- Cross-checks the spec against `brainstorm.md` and lists every contradiction for the
  developer to decide.

### 3. Spec review — `sdd-review` (spec mode)

Four fresh-context stages, each ending with a gate:

| Stage | Reviewers | Checks |
| --- | --- | --- |
| 5 Zoom Out | zoom out, inventive catalyst | root problem, simplicity, reuse, false compromises |
| 6 Comprehension | comprehension | can a fresh reader explain the spec back? |
| 7 Critic | design critic, structural, style | feature-spec standard: alternatives, risks, structure, prose |
| 8 Zoom Out again | zoom out | what stages 6–7 added |

At stages 7 and 9 the developer can have the agent resolve findings: each finding is
marked fix, contest (with evidence the reviewer could read), escalate (needs the
developer's knowledge), or decline, and the developer approves the edits. Each stage allows
one re-review. Reports go to `review/` in the change.

### 4. Tasks — `sdd-spec-to-tasks`

- Splits the work into phases; each phase leaves the code buildable and testable and has a
  checkpoint line.
- `tasks.md` holds one checkbox per task (short description, main file, link). The full
  definition of each task is in `task-details/<N.M>-<slug>.md`: blocking tasks, spec
  scenarios, design decisions, `#security` flag, every file to create/modify/delete with
  rationale, and the test plan (Verify via, Preconditions, Test Cases, Not tested).
- No design decision is left to the implementer; a missing one goes back to the developer.
- Ends with an approve/revise gate and a Verification Plan (build, tests, lint, manual
  checks).

### 5. Tasks review — `sdd-review` (tasks mode)

Stage 9, Implementation Readiness: implementation-readiness, codebase-details and style
reviewers check that a fresh coder could execute every task without guessing.

### 6. Implement — `sdd-orchestrate`

The orchestrator never writes code. Per task, once the tasks blocking it are checked:

1. **Security advisory** before the first `#security` task of a phase.
2. **Coder** — a fresh subagent with only the task definition and what it cites. Writes
   tests red → green at the task's seam, changes only the listed files, and stops with a
   plan deviation when the plan does not work.
3. **Reviewer** — correctness, spec conformance, test plan conformance, write boundary,
   tautological or unplanned tests. One fix round and one follow-up review; open blocking
   findings go to the developer.
4. **Checkbox** ticked with a single-line edit.

Per phase: a security gate when it has `#security` tasks, the checkpoint command, and a
commit. Per change: style gate, `openspec validate --strict`, `openspec-verify-change`, the
developer's manual checks, then archive (main specs updated, change moved to the archive).

`tasks.md` checkboxes are the only task state, so a run can resume. `progress.md` in the
change is an append-only journal of the run.

## Small task and bug flow

`sdd-quick` runs one small task or bug fix through the same cycle without an OpenSpec
change. State lives in the conversation and in git.

1. **Clarify** — Bug: observed, expected, reproduction, impact. Task: outcome, location,
   definition of done.
2. **Size check** — redirects to the feature flow if the work has 5+ sub-tasks, crosses
   services architecturally, needs a schema migration, makes architectural or security-model
   decisions, needs stakeholder alignment, or **changes behaviour described in
   `openspec/specs/`**. In doubt, it redirects.
3. **Plan** in the conversation, approved by the developer: files, seam, test cases with the
   source of each expected value, not tested, `#security`, one manual check. In Bug mode the
   cause is stated first, and the first test reproduces the bug — its failure must show the
   observed behaviour.
4. **Execute** with the same roles: coder, reviewer, security advisory and gate if
   `#security`, style gate.
5. **Finish** — tests and build, the developer's manual check, a conventional commit on
   approval, and a report. The report notes when the fixed behaviour is not described in any
   spec.

## Testing

The internal `tdd` skill sets the test rules for every flow:

- **Seams.** Tests run only at pre-agreed seams — public boundaries where behaviour is
  observable. `sdd-create-spec` agrees them with the developer, each with a note on what it
  catches and misses; tasks name their seam in `Verify via`.
- **Seam registry.** In a large repository, seams are recorded per area: an index maps paths
  to area files that name the seams and the boundaries tests may fake. The project's
  instructions name the registry (`Seam registry: <path>`).
- **Good tests** verify behaviour through public interfaces and survive refactors.
- **Anti-patterns:** implementation-coupled tests; tautological tests (an expected value
  recomputed the way the code computes it, or a constant asserted equal to itself);
  horizontal slicing (all tests first, then all code).
- **Red → green**, one slice at a time; refactoring belongs to review.
- **Against test bloat:** every test case names the source of its expected value (a spec
  scenario, a known literal, a worked example); `Not tested` excludes shape, what tooling
  already proves, and values the code itself declares; reviewers delete unplanned and
  tautological tests.

## Keeping specs and code in step

- **Revising a change:** `openspec-update-change` revises existing artifacts and keeps them
  consistent; it never edits code.
- **Before archive:** `sdd-orchestrate` runs `openspec-verify-change`, which compares the
  implementation with the change.
- **Any time:** `sdd-drift-report` compares the main specs in `openspec/specs/` (or another
  document folder) with the current code and reports only the differences. The developer
  decides whether the spec or the code is wrong; a spec change goes through a new change.

## Maintaining the skillset

### Updating the OpenSpec skills

The `openspec-*` skills are generated by the OpenSpec CLI and copied unchanged. Regenerate
them in a scratch folder with a throwaway config, so the global OpenSpec profile is not
touched, and without the `apply` workflow (implementation goes through `sdd-orchestrate`):

```sh
export XDG_CONFIG_HOME="$(mktemp -d)"
openspec config set profile custom
openspec config set workflows '["new","continue","update","verify","sync","archive"]'
cd "$(mktemp -d)" && openspec init --tools codex .
cp -R .agents/skills/openspec-* <this-repo>/.agents/skills/
```

The `codex` target writes to `.agents/skills/` with tool-neutral wording. Running
`openspec init` or `openspec update` with the default profile directly in a project adds
`openspec-apply-change`, `openspec-propose` and `openspec-explore` back.

### Conventions

- `sdd-` prefix: skills the developer runs. Internal helpers (`tdd`, `roles`) have no prefix.
- Text taken from upstream projects is copied verbatim where possible, with a notice line
  naming the source and the changes, and the upstream licence next to it.

### Verified so far

GitHub Copilot discovers the skills in `.agents/skills/` and starts `sdd-brainstorm`'s
reviewer subagents. OpenSpec 1.17 accepts the extra files and sections these skills add
(`brainstorm.md`, `review/`, `task-details/`, `progress.md` and checkpoint lines). The other
flows have not yet been run end to end.

## Credits

- [Elephant-Goldfish Model](https://github.com/google/elephant-goldfish-model) (Apache-2.0)
  — the brainstorm, drafting and review workflows, reviewer personas, document standards
  and drift report. Licence copies are in `sdd-brainstorm/lenses/`, `sdd-review/` and
  `sdd-drift-report/`.
- [Matt Pocock's skills](https://github.com/mattpocock/skills) (MIT) — the `tdd` skill and
  its test and mocking guides. Licence copy in `tdd/`.
- [OpenSpec](https://github.com/Fission-AI/OpenSpec) (MIT) — the `openspec-*` skills,
  generated by its CLI. Licence copy in each `openspec-*` folder.
- Birgitta Böckeler, [TDD in the agent loop](https://martinfowler.com/articles/exploring-gen-ai/tdd-in-the-agent-loop.html)
  — the reasoning behind the anti-bloat rules and approved scenarios as the source of truth.

## License

[Apache License 2.0](LICENSE). Files taken from other projects keep their own licence, in the
`LICENSE` file next to them (see [Credits](#credits)). Not affiliated with Google or the
Elephant-Goldfish Model project.
