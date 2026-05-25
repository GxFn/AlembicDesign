# AlembicDesign Docs

更新日期：2026-05-25

This index maps the requirement-design window. Current design drafts live under `docs/current/`. Stable operating rules live directly under `docs/`. Reusable formats live under `templates/`.

## Current Map

| Type | Document | Status | Purpose |
| --- | --- | --- | --- |
| Operating policy | [design-window-operating-policy.md](design-window-operating-policy.md) | Active | Defines how AlembicDesign collaborates with AlembicWorkspace. |
| Current drafts | [current/README.md](current/README.md) | Empty | Holds active original plans, design drafts, and handoff drafts. |
| Original plan template | [../templates/original-plan-template.md](../templates/original-plan-template.md) | Active | First pass for user confirmation. |
| Requirement design template | [../templates/requirement-design-template.md](../templates/requirement-design-template.md) | Active | Detailed design after original plan confirmation. |
| Workspace handoff template | [../templates/workspace-handoff-template.md](../templates/workspace-handoff-template.md) | Active | Handoff draft for AlembicWorkspace. |

## File Placement

- `docs/current/<topic>-original-plan-YYYY-MM-DD.md`: active original plan drafts.
- `docs/current/<topic>-requirement-design-YYYY-MM-DD.md`: active detailed design drafts.
- `docs/current/<topic>-workspace-handoff-YYYY-MM-DD.md`: handoff drafts for AlembicWorkspace.
- `docs/<topic>-policy.md` or `docs/<topic>-contract.md`: stable design-window rules only.

Use lower-case kebab-case names and execution date `YYYY-MM-DD`.
