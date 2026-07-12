# LSL, explained simply

LSL (Learner State Layer) is a small, open file format that describes what a person currently knows. It's written so a human can read it and an AI can act on it.

This page covers what it is, why it exists, and how to use it. The full technical spec lives in [SPEC.md](./SPEC.md).

---

## The problem

Every learning platform you use builds up a picture of what you know, and every one of them keeps that picture locked inside itself in its own private format. Your coding practice site knows you're solid on algorithms. Your language app knows you keep mixing up two verb tenses. Neither can tell the other, and neither can tell the AI assistant you actually study with.

So the same annoying things keep happening. You can spend a year getting good at algorithms on one platform, then sign up for a new one and get treated like a total beginner, complete with a placement test and intro content, because your progress couldn't come with you. If you learn with an AI, every conversation starts from scratch: it doesn't know you've already mastered recursion, or that you have a stubborn misconception about memoization that keeps wrecking your practice, so you end up re-explaining yourself (badly) every single time. Meanwhile, anyone building a tutor or an adaptive course has to invent their own way of modeling "what does this learner know right now," which means the same expensive problem gets solved privately, over and over, in ways that don't talk to each other.

There are already standards in education technology, and they're good at their job, but it's a different job. xAPI and Caliper record what the learner *did*: an event log along the lines of "completed quiz 14, scored 80%." Nothing standard answers what the learner *knows now*. That's the gap LSL fills. It's a snapshot of current state, compiled from those events, kept up to date, and readable at a glance.

## What it looks like

An LSL document is a short, human-readable file. Here's a fragment:

```yaml
lsl: "1.0"
learner: "learner-8x2k"
updated: 2026-07-08
subject: "Data Structures & Algorithms"

goals:
  - objective: "Interview-ready in DSA"
    by: 2026-09-01

concepts:
  - name: "Memoization"
    mastery: 0.68            # 0–1 estimate of current ability
    confidence: med          # how much to trust that number
    last_seen: 2026-07-08    # knowledge decays; freshness matters
    can:
      - "Writes a correct top-down memoized solution for standard problems"
    cannot_yet:
      - "Analyzes space tradeoffs of memoized vs tabulated solutions"
    misconceptions:
      - "Believes memoization always reduces time complexity to O(n)"

next:
  learn: ["DP Tabulation"]
  review: ["Linked Lists"]
```

Notice there are no quiz scores in there. The file makes claims about the person: what they can demonstrably do, what they've tried and still can't do, and (probably the most useful field of all) what they falsely believe. A tutor that knows your misconception can go fix it directly instead of teaching on top of it.

Notice also that the fields are plain sentences. That's on purpose, and it's the main thing separating LSL from the standards that came before it.

## The key idea: the reader is an AI

Older interoperability standards were built for machines that needed exact, pre-agreed vocabularies to talk to each other. In practice that meant committees, controlled word lists, and years of alignment meetings before anything actually interoperated.

LSL assumes the reader that actually exists now: a language model. Hand an AI tutor the document above and it doesn't need a lookup table to act on it. "Believes memoization always reduces time complexity to O(n)" already tells it exactly what to do. Two platforms can name their concepts completely differently and the AI reconciles them just by reading, the same way a human tutor would make sense of report cards from two different schools.

This is what lets LSL stay small. It skips the universal-taxonomy problem that stalled a whole generation of earlier standards. All it asks is that every claim be a clear sentence, every estimate carry a confidence level and a date, and everything else stay optional.

There's also a built-in `prompt` format. Request a document as `?format=prompt&budget=800` and you get back a token-budgeted plain-text block ready to drop into an AI's context window. If the budget gets tight, misconceptions are protected from being cut, since that's the one thing a tutor most needs to see first.

## What LSL is honest about

A format is only trustworthy if it's upfront about its limits, so a few rules are baked in.

Watching a video isn't the same as knowing something. If all a learner did was watch a video, no LSL compiler is allowed to claim they can now do anything. Capability claims (`can` items) require an actual demonstration; exposure alone gets, at most, the lowest mastery band with low confidence.

Estimates are labeled as estimates. Every mastery number carries a confidence level and a date, and with the optional Provenance layer it also carries the method that produced it plus links back to the underlying evidence.

And unsigned claims aren't credentials. A basic LSL document is closer to a well-organized self-introduction. That's plenty for a tutor, which can verify anything with a couple of follow-up questions, but it's not something to base a hiring decision on. Verified, signed profiles are a separate optional layer (Federation) with cryptographic signatures and per-source records, and the spec explicitly discourages using anything less for high-stakes decisions.

## How adoption works

LSL is layered so you can start small:

**Core** is the plain document shown above. Any learning product can produce valid LSL in an afternoon: a concept list with mastery, confidence, dates, and whatever capability sentences your assessments already support. Empty lists are valid. A thin, honest document beats a rich, fabricated one.

**Provenance** adds the method behind each estimate and links to the underlying evidence. If you already log events, through xAPI or your own system, this is mostly plumbing.

**Relationships** adds prerequisite structure between concepts, for systems that need to sequence a curriculum.

**Federation** adds issuer identity, signatures, and disclosure scopes, for documents that travel between organizations and get aggregated. This is also where privacy rules become mandatory: gap and misconception fields must never be served publicly.

Adoption stays cheap for two reasons. You don't have to change how you store anything, because LSL is a compiled output, a projection of data you already have, whether that's xAPI statements, quiz tables in Postgres, or an AI tutor's session logs. The spec even sorts every field by how hard it is to derive: the simple ones are trivial, the aggregates are easy, and the fields that need real content knowledge can start empty and improve later. There's also no registry and no gatekeeper. Concepts are identified by URLs on your own domain, so anyone can issue LSL today without asking permission.

## What LSL deliberately is not

It doesn't replace xAPI or Caliper; those record history, LSL summarizes the present, and the two are meant to sit side by side. It's not a credential format, since Open Badges and CLR already handle that (though a badge can cite an LSL claim if it wants to). It has no opinion on how anyone should teach. And it stores nothing but claims about the learner: no content, no courses, no sessions.

## Who this is for

If you're building AI tutors or learning tools, consuming LSL means your product is personalized from the first message, with no cold start and no interrogation phase.

If you run a learning platform, emitting LSL turns your users' progress into something they can actually take with them, including into the AI assistants they already use alongside your product.

And if you're the learner, LSL is what makes "my AI already knows what I know" possible. The document is yours: you own it, you decide who sees it, and you can read every claim in it, because every claim is a sentence.

## Status and how to get involved

LSL is a public draft (v1.0). The spec, examples, and a JSON Schema live in this repository. Feedback, implementations, and critical review are all welcome, especially from xAPI profile authors and psychometricians, who will spot the weak points fastest.

Start with [SPEC.md](./SPEC.md). Open an issue if you disagree with something. Build a compiler for your platform and tell us about it. No one owns this format, so the sooner disagreements surface, the better it gets.