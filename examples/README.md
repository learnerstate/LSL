# Examples

- **minimal-core.yml** — the smallest valid document: one concept claim,
  no extensions.
- **full-core-provenance.yml** — multiple concepts, goals, `teaches_best`,
  `next`, and document-level Provenance (compiler identity via `compiled_by`,
  plus `sources`). This is the "light Provenance" case from SPEC.md §7; it
  doesn't include per-claim `method`/`evidence` (see SPEC.md §4.1 for that
  shape).
- **lsl-example.json** — the JSON serialization of the same document as
  `full-core-provenance.yml`, showing the alternate `text` + `evidence`
  object shape for `can`/`cannot_yet`/`misconceptions` items (§4.1) on the
  Memoization concept.
- **federation-signed.yml** — a canonical, aggregated document using the
  Federation extension: issuer identity, a Data Integrity proof, and
  per-origin claims preserved via `from` blocks.
- **prompt-rendering.txt** — the `?format=prompt&budget=800` plain-text
  rendering of `full-core-provenance.yml`, per SPEC.md section 8.
