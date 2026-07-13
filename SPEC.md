# LSL — Learner State Layer

## An LLM-Optimized Learning Inference Schema

**Version:** 1.0-draft · **Status:** Public draft for implementation - breaking changes possible until 1.0-final · **Date:** 2026-07-12

**License:** This specification text is licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Schemas and example code in this repository are MIT-licensed. LSL is open: anyone may implement it without permission or royalty.

*New here? Read [the plain-language explainer](./README.md) first.*

---

## 1. Design Philosophy

### 1.1 The problem, briefly

xAPI, the format most LMS platforms use for tracking activity, can tell you what a learner did: which quizzes they took, which videos they watched, whether they passed. What it can't tell you is what they actually know, where they're confused, or what they should study next. Any AI tutor or adaptive system that wants those answers has to dig them out of raw event logs on its own, which means every platform keeps re-solving the same problem in private.

### 1.2 The central design decision: state, not events

Formats like xAPI is used to record user logs, a running list of statements like "actor did X to object Y." Logs are great for recording what happened. They're much worse for answering "where does this learner stand right now," because anyone who wants current state has to replay the whole history and aggregate it themselves. And since different consumers aggregate differently, two systems reading the same log will happily arrive at two different pictures of the same learner.

LSL goes the other way. It's a snapshot: a compiled summary of what the learner currently knows, organized by concept rather than by event. If you like the database analogy, xAPI is the write-side log and LSL is the view built from it.

That one decision shapes most of what follows in this spec. LSL documents are replaced, not appended to; history stays in xAPI (or whatever internal logs a platform already keeps), and an LSL document only ever describes the present. LSL also only makes claims about the learner, never about the content, the session, or how something was delivered. If a field describes an event rather than a current fact about the learner, it belongs in the logging format instead.

One more consequence worth stating plainly. Raw events are facts. A mastery score is a model's opinion about those facts. LSL doesn't pretend the second thing is the first: any non-trivial claim can state what method produced it.

### 1.3 Written for an AI reader

The intended reader of an LSL document is a reasoning system, an LLM tutor, an adaptive engine, a recommender, at the moment right before it decides what to do next with a learner. That gives us a simple test to apply to every field in the spec:

> Would this field change what a teaching system does in the learner's next session?

Misconceptions, capability statements, the learning frontier, mastery, preferences, and goals all pass, so they're core fields. Things that serve the wider ecosystem without directly changing tutoring behavior (signatures, provenance, merge metadata) live in optional extensions.

It also means the core schema has to be readable without documentation. Hand a capable LLM a bare LSL document with zero explanation and it should interpret it correctly anyway. A field that requires a spec lookup before an LLM can use it has failed. This is why the core field names are in plain English, `can`, `cannot_yet`, `misconceptions`, `next`  instead of technical vocabulary.

### 1.4 Layered, not monolithic

Specs in this space tend to die one of two deaths: too thin to be trusted (unsigned self-published claims nobody can verify) or too heavy to be adopted (a committee standard nobody wants to implement). LSL tries to dodge both with strict layering:

| Layer | Audience | Adds |
|---|---|---|
| **Core** | Anyone, a solo developer, a script, an LLM | The teaching-relevant state. ~10 field types |
| **Provenance** (ext.) | Platforms compiling from xAPI/logs | Evidence links, methods, compiler identity |
| **Relationships** (ext.) | Knowledge-graph and adaptive systems | Prerequisite/graph structure between concepts |
| **Federation** (ext.) | Multi-platform aggregators | Issuer identity, signatures, per-origin claims, disclosure scopes |

Extensions only add fields. They never redefine what a Core field means, so a consumer that understands Core alone and ignores anything unfamiliar loses nothing it actually needed. A document is valid at any single layer on its own, and most implementers can stop at Core and be done.

### 1.5 Two ways to produce a document

LSL can be produced from either direction:

- **From xAPI:** a compiler turns a stream of statements into an LSL document, using the defined and mostly deterministic mapping in  6. Nothing about the schema assumes xAPI exists, though.
- **Standalone:** a platform with no xAPI infrastructure at all can maintain an LSL document directly, updating it whenever its own scoring logic runs.

The resulting document looks identical either way; only the optional Provenance section differs (internal log IDs instead of xAPI statement IDs, or no evidence links at all). This matters because a lot of the platforms most likely to adopt LSL first don't run an LRS and never will. Requiring one would rule them out before they started.

---

## 2. Core Principles

1. **State over events.** LSL describes what's true right now; xAPI describes what happened.
2. **Inference over logging.** Every LSL claim is an interpretation of evidence, and the schema is explicit about that instead of pretending its numbers are raw facts.
3. **Concept-keyed.** Everything is organized around the concept or competency, never the session, course, or content item.
4. **Plain-language claims.** Capabilities and misconceptions are written as ordinary sentences, so both an LLM and the learner they describe can read them directly.
5. **Quantified uncertainty, kept coarse.** Every estimate carries a rough confidence label. Core keeps it simple (`low|med|high`); Provenance can attach real numbers like intervals, observation counts, etc.
6. **Evidence on demand.** Claims may link to evidence, but nothing in Core requires it. Mandatory evidence is a Federation-layer concern; forcing it into Core would burden every cooperative use case to guard against a handful of adversarial ones.
7. **Freshness is first-class.** Knowledge fades. Every concept claim carries a date, and staleness is meaningful.
8. **Additive extensibility.** Unknown fields are ignored, never treated as errors. Extensions cannot change what a Core field means.
9. **The learner owns the document.** Decisions about serving, scoping, and disclosure belong to the learner.
10. **Say what this doesn't do.** Core LSL documents are unverified claims. That's fine for interactive use, since a tutor can check any claim by asking a follow-up question. High-stakes, non-interactive use (hiring, credentialing) needs the Federation layer, and v1 actively discourages using Core for it.

---

## 3. Schema Specification — Core

### 3.1 Serialization and transport

- **Canonical serializations:** YAML (the human/agent default) and JSON. Both use the same structure and the same snake_case keys, so one schema validates both and no key-mapping layer is needed. One document equals one learner in one scope.
- **Transport:** any. Recommended conventions: HTTPS at `{domain}/{learner}/lsl.yml`, optionally scoped as `{domain}/{learner}/{subject-slug}/lsl.yml`; or stored as an xAPI **Agent Profile document** (profileId `https://learnerstate.org/profile/v1`) for LRS-native ecosystems; or passed inline — file upload, MCP resource, injected directly into a prompt by an orchestrator.
- **Browser-to-browser handoff:** a companion convention, **LSL Connect**, defines a one-click consented transfer of a document between two web platforms (popup + `postMessage` with pinned origins; signed documents where the issuer supports the Federation layer). It's specified separately, and Core documents need no changes to participate.
- **Content negotiation:** `?format=yaml|json|prompt`. The `prompt` format returns a token-budgeted plain-text rendering: `?format=prompt&budget=800`.

### 3.2 Top-level structure

```yaml
lsl: "1.0"                     # spec version. REQUIRED.
learner: "lrn-8x2ka"           # identifier. Core: any stable string. REQUIRED.
updated: 2026-07-08            # date (or datetime) of last compilation. REQUIRED.
subject: "Data Structures & Algorithms"   # human name of scope. OPTIONAL.
goals: []                      # learner-declared intents. OPTIONAL.
concepts: []                   # the heart of the document. REQUIRED.
teaches_best: []               # derived learning preferences. OPTIONAL.
next: {}                       # the frontier: recommended actions. OPTIONAL.
```

**Learner identifiers and privacy (normative).** `learner` SHOULD be a pseudonymous, per-scope identifier, not a real name, an email, or a platform username reused across contexts. An LSL document contains a machine-readable record of someone's gaps and misconceptions, and attaching a stable global identifier to that record turns it into a linkable deficit profile. Issuers SHOULD mint opaque per-subject identifiers and keep the mapping to real identity private. Where binding to a real identity is genuinely needed, it belongs in the Federation layer (`learner_aliases`).

### 3.3 Concept claim (the core object)

One entry per concept the learner has meaningful state about.

```yaml
concepts:
  - name: "Memoization"        # REQUIRED. Human-readable concept name.
    id: "https://leetcode.com/concepts/cs/dsa/memoization"
                               # OPTIONAL but recommended. When present, an IRI —
                               # minted on the platform's own domain or drawn from an
                               # external framework (CASE, CTDL). See note below.
    mastery: 0.68              # REQUIRED. 0.0–1.0. See  3.4 for semantics.
    confidence: med            # REQUIRED. low | med | high. How much to trust `mastery`.
    last_seen: 2026-07-08      # REQUIRED. Date of most recent supporting evidence.
    trend: rising              # OPTIONAL. rising | stable | falling. Direction over
                               # the recent evidence window.
    can:                       # REQUIRED (may be empty). Demonstrated capabilities.
      - "Writes a correct top-down memoized solution for standard problems"
    cannot_yet:                # REQUIRED (may be empty). Attempted-and-failed, or
      - "Analyzes space tradeoffs of memoized vs tabulated solutions"   # known gaps.
    misconceptions:            # REQUIRED (may be empty). Active false beliefs.
      - "Believes memoization always reduces time complexity to O(n)"
```

**Semantics that MUST hold:**

- `can` items are evidence-backed. The learner demonstrated this, at least once, to the compiling system's satisfaction. Aspirations and self-reports don't belong here.
- `cannot_yet` items are observed gaps: attempted and failed, or explicitly assessed as absent. Material the learner simply never touched doesn't show up here at all, "hasn't tried yet" is not a gap.
- `misconceptions` are specific false beliefs, stated as the belief itself ("Believes X") rather than rephrased as a skill gap. Of everything in the document, this field changes tutor behavior the most. A gap says teach this; a misconception says un-teach that first.
- All three are plain sentences. No codes, no enum values, no platform-specific slugs. Each sentence should stand on its own without needing the others for context.

**A note on `id`:** when present, `id` is an IRI. Platforms mint IRIs on their own domains (`https://coursera.org/concepts/cs/dsa/memoization`); there's no central registry and none is required, so any platform can host its own without asking permission. Ideally the IRI resolves to a short concept definition (name, description, optional links to CASE/CTDL equivalents), but consumers degrade gracefully if it doesn't, an unresolvable IRI just gets used as an opaque stable key. `name` stays the required human- and LLM-readable label, and renderers should always display the name rather than the raw IRI.

### 3.4 Mastery scale

`mastery` is a 0.0–1.0 estimate of current ability to *use* the concept:

| Range | Interpretation |
|---|---|
| 0.00–0.39 | Exposed; cannot yet explain or apply independently |
| 0.40–0.59 | Can explain; struggles to apply |
| 0.60–0.79 | Applies reliably to standard/familiar problems |
| 0.80–1.00 | Transfers to novel, unseen problems |

Treat the number as a pointer to a band, not a precise measurement. Core alone doesn't guarantee that raw numbers are comparable across platforms; two platforms' 0.7 can mean different things. The Provenance extension's `method` field is what makes a number auditable, and the Federation layer is what makes it comparable across sources.

In practice, consumers should read `mastery`, `confidence`, and `last_seen` together. `0.74 / high / last week` and `0.74 / low / four months ago` call for very different tutoring behavior even though the mastery number is identical.

### 3.5 Goals

```yaml
goals:
  - objective: "Interview-ready in DSA"
    target: "cs/dsa"           # OPTIONAL. Concept or subject id this aims at.
    by: 2026-09-01             # OPTIONAL.
```

Goals are the one section a learner writes directly; everything else in the document is compiled. They're here because a tutor that knows mastery but not purpose ends up optimizing for the wrong thing.

### 3.6 Learning preferences

```yaml
teaches_best:
  - "Worked examples before open-ended practice"
  - "Sessions under 35 minutes"
```

Plain sentences describing how this particular learner learns best. These must be derived from actual differences in outcomes across sessions — never from a self-reported "learning style." If a compiler can't back a preference with an observed outcome difference, it should leave it out. Self-reported learning styles have poor empirical support; measured differences in what actually worked do not.

### 3.7 Next (the frontier)

```yaml
next:
  learn:                       # concepts whose prerequisites are met, not yet started
    - "DP Tabulation"
  review:                      # known concepts predicted to have decayed
    - "Linked Lists"
  blocked:                     # desired concepts with missing prerequisites
    "DP on Graphs": "needs DP Tabulation"
```

`next` is the compiler's precomputed recommendation. Prerequisite-graph traversal and forgetting-curve estimation are exactly the kind of thing an LLM is worst at reconstructing on the fly from a raw transcript, and exactly the kind of thing every adaptive engine otherwise rebuilds for itself, so the compiler does it once. With `next` in place, a consumer with no knowledge graph of its own can still act sensibly from the very first message.

---

## 4. Schema Specification — Extensions

### 4.1 Provenance extension

Adds auditability. Can be attached per-document and per-claim.

```yaml
# document level
compiled_by:
  name: "leetcode-lsl-exporter"
  version: "1.2.0"
sources:
  - "https://lrs.leetcode.com/xapi/"     # or "internal" for non-xAPI platforms

# concept level — richer mastery object replaces the bare fields
concepts:
  - name: "Memoization"
    mastery: 0.68
    confidence: med
    provenance:
      method: "bkt-v2"                   # the model that produced the estimate
      n_observations: 14
      ci: [0.55, 0.79]                   # optional interval
    can:
      - text: "Writes a correct top-down memoized solution for standard problems"
        evidence:                        # xAPI statement IDs, or internal log URIs
          - "urn:uuid:4f2a91d0-...-0a44"
```

Note the shape rule: in Core, `can` items are bare strings; with Provenance, they can be objects with `text` + `evidence`. Consumers MUST accept both forms (a string is equivalent to an object with only `text` set).

### 4.2 Relationships extension

For knowledge-graph consumers. Declares structure *between* concepts in this document:

```yaml
relationships:
  - from: "https://leetcode.com/concepts/cs/dsa/memoization"
    to: "https://leetcode.com/concepts/cs/dsa/recursion"
    type: requires              # requires | part_of | related
```

This was left out of Core on purpose. Most tutors work fine off the precomputed `next` block and never need the raw graph; curriculum sequencers and KG builders can opt in when they do.

### 4.3 Federation extension

For multi-platform aggregation, and for any consumer acting on the document without directly interacting with the learner.

```yaml
document_type: local            # local | canonical
issuer: "https://leetcode.com"
learner_aliases: ["did:web:sarah.dev"]
disclosure: tutor               # self | tutor | recommender | public
proof:                          # W3C Data Integrity signature over canonical form
  type: "DataIntegrityProof"
  cryptosuite: "eddsa-jcs-2022"
  verification_method: "https://leetcode.com/.well-known/lsl-keys#key-2026"
  proof_value: "z8KpQm..."

# in canonical (aggregated) documents, per-origin claims survive the merge:
concepts:
  - name: "Recursion"
    mastery: 0.82
    confidence: high
    from:
      - issuer: "https://leetcode.com"
        mastery: 0.85
        provenance: { method: "elo", n_observations: 19 }
      - issuer: "https://coursera.org"
        mastery: 0.78
        provenance: { method: "quiz-average", n_observations: 12 }
```

Federation rules, summarized (normative): aggregators verify origin signatures before citing a claim. Merges are represented via `from` blocks, never applied by silently overwriting. Conflicts get resolved by pooling evidence, or by recording new adjudicating evidence as an ordinary statement in the aggregator's own source, never by rewriting history. Disclosure scopes gate the sensitive fields: `cannot_yet` and `misconceptions` MUST NOT be served below `tutor` scope.

---

## 5. Why These Fields Exist (compared to xAPI)

xAPI already covers a lot of ground, so each field here has to justify itself. This section walks through what xAPI is missing that makes each one necessary, and what adding it costs.

**`concepts` — organizing by concept instead of activity.** xAPI is keyed by activity (a specific content item), and the link from activity to concept, if it exists at all, lives buried in `contextActivities` or in an activity definition with no standard shape. You can't reliably read that link back out: two platforms' activities on "memoization" won't share an identifier, and aligning them requires a taxonomy decision xAPI never makes. LSL makes concept the primary key and drops activity out of view entirely. The cost is that someone has to own the activity-to-concept mapping, and doing that well is the single largest cost of building an LSL compiler.

**`mastery`, `confidence`, `trend`.** xAPI's `result.score` is a grade on one event, not an estimate of ability. There's no standard way to aggregate scores across events, no uncertainty measure, no notion of decay. Any aggregate needs a model (average? Elo? Bayesian knowledge tracing? recency-weighted?) and xAPI deliberately doesn't pick one, so two consumers reading the same statements silently choose different models. LSL carries one estimate, produced by a method that can be declared in Provenance, with a coarse trust label attached. That estimate is only as good as the compiler behind it. LSL makes the model visible; it can't make the model good.

**`can` / `cannot_yet` — plain-language capability claims.** The closest thing xAPI has is verb plus activity ("passed quiz-14"), which tells a consumer nothing about what ability that demonstrates. That meaning lives in the content platform, not in the statement stream, so it can't be recovered from raw events. The compiler, which does know its own content, writes the capability sentence directly. The tradeoff is that free text doesn't support SQL-style analytics; consumers who want structured joins should match on `id`, not on sentence text. LSL accepts that tradeoff because its primary consumer is an LLM, which handles sentences better than it handles enum codes.

**`misconceptions`.** This doesn't exist in xAPI at all. `result.success: false` records a failure, not the belief that caused it. Diagnosing a misconception means reading the pattern across wrong answers against the content's distractor design, and only the platform (or an LLM reviewing the responses) has access to that. LSL gives the diagnosis a dedicated field, stated as the false belief itself. It's also the highest-value field and the hardest one to compile reliably. Compilers without misconception detection should just emit empty lists.

**`last_seen` and decay-driven `review`.** xAPI timestamps events but has no concept of knowledge decay; a "passed" from three years ago and one from yesterday read identically. LSL puts a freshness date on every claim, and predicted decay surfaces through `next.review`. The decay model itself is compiler-dependent. Core only exposes the conclusion ("review this"), and Provenance can expose the reasoning behind it.

**`goals`.** xAPI can record the event of setting a goal, but has no standing representation of current intent. LSL gives goals a dedicated, learner-owned section. This is the one field that's meant to be self-reported, because intent genuinely can't be observed from the outside.

**`teaches_best`.** Nothing in xAPI relates delivery style to outcomes. Getting there requires a cross-session regression (gains against content style) using both the event history and content metadata, which single statements can't provide. The compiler runs that comparison and reports only findings backed by an actual outcome difference. Done carelessly, this field risks resurrecting learning-styles pseudoscience, which is exactly why  3.6 makes "derived only" a hard rule rather than a suggestion.

**`next`.** xAPI has no prerequisite model and no notion of a recommendation, so LSL precomputes the frontier. That does embed the compiler's own curricular opinions into the document, which is why `blocked` states its reasons: a consumer that disagrees can override it.

---

## 6. Deriving LSL from xAPI

### 6.1 Mapping classes

Every LSL field falls into one of three derivation classes:

| Class | Meaning | Fields |
|---|---|---|
| **D — Direct** | Deterministic 1:1 mapping from statement fields | `learner`, `last_seen`, `updated`, evidence IDs |
| **A — Aggregation** | Deterministic *given a declared algorithm* over a statement set | `mastery`, `confidence`, `trend`, `next.review` |
| **S — Semantic inference** | Requires content knowledge or a model; not derivable from statements alone | `can`/`cannot_yet` sentences, `misconceptions`, `teaches_best`, concept mapping, `next.learn`/`blocked` |

This table is really just an honest answer to "how hard is this conversion." D is trivial. A is easy and reproducible once you pick an algorithm. S is where the real engineering work lives.

A minimal compliant compiler can ship D and A only, emit empty S fields, and still produce a valid, useful document: a mastery map with freshness and a review list. S fields get filled in as the compiler gains content awareness, whether through hand-built mappings or an LLM-assisted pipeline.

### 6.2 Field mapping table

| LSL field | xAPI source | Class | Rule |
|---|---|---|---|
| `learner` | `actor` (account or mbox, hashed/aliased) | D | Stable pseudonymous identifier per actor ( 3.2) |
| `concepts[].id` | `object.id` → concept, via compiler's activity→concept map; or `contextActivities.grouping` | S | The load-bearing mapping decision |
| `concepts[].last_seen` | max `timestamp` of contributing statements | D | — |
| `concepts[].mastery` | `result.score`, `result.success`, verb semantics over the statement set | A | Declared algorithm (`method` in Provenance) |
| `concepts[].confidence` | observation count + variance from the same set | A | e.g., n<5 → low; 5–15 → med; >15 → high (compiler-defined, declared) |
| `concepts[].trend` | time-windowed comparison of the estimate | A | — |
| `can[]` | statements with mastery-indicating verbs (`mastered`, `passed`, `completed`+success) × content's competency description | S | Compiler translates "passed activity X" into the capability X assesses |
| `cannot_yet[]` | `failed`/`success:false` patterns × content descriptions | S | Persistent failure, not single miss ( 9.1) |
| `misconceptions[]` | `result.response` wrong-answer patterns × distractor semantics | S | Highest value, hardest to automate; LLM-assisted review of responses is a legitimate compiler strategy |
| `goals[]` | statements with goal verbs, if the platform records them; else platform UI | D/S | Pass-through of declared intent |
| `teaches_best[]` | cross-session regression: gains vs. content metadata | S | Emit only above an outcome-difference threshold |
| `next.review` | decay model over `last_seen` + mastery | A | — |
| `next.learn`/`blocked` | prerequisite graph (compiler-owned) × mastery map | S | Graph is compiler knowledge, not xAPI data |
| Provenance `evidence` | statement `id` | D | — |

**Statement voiding:** compilers MUST honor xAPI voiding. A voided statement drops out of all A and S computations at the next compile. This is one place where state documents are genuinely easier to work with than event logs: recompile and the claim quietly corrects itself, whereas systems consuming the raw event log tend to get this wrong.

### 6.3 Worked mapping example

**Input — three xAPI statements (abbreviated):**

```json
{"id":"urn:uuid:4f2a...", "timestamp":"2026-07-01T09:12:00Z",
 "actor":{"account":{"name":"sarah"}},
 "verb":{"id":".../verbs/passed"},
 "object":{"id":"https://leetcode.com/problems/climbing-stairs",
   "definition":{"name":{"en":"Climbing Stairs"},
     "extensions":{".../concepts":["https://leetcode.com/concepts/cs/dsa/memoization"]}}},
 "result":{"success":true,"score":{"scaled":0.9}}}

{"id":"urn:uuid:9c1b...", "timestamp":"2026-07-04T10:02:00Z",
 "verb":{"id":".../verbs/failed"},
 "object":{"id":"https://leetcode.com/problems/edit-distance-space",
   "definition":{"extensions":{".../concepts":["cs/dsa/memoization"],
     ".../assesses":"space-complexity analysis of DP solutions"}}},
 "result":{"success":false,
   "response":"chose O(n) for time complexity of memoized fib with O(1) claimed space"}}

{"id":"urn:uuid:d81f...", "timestamp":"2026-07-08T11:00:00Z",
 "verb":{"id":".../verbs/passed"},
 "object":{"id":"https://leetcode.com/problems/house-robber-tabulation"},
 "result":{"success":true,"score":{"scaled":0.8}}}
```

**Compiler steps:** (1) group by concept using the activity→concept map [S]; (2) run the mastery model over the set [A]; (3) translate passed activities into capability sentences using each activity's `assesses` metadata [S]; (4) flag the persistent space-complexity failure as `cannot_yet`, and flag the response pattern, repeatedly claiming O(n)/O(1) for memoized solutions, as a misconception candidate [S]. The platform's account name (`sarah`) gets aliased to a pseudonymous LSL identifier during this step, per  3.2.

**Output — the affected fragment of the learner's LSL document:**

```yaml
concepts:
  - name: "Memoization"
    id: "https://leetcode.com/concepts/cs/dsa/memoization"
    mastery: 0.68
    confidence: med
    last_seen: 2026-07-08
    trend: rising
    can:
      - "Writes a correct top-down memoized solution for standard problems"
      - "Converts a memoized solution to bottom-up tabulation with guidance"
    cannot_yet:
      - "Analyzes space complexity tradeoffs of memoized vs tabulated solutions"
    misconceptions:
      - "Believes memoization always reduces time complexity to O(n)"
```

Individual timestamps, scores, and response texts get compressed away here, deliberately. They remain recoverable through Provenance `evidence` links back to the original statements. What's gained is what no single statement contained: the capability sentences, the misconception, and the aggregate estimate.

### 6.4 Standalone (non-xAPI) derivation

A platform with no LRS follows the same three-class model against its own logs. D fields come from its user/event tables, A fields from its own scoring model, S fields from its content metadata. Provenance `evidence` can cite internal URIs, or be left out entirely. The resulting document is byte-compatible with one derived from xAPI, consumers never need to know or care which pipeline produced it.

---

## 7. Complete Example Document (Core + light Provenance)

```yaml
lsl: "1.0"
learner: "lrn-8x2ka"
updated: 2026-07-08
subject: "Data Structures & Algorithms"
compiled_by: { name: "leetcode-lsl-exporter", version: "1.2.0" }
sources: ["https://lrs.leetcode.com/xapi/"]

goals:
  - objective: "Interview-ready in DSA"
    by: 2026-09-01

concepts:
  - name: "Recursion"
    id: "https://leetcode.com/concepts/cs/dsa/recursion"
    mastery: 0.85
    confidence: high
    last_seen: 2026-05-30
    trend: stable
    can:
      - "Designs recursive solutions with correct base cases for novel problems"
    cannot_yet: []
    misconceptions: []

  - name: "Linked Lists"
    id: "https://leetcode.com/concepts/cs/dsa/linked-lists"
    mastery: 0.74
    confidence: high
    last_seen: 2026-06-28
    trend: stable
    can:
      - "Explains node/pointer structure and traverses correctly"
      - "Implements insertion and deletion at any position"
    cannot_yet:
      - "Handles empty-list and single-node edge cases without prompting"
    misconceptions: []

  - name: "Memoization"
    id: "https://leetcode.com/concepts/cs/dsa/memoization"
    mastery: 0.68
    confidence: med
    last_seen: 2026-07-08
    trend: rising
    can:
      - "Writes a correct top-down memoized solution for standard problems"
      - "Converts a memoized solution to bottom-up tabulation with guidance"
    cannot_yet:
      - "Analyzes space complexity tradeoffs of memoized vs tabulated solutions"
    misconceptions:
      - "Believes memoization always reduces time complexity to O(n)"

teaches_best:
  - "Worked examples before open-ended practice"
  - "Sessions under 35 minutes"

next:
  learn: ["DP Tabulation"]
  review: ["Linked Lists"]
  blocked:
    "DP on Graphs": "needs DP Tabulation"
```

That's the whole document, about thirty effective lines. The acceptance test for this spec: hand it to any competent LLM with the instruction "tutor this person" and you should get a well-targeted first session.

---

## 8. The `prompt` Format

A normative plain-text rendering meant for direct injection into LLM context, requested via `?format=prompt&budget={tokens}`.

**Budget semantics.** `budget` is a ceiling in tokens, not a target. The renderer stops before going over and never pads to fill space. If omitted, it defaults to 800. Renderers should report the actual token count of the response (e.g. an `X-LSL-Tokens` header over HTTP) so orchestrators can account for it precisely.

**Section priority.** Sections render in this fixed order, and rendering stops as soon as the next section would exceed the remaining budget:

1. Identity line (learner · subject · updated)
2. Goals
3. Active misconceptions (⚠-prefixed)
4. cannot_yet (merged across concepts)
5. next (learn | review | blocked, with reasons)
6. Mastery summary (name, estimate, confidence, trend, staleness)
7. can (merged across concepts)
8. teaches_best

Two hard rules. First, misconceptions never get dropped while any budget remains after the identity line, if that means they render alone, so be it. Second, under budget pressure, `can` lists get truncated first, since they're the most redundant with the mastery summary anyway.

The ordering follows roughly how much harm comes from each section going missing. A tutor that doesn't know a learner's strengths wastes a little time re-verifying them. A tutor that doesn't know about an active misconception teaches on top of a false belief.

```
LEARNER: lrn-8x2ka · DSA · updated 2026-07-08
GOAL: interview-ready by 2026-09-01
MISCONCEPTION: believes memoization always yields O(n) time — probe & correct first
CANNOT YET: space tradeoffs memo vs tabulation · LL empty/single-node edge cases
NEXT: learn DP Tabulation | review Linked Lists | blocked: DP on Graphs (needs Tabulation)
MASTERY: recursion 0.85↑high · linked-lists 0.74 high (stale 10d) · memoization 0.68↑med
CAN: novel recursive design w/ base cases · top-down memo solutions · LL insert/delete
TEACHES BEST: worked examples first · <35 min sessions
```

That's around 140 tokens. Don't mistake this format for an afterthought, for AI adoption it's probably the single feature that matters most, because it turns learner state into something a consumer can fetch once, fit in a fixed token budget, and drop straight into context.

---

## 9. Edge Cases

**9.1 A single failure isn't `cannot_yet`.** One wrong answer is noise. `cannot_yet` requires either repeated failure, a designed assessment of that specific capability, or explicit compiler confidence; otherwise the concept just shows lower mastery and lower confidence. The same threshold logic applies to misconceptions: a pattern of consistently wrong-belief-shaped answers, not one slip.

**9.2 Exposure isn't assessment.** The learner watched a video and nothing followed it up. xAPI records `experienced`. LSL's rule: consumption alone never generates a `can` item, and moves mastery at most into the lowest band with `confidence: low`. This deliberately breaks from completion-oriented standards like SCORM, where "completed" was treated as equivalent to competence.

**9.3 Self-assessment.** "I already know recursion." That gets recorded (as an xAPI statement or a platform input) and the compiler can use it as weak evidence affecting mastery or confidence, but it never becomes a `can` item — those are reserved for actual demonstration. If a platform wants to surface an unverified self-claim, it belongs in `goals` ("validate existing recursion knowledge").

**9.4 Forgetting and stale documents.** A document compiled in March, read in July. Consumers MUST treat `updated` and per-concept `last_seen` as load-bearing; stale high mastery is really a review candidate. Compilers SHOULD recompile on a schedule even without new events, so the decay model can lower `confidence` and populate `review` as time passes.

**9.5 Concept granularity mismatch.** Platform A models "linked lists" as one concept; platform B splits it into "singly-" and "doubly-linked lists." No schema fixes this. It's a taxonomy problem, and LSL's mitigations are partial: `id` alignment to external frameworks where one exists, `relationships.part_of` for expressing nesting, and pushing crosswalking work to Federation-layer aggregators. A Core consumer just sees two or three concepts where there should logically be one. Imperfect, but nothing breaks.

**9.6 Conflicting evidence at compile time.** Passed hard problems on Monday, failed easy ones on Wednesday. That's the compiler's problem, its model has to handle noise , but the schema gives it honest ways to express the result: lower `confidence`, `trend: falling`, and if the split is real and persistent, a `cannot_yet` naming the specific failing subskill.

**9.7 Gaming and false positives.** Copied solutions inflate mastery. LSL inherits this weakness from its sources; garbage statements in, confident garbage out. The mitigations live in Provenance (methods that weight proctored or verified assessments) and in the assumption that the consumer is interactive. A tutor probing a claimed 0.85 will usually find the truth within two questions. This is also why  2.10 warns against non-interactive, high-stakes use of unsigned documents.

**9.8 Group activities.** An xAPI statement with a team as the actor. LSL is strictly per-learner: compilers either attribute credit to individuals, if the platform can tell them apart, or exclude group statements from individual mastery calculations, optionally noting the collaborative work as weak evidence.

**9.9 Multiple subjects and scoping.** One learner, many domains. A full-feed document can hold every concept the learner has state on, and the serving convention supports subject-scoped URLs on top of that. A consumer requesting `?scope=` gets back a filtered document with exactly the same structure, scoping never changes the shape of the document, only its contents.

---

## 10. Trade-offs and Weaknesses

Worth being direct about these, since a spec that hides its weak points just attracts the wrong adopters.

1. **The schema can't be better than the compiler behind it.** LSL standardizes how inference gets expressed, not how good that inference is. A bad mastery model still produces a fluent, well-formed, wrong document, and arguably that's worse than raw logs, because it reads as authoritative. Provenance makes the model auditable. Nothing makes it correct.
2. **Core numbers aren't comparable across platforms.** By design, comparability is left to Federation, but consumers will compare the raw numbers anyway. The best the spec can do is keep insisting: read mastery with confidence and method, never alone.
3. **Free-text claims resist ordinary analytics.** SQL-style systems can work with concept IDs and mastery numbers fine, but the richest fields (`can`, `misconceptions`) are prose and don't join cleanly. If it turns out that classical data pipelines, not LLMs, become the main consumers of these documents, LSL will have optimized for the wrong reader.
4. **The hard fields (class S) concentrate almost all the difficulty.** Misconception detection and capability translation are genuinely hard, and many early compilers will ship with empty S fields. Early LSL documents will therefore mostly be bare mastery maps rather than the rich documents in the examples. That's an acceptable starting point, an accurate mastery map is already useful, but early adopters should calibrate their expectations accordingly.
5. **Taxonomy is still unsolved.** LSL defers concept alignment to external frameworks and to aggregators rather than inventing its own, because the alternative is yet another ungoverned namespace. It does mean cross-platform concept identity is only as good as whatever frameworks end up getting adopted.
6. **Privacy is a real concern, and Core's protections are soft.** `cannot_yet` and `misconceptions` amount to a machine-readable deficit record, a document that says, in plain sentences, what someone is bad at and what they falsely believe. In the wrong hands (an employer, an insurer, a data broker) that's exactly the kind of profile people should worry about. Core's mitigations are conventions rather than enforcement: pseudonymous identifiers ( 3.2), learner ownership, and a disclosure rule that only becomes mandatory in Federation. A Core-only deployment served publicly can absolutely leak sensitive information, and nothing in the schema will stop it. Implementations should default to private serving, full stop.
7. **Unsigned claims will get misused anyway.** Someone will screen job candidates against Core documents despite  2.10 telling them not to. The spec can warn against it; it can't stop it.
8. **Replacing state loses history's texture.** A recompile that fixes an error also erases the trace of that error, unless the original sources are kept around separately. Anyone doing longitudinal research needs to retain the xAPI layer for that. LSL isn't an archival format.

---

## 11. Evolution Path

**v1.x (additive, near-term):** A JSON-LD context mapping Core fields to CTDL/CASE terms, so graph consumers get real semantics without any change to the YAML documents themselves. A registered xAPI profile for LSL-relevant verbs and the `concepts`/`assesses` activity extensions, making statement streams easier to compile in the first place. A reference open-source compiler (xAPI → LSL, D+A complete, with an optional LLM-assisted S pipeline). The LSL Connect companion convention ( 3.1) for consented browser-to-browser handoff. A conformance test suite: golden statement sets mapped to expected documents.

**v2 (depends on adoption):** Federation hardening, key discovery conventions, canonical-document conformance, and a dispute/correction protocol where a learner contests a claim and the issuer either re-evidences it or retracts it. A delta/webhook convention (`lsl-updated` ping) so consumers can subscribe instead of polling. A standard disclosure-scope vocabulary, finalized after privacy review.

**v3 (research-dependent):** Calibration benchmarks that would allow bounded cross-method comparability, by mapping a platform's own method onto reference assessments. Possibly an affect/engagement extension, though it stays explicitly excluded until measurement validity and privacy governance actually mature. Group/cohort documents for classroom orchestration.

Some things are permanently out of scope: replacing xAPI as the event record, credentialing (that's CLR/Open Badges territory — a badge citing an LSL claim is the right relationship, not competition), prescribing pedagogy, and storing content.

---

## 12. How LSL Fits Next to Existing Standards

| | xAPI | SCORM/cmi5 | Caliper | CLR / OB3 | **LSL** |
|---|---|---|---|---|---|
| Unit | event | course session | event | credential | **learner-state claim** |
| Answers | what happened | did they complete it | what happened (edu-vocab) | what was awarded | **what do they know now** |
| Consumer | LRS/analytics | LMS | analytics | verifiers | **AI tutors, adaptive engines** |
| Verified | via LRS auth | via LMS | via platform | signed | optional (Federation) |
| LLM-ready | no | no | no | partially | **by design (`prompt` format)** |

None of these are competitors. LSL fills a gap the rest were never built to cover: giving reasoning systems a fast, current answer to what a learner actually knows, instead of making them reconstruct it from an event log every time.

---

*End of specification draft. See [CHANGELOG.md](https://github.com/learnerstate/LSL/blob/main/CHANGELOG.md) for revision history. Feedback, reference-compiler collaboration, and critical review are all welcome, especially from xAPI profile authors and psychometricians.*

 
*Drafting note: this document was formatted with LLM assistance. The design decisions and normative rules are the maintainers', who reviewed the full text; report anything that reads wrong via an issue, same as any other bug.*
 