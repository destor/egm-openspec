---
name: roles
license: Apache-2.0
description: Internal role prompts for the subagents that sdd-orchestrate and sdd-quick spawn — coder, reviewer, security auditor, style auditor. Not run directly; the orchestrating skill tells each subagent which role file to read.
---

# Roles

Each file is the full prompt for one subagent role. The orchestrating skill (`sdd-orchestrate`
or `sdd-quick`) spawns a general-purpose subagent, gives it the absolute path of its role
file, and owns when the role runs, its round limits, and how its verdict is used.

| Role | File | Writes code |
| --- | --- | --- |
| Coder | [`coder.md`](coder.md) | yes |
| Reviewer | [`reviewer.md`](reviewer.md) | no |
| Security auditor | [`security-auditor.md`](security-auditor.md) | no |
| Style auditor | [`style-auditor.md`](style-auditor.md) | no |
