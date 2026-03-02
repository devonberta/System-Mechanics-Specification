# System Mechanics Specification (SMS)

SMS is an **intent-first**, **normative**, **verifiable** specification for describing the full lifecycle of software systems—product intent, system mechanics, governance, and operational posture—in one artifact surface.

The goal is not “better documentation.” The goal is a foundation for **spec-driven software factories**: a world where the unit of truth is a validated artifact set, and software can be produced and evolved with deterministic safety gates.

## Why this exists (the thesis)

As software throughput increases (especially with AI), organizations hit a familiar failure mode: **faster code, slower coherence**. Without a shared, verifiable unit of truth, change accelerates entropy faster than value.

SMS is an attempt to put the “meaning of the system” into a form that can be:

- **validated** deterministically (normative MUST/SHALL/MAY rules)
- **bounded** by explicit **invariants** (e.g., transport boundaries, governance enforcement)
- **promoted and audited** as versioned **artifacts** via a **control plane**
- **compiled** into runtime scaffolding and behavior, then **verified** back against the declared constraints

## Core ideas (aligned with the article series)

- **Intent-first**: declared intent + flows/transitions are primary; endpoints, services, and UIs are compilation targets.
- **Safety boundary = validators + gates**: LLMs can propose artifacts; deterministic validation decides what becomes real.
- **The 95% / 5% split**:
  - **95% pattern-based scaffolding**: deterministic generation of repeated structure (wiring, routing bindings, storage/caching bindings, observability hooks, retries/backoff, client sync/cache posture).
  - **5% constraint-to-logic synthesis**: compile declared constraints, policies, invariants, guards, and transition rules into executable behavior—then verify it against those same declarations.
- **Build + run closure (v1.6)**: infrastructure intent is part of the same artifact set (resources, storage, caching, routing, client behavior), so operations is not a separate language.

## Start here (SMS v1.6 optimized)

The recommended entrypoint is `1.6_spec_optimized/` (consolidated docs, optimized for humans and LLM workflows):

- **Specification (normative)**: [`SMS-v1.6-Specification.md`](1.6_spec_optimized/SMS-v1.6-Specification.md)
- **Quick reference**: [`SMS-v1.6-Quick-Reference.md`](1.6_spec_optimized/SMS-v1.6-Quick-Reference.md)
- **Implementation guide**: [`SMS-v1.6-Implementation-Guide.md`](1.6_spec_optimized/SMS-v1.6-Implementation-Guide.md)
- **Reference examples**: [`SMS-v1.6-Reference-Examples.md`](1.6_spec_optimized/SMS-v1.6-Reference-Examples.md)
- **Formal EBNF grammar**: [`SMS-v1.6-EBNF-Grammar.md`](1.6_spec_optimized/SMS-v1.6-EBNF-Grammar.md)

## Article series (software factory framing)

If you want the narrative framing first, start with the series posts:

- [Article 0 — The End of “Software as Craft”](articles/medium-article-0-the-end-of-software-as-craft.md)
- [Article 1 — Two Companies, Same Destination](articles/medium-article-1-two-companies-same-destination.md)
- [Article 2 — Convergence — One Standard Across Product + Engineering + Ops](articles/medium-article-2-convergence-one-standard.md)
- [Article 3 — The Software Factory — Deterministic Production, Verified Safety](articles/medium-article-3-the-software-factory.md)
- [Article 4 — Operating the Factory — From DevOps to System Stewardship](articles/medium-article-4-operating-the-factory.md)
- [Article 5 — Value Migration — From Code to Business Integration](articles/medium-article-5-value-migration.md)
- [Article 6 — The “Intent Interface” Replaces Apps](articles/medium-article-6-intent-interface.md)

## Version history

This repository includes multiple versions so the evolution can be inspected:

- `1.1_1.3/`, `1.4_spec/`, `1.5_spec/`, `1.6_spec/`: modular documents by topic
- `*_spec_optimized/`: consolidated, “reader-first” documents (recommended for most readers)

## Notes

This specification is still evolving. Contributions are welcome, especially around strengthening invariants, improving normative validation clarity, and expanding reference examples while preserving the intent-first boundary.