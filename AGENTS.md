# Architecture repository guide

This repository defines Trajectory system boundaries, service ownership, data flow, security decisions, ADRs, plans, and PlantUML diagrams.

## Sources and scope

- Start at `Системный анализ/README.md`, then read only documents relevant to the task.
- Service specifications live in `Системный анализ/services/`.
- Cross-cutting design lives in `Системный анализ/architecture/`.
- Durable decisions live in `Системный анализ/adr/`.
- Delivery briefs live in `Системный анализ/plans/`; a plan does not override an accepted ADR or service invariant without an explicit change.
- Engineering conventions live in `Системный анализ/engineering/code-conventions.md`.

Keep documentation internally consistent. When a change affects a boundary, API ownership, data flow, security rule, or delivery guarantee, update every directly affected document and diagram. Add or amend an ADR only for a durable architectural decision, not routine implementation detail.

## Writing and diagrams

- Documentation prose and diagram descriptions use Russian.
- Technology names, service names, event subjects, RPC methods, schema fields, and code identifiers use English.
- ADRs use: Context, Decision, Alternatives, Consequences.
- Preserve existing PlantUML and C4 style. Do not generate or commit rendered images unless requested.
- Validate changed `.puml` files with `plantuml -checkonly <files>` when PlantUML is available. If unavailable, report that check as not run.

Keep edits focused. Do not rewrite unrelated documents for style. Cite exact documents that establish a decision or reveal a conflict.
