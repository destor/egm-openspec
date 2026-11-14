---
name: sdd-drift-report
license: Apache-2.0
description: >-
  Runs drift detection on the OpenSpec main specs (openspec/specs/) or another
  directory of design documents to check for document and code consistency.
  Produces an actionable report with differences. Use when the user asks to
  detect drift or check whether the specs still match the code.
---

<!-- Copied from github.com/google/elephant-goldfish-model skills/egm-drift-report/SKILL.md (Apache-2.0, see LICENSE). Changed: defaults to OpenSpec main specs; harness notes removed; handoff added. -->

# Detect Drift In Specs and Code

When the user asks to detect drift across all designs or a set of READMEs, the
goal is to identify all instances where the codebase has diverged or failed to
implement the specifications described by those designs or READMEs.

`openspec-verify-change` checks one active change before it is archived. This
skill checks the accumulated main specs — or any other document directory —
against the current code at any time.

1.  **Check Input**: Use the directory the user gives. If none is given, use the
    main specs at `openspec/specs/` (one `<capability>/spec.md` per capability)
    and confirm with the user. If neither exists, ask the user to specify the
    target directory.
2.  **List Documents**: List all files in the design directory.
3.  **Analyze**: For EACH design document:

    *   **Contextualize**: Read the design document. Identify the intent,
        proposed changes, and specific filenames/symbols mentioned. In an
        OpenSpec spec, every `### Requirement:` and `#### Scenario:` states
        behavior the code must have.
    *   **Locate Source**: Find the corresponding source code files in the
        source tree. If paths aren't explicit, use code search or grep to
        locate the relevant classes, functions, or API endpoints mentioned in
        the doc. Prompt the user to add missing links to the document to make
        future detection cheaper.
    *   **Verify**: Read the current state of those source files.
    *   **Detect Drift**: Compare the design vs. reality. Look specifically for:
        *   Missing Features: Logic described in the doc but absent in the code.
        *   Incomplete Models: Dataclasses/Classes/Structs/Protos missing fields
            specified in the design.
        *   Logic Shortcuts: Hardcoded values where dynamic logic was planned.
        *   Architectural Divergence: Implementation using different
            patterns/libraries than proposed.
    *   **Filter**: Report ONLY the differences. Do not describe what is working
        or what matches. If the code matches the design perfectly, output "No
        drift detected."

4.  **Report**: Produce a single report listing all discrepancies found.

    *   Append the analysis for each file to `{YYYY-MM-DD}_DRIFT_REPORT.md`
        using your file-writing tool.
    *   Strict Format:

        ```markdown
        ## <DESIGN_DOC_NAME>

        <Bulleted list of specific divergences found>
        <Or "No drift detected">
        ```

5.  **Guidance**

    -   Be rigorous. If a design doc mentions a specific object, field or file
        name and the code doesn't have it, that is a reportable drift. Double
        check for coverage. If there is some part of the logic that is not
        referenced in a design or README, this is a reportable drift.
    -   Read files liberally to ensure you are looking at the current state
        of the code, not relying on memory.
    -   Process the files sequentially to ensure the tool output buffer doesn't
        overflow.

6.  **Handoff**: Give the report path. The user decides, per divergence, whether
    to update the document or the code. Do not edit either. A spec that should
    change goes through a new OpenSpec change
    ([`sdd-brainstorm`](../sdd-brainstorm/SKILL.md) or
    [`sdd-create-spec`](../sdd-create-spec/SKILL.md)), not a direct edit of
    `openspec/specs/`.
