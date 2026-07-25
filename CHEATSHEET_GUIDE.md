# How to Write a System Design Cheat Sheet

Instructions for building entries like the ones in `SYSTEM_DESIGN_CHEATSHEETS.md`. Each entry covers one concept/topic (e.g. "Caching", "Sharding", "Distributed Transactions").

## Core constraint: readable in under 1 minute

Every entry must be skimmable by a human in ~60 seconds. This drives everything else:
- Use fragments, not full sentences. Cut filler words.
- Bullet points with **bolded labels**, not prose paragraphs.
- One line per sub-point wherever possible — pack reasoning into a short clause, not a paragraph.
- Prefer `→` to point from a strategy straight to its use case instead of a separate sentence.
- If a section is running long, cut detail, don't add subsections. Create a separate section with subsections following this point instead.

## Required structure

For each concept, cover these four things, in this order:

### 1. Whether to introduce the concept at all (if applicable)
Not every concept is binary-gated (e.g. "REST vs gRPC" is a pure comparison, no "skip" state), but if the concept is something you'd add on top of a simpler baseline (caching, sharding, partitioning, replication, rate limiting, circuit breakers, real-time transport), give a **Skip if** / **Need if** (or equivalent) pair:
- **Skip if:** the conditions under which the simple/default approach is still fine — name concrete thresholds or symptoms, not vague "if not needed."
- **Need if:** the concrete symptoms/conditions that justify adding the complexity.

This is the most important judgment call in the entry — lead with it.

### 2. If it's a head-to-head comparison, give the decision criteria
When the concept is fundamentally "A vs B" (Relational vs NoSQL, Sync vs Async, REST vs gRPC, sync vs async replication), state the deciding factors for each side as short criteria lists, not an essay. Reader should be able to match their situation to a side in one glance.

### 3. Sub-strategies / sub-categories, each with reasoning
Within the concept (or within each side of a comparison), list the actual options (e.g. hash/range/directory/geographic sharding; fixed window/sliding log/sliding counter/token bucket/leaky bucket for rate limiting). For each:
- Say what it is in a few words.
- Give the **tradeoff or defining property** that drives the choice (e.g. "even distribution, poor range queries").
- State when to pick it over the siblings — this is comparative reasoning, not just a definition.

### 4. Real-life example for every concept/sub-item
Every concept, comparison side, and sub-strategy needs a concrete real-world anchor — a company, product, or well-known scenario. Use `→ example` inline at the end of the line rather than a separate sentence. Prefer specific, well-known systems (Stripe, Uber, Netflix Hystrix, Cassandra, Revolut) over generic placeholders ("a big company"). If there's no crisp example, at least name the concrete use-case shape (e.g. "time-series/trading data").

## Formatting conventions to match existing entries

- Numbered `##` heading: `## N. Concept Name`, separated by `---`.
- Open with the skip/need or comparison line(s) before diving into sub-bullets.
- Sub-strategies as a `-` list, format: `**Name**: definition/tradeoff → example`.
- Bold key terms and decision triggers so the eye catches them on a skim.
- End with an "Enforce at / Configure / Implement / Pair with" line only if there's a genuinely useful operational note (where it's applied, key knobs, common tooling) — omit if it doesn't add decision-relevant info.
- Keep total entry length comparable to existing ones (roughly 6-12 lines) — if it's longer, it's failing the 1-minute test.

## Checklist before finalizing an entry

- [ ] Can a human read it in under a minute?
- [ ] Does it say when to use the concept at all (or, for pure comparisons, when to pick each side)?
- [ ] Does every sub-strategy/sub-category have a one-line reason to choose it over its siblings?
- [ ] Does every concept/side/sub-strategy have a concrete real-world example?
- [ ] Is every line a fragment/bullet, not a full paragraph?
