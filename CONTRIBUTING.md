# Contributing to LSL

LSL gets better through disagreement — the goal here is to have that
disagreement productively.

## Before opening an issue

- Read [SPEC.md](SPEC.md). If your concern is about non-normative
  behavior (vocabularies, prompt rendering, sync), it may belong outside
  Core rather than as a change to it.
- Check open issues and discussions for prior art — many objections to
  Core's shape have already been debated once.

## Kinds of contribution

- **Spec feedback** — a Core field is wrong, ambiguous, or missing. Use
  the "Spec feedback" issue template. Explain the failure mode, not just
  the preference: what breaks, for whom, under what document?
- **Implementation reports** — you built something against LSL. Tell us
  what worked, what you had to work around, and where the spec's words
  didn't match what you needed to build. These reports are the primary
  input to what changes before 1.0.
- **Editorial** — typos, unclear wording, broken examples. PRs welcome
  without prior discussion.

## How disagreements get resolved

1. State the problem before the solution. "Field X can't represent Y"
   is more useful than "field X should be renamed."
2. Prefer extension over redefinition. If a change can live in `x-`
   fields or a companion spec without touching Core, it should.
3. Changes to Core require a documented failure case — not just an
   aesthetic preference — since every Core field is a promise every
   implementation has to keep.
4. When editors disagree, the document optimizes for the smallest Core
   that still lets independent implementations interoperate.

## Code changes

Schema and examples are covered by [LICENSE-CODE](LICENSE-CODE) (MIT).
Before submitting:
- Validate examples against the schema — see
  [schema/README.md](schema/README.md).
- Add or update an example if you change what a valid document looks
  like.
