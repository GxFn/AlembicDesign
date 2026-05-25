# AlembicDesign Operating Policy

更新日期：2026-05-25

## Purpose

AlembicDesign separates requirement discussion from implementation dispatch. It gives the user a focused place to explore ideas while AlembicWorkspace remains the controller for execution, acceptance, TODO ownership, and archive.

## Control Relationship

- AlembicWorkspace owns final state, current mainline, TODO priority, window dispatch, test handoff, acceptance, and archive.
- AlembicDesign owns exploratory requirement discussion and design draft preparation.
- A design draft does not become executable work until AlembicWorkspace accepts it and creates the relevant workspace confirmation / wave documents.

## Allowed Work

- Clarify product goals and user scenarios.
- Draft original plans and requirement designs.
- Compare architecture options and tradeoffs.
- Identify code-research requests for AlembicWorkspace or source-repo windows.
- Produce workspace handoff drafts.
- Maintain design-window templates and local design docs.

## Forbidden Work

- Product source changes.
- Direct test execution for real projects.
- Direct cross-repo task dispatch.
- Changing AlembicWorkspace current status as the source of truth.
- Writing secrets, private tokens, or user machine-specific data into long-term docs.

## Handoff Contract

Each handoff to AlembicWorkspace should include:

1. Requirement title and user goal.
2. Design status: draft, user-confirmed, blocked, or ready for workspace review.
3. Final completion definition.
4. Known facts and evidence.
5. Open questions and decisions.
6. Proposed repository coverage.
7. Suggested phase order.
8. Validation requirements.
9. Non-goals and forbidden shortcuts.

The handoff should be clear enough that AlembicWorkspace can decide the next step without replaying the whole discussion.

## Current Mainline Awareness

AlembicDesign must not interrupt the active AlembicWorkspace mainline by default. If a new idea arrives while implementation is running, AlembicDesign may draft it and mark whether it is:

- background discussion,
- TODO candidate,
- next-mainline candidate,
- urgent blocker for the current mainline.

Only AlembicWorkspace can promote it into the current execution line.
