# Documentation map

These documents are inputs, not an archive. They have different **authority** and different
**audiences**, and loading the wrong set into an agent's context makes it worse, not better.

Total repository documentation: ~3,900 lines.
**What an agent is given: ~300 lines.**

That ratio is deliberate.

---

## Precedence (highest first)

| Rank | Document | Governs | On conflict |
| --- | --- | --- | --- |
| 1 | [`SCHEMA.md`](SCHEMA.md) | the `Finding` contract, ids, evidence, time | wins over everything |
| 2 | [`../AGENTS.md`](../AGENTS.md) | rules of engagement for coding agents | wins over any task card |
| 3 | [`TASKS.md`](TASKS.md) | the current task and its acceptance tests | wins over plan/roadmap |
| 4 | [`PLAN.md`](PLAN.md) | scope, budgets, crate map, schedule | wins over all reference docs |
| 5 | [`PROJECTS.md`](PROJECTS.md) | what belongs in this repo vs another | wins over VISION/FLEET |
| 6 | Reference | mechanism, rationale, history | amended by anything above |

If two documents disagree, the higher-ranked one is correct **and the lower one gets a PR**.

---

## Reading sets

### Agent set (~300 lines)

1. `AGENTS.md` — the rules (103 lines)
2. `docs/SCHEMA.md` — the contract (149 lines)
3. **One** task card from `docs/TASKS.md` (~40 lines)

Do not add anything else. Not the vision, not the architecture, not the audit. An agent that has read
3,900 lines will follow the most recent and most verbose idea rather than the invariant that matters.

### Human set (~900 lines)

`README.md` → `docs/PROJECTS.md` → `docs/PLAN.md` → `docs/TASKS.md`.

That is enough to understand, scope, and run the project.

### Reference set (consult, don't load)

| Document | Answers | Lines |
| --- | --- | --- |
| [`ARCHITECTURE.md`](ARCHITECTURE.md) | how each mechanism works: probes, detectors, attribution math, armed capture | 636 |
| [`PLAN.md`](PLAN.md) | the authoritative scope, budgets, coverage maps, schedule, go/no-go | 464 |
| [`DEPLOYMENT.md`](DEPLOYMENT.md) | host vs container vs Kubernetes, capability matrix, privileges | 416 |
| [`FLEET.md`](FLEET.md) | hub, causal model, roles/edges, privacy, federation | 399 |
| [`AUDIT.md`](AUDIT.md) | why scope was cut; the challenges to our own assumptions; threat model | 318 |
| [`TASKS.md`](TASKS.md) | task ordering and cards | 278 |
| [`VISION.md`](VISION.md) | why this exists, who it is for, what it refuses to be | 261 |
| [`LAUNCH.md`](LAUNCH.md) | GitHub About text, topics, issues, launch copy | 189 |
| [`PRIOR-ART.md`](PRIOR-ART.md) | competitive landscape and the honest gaps | 188 |
| [`LESSONS-STALLWATCH.md`](LESSONS-STALLWATCH.md) | the closest prior art's post-mortem, turned into requirements | 180 |
| [`PROJECTS.md`](PROJECTS.md) | three projects, one contract; what "useful" means for each | 88 |
| [`adr/`](adr/) | decisions that are expensive to reverse | ~215 |
| [`SCHEMA.md`](SCHEMA.md) | the contract | 149 |

---

## Doc hygiene rules

1. **Behaviour changes update the doc in the same PR.** A stale reference doc is a bug, not debt.
2. **Task cards are rolling.** Write the card for task N+1 at the end of task N. Do not pre-write
   twenty cards; they rot.
3. **Never duplicate the schema.** If a doc needs to show a `Finding`, link to `SCHEMA.md` instead.
4. **Disagreement is resolved by rank, not by argument.** Record it in an ADR if it is expensive.
5. **No document is allowed to be aspirational without a phase label.** "Post-1.0" or "exploratory",
   or it does not ship.

---

## Why this structure exists

An agent does not fail because it lacks ambition. It fails because a large document contains two
plausible readings, and it picks one confidently. The defence is not more prose; it is:

- a **short contract** with numbered invariants,
- **machine-checked** versions of those invariants (tests, lints, alloc counters, dep budgets),
- **one task at a time**, each starting from a failing test,
- a **held-out corpus** the agent cannot tune against,
- and a **stop-and-ask** list.

Everything longer than the agent set exists so a human can decide what the agent is *not* allowed to
do yet.
