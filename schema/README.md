# Schema

`lsl-core.schema.json` is the JSON Schema (2020-12) for the Core layer
defined in [SPEC.md §3](https://github.com/learnerstate/LSL/blob/main/SPEC.md#3-schema-specification--core): `lsl`,
`learner`, `updated`, `subject`, `goals`, `concepts[]`, `teaches_best`,
`next`.

It intentionally allows unknown/additional properties, since a document
using the Provenance, Relationships, or Federation extensions ([SPEC.md
§4](https://github.com/learnerstate/LSL/blob/main/SPEC.md#4-schema-specification--extensions)) is still a valid Core
document — extensions are strictly additive. This schema does not validate
extension field shapes; check those against SPEC.md §4 by hand.

## Validate a document

```sh
npx ajv-cli validate \
  -s schema/lsl-core.schema.json \
  -d examples/minimal-core.yml \
  --spec=draft2020
```

`ajv-cli` parses YAML transparently, so this works against the `.yml`
examples directly. This is the same command CI runs against every file in
`examples/` — see
[.github/workflows/validate-examples.yml](../.github/workflows/validate-examples.yml).
