# AlembicDesign

AlembicDesign is the requirement-design window for the Alembic workspace. It helps the user and the AlembicWorkspace controller discuss product intent, shape requirement plans, compare design options, and prepare handoff artifacts before implementation waves are dispatched.

This repository does not implement Alembic product code. It is controlled by the AlembicWorkspace total-control window.

## Responsibilities

- Discuss new requirements with the user and make the goal concrete.
- Produce original plans, requirement designs, design notes, option comparisons, and handoff drafts.
- Identify missing facts, decision points, risks, non-goals, and validation needs.
- Request or summarize code research, but do not dispatch implementation windows directly.
- Handoff confirmed design artifacts back to AlembicWorkspace for stage confirmation, wave dispatch, verification, and archive.

## Non-Responsibilities

- Do not modify `Alembic`, `AlembicCore`, `AlembicAgent`, `AlembicDashboard`, `AlembicPlugin`, `AlembicTest`, or real test project source code.
- Do not create implementation wave plans as the authority of record.
- Do not tell implementation windows to start work unless AlembicWorkspace has assigned that action.
- Do not replace the workspace TODO board, test exchange, or final acceptance flow.

## Entry Points

- [AGENTS.md](AGENTS.md): operating instructions for the design window.
- [docs/index.md](docs/index.md): design document map.
- [docs/design-window-operating-policy.md](docs/design-window-operating-policy.md): long-term policy and handoff contract.
- [templates/original-plan-template.md](templates/original-plan-template.md): first-pass requirement framing.
- [templates/requirement-design-template.md](templates/requirement-design-template.md): detailed design format.
- [templates/workspace-handoff-template.md](templates/workspace-handoff-template.md): handoff back to AlembicWorkspace.
