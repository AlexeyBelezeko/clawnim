Clawnim — coding agent orchestration platform inspired by nanoclaw.

Spec-driven development: specification is the source of truth. All changes must be reflected in specs before or alongside code. Docs live in `docs/` (adr, references, components).

- `docs/components/` — main spec. Describes what each part does and how it behaves. Language-agnostic.
- `docs/adr/` — decision history. Records why choices were made. Do not use as implementation reference.
- `docs/references/` — external context (nanoclaw source structure, etc).
