# Changelog

All notable changes to the LSL specification are documented here. This
project does not yet follow strict semantic versioning while in draft —
see [SPEC.md §11](SPEC.md#11-evolution-path) for the planned evolution path.

## [1.0-draft] — 2026-07-12

Initial public draft.

### Added
- Core layer: `lsl`, `learner`, `updated`, `subject`, `goals`, `concepts[]`
  (`name`, `id`, `mastery`, `confidence`, `last_seen`, `trend`, `can`,
  `cannot_yet`, `misconceptions`), `teaches_best`, `next`.
- Extensions: Provenance (evidence and compiler identity), Relationships
  (concept graph structure), Federation (issuer identity, signatures,
  per-origin claims, disclosure scopes).
- The `prompt` rendering format (`?format=prompt&budget={tokens}`) for
  direct LLM context injection.
- JSON Schema (2020-12) for the Core layer.
- Example documents: minimal Core, full Core with Provenance, a signed
  Federation document, and a rendered `prompt` example.
