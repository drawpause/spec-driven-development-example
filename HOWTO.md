# HOWTO: Working with These Specs (for Developers New to SDD)

You already know how to ship software. This is a short guide to the one thing that's
different here: **the spec is the source of truth, and a coding agent is the primary
implementer.** Your job shifts from writing every line to *reading specs, steering
agents, and keeping the specs honest*.

If you remember nothing else: **the spec is the contract; code is its output. When they
disagree, fix the code — or change the spec first, on purpose.**

---

## 1. How the specs are organized

Every spec under `specs/` follows the same machine-executable shape (defined in
[CONTRIBUTING.md](CONTRIBUTING.md)). Learn to skim a spec in this order:

| Section | What it tells you | Why you care |
|---|---|---|
| `owns` (front-matter) | The file globs this spec is authoritative for | Tells you *which* spec governs the code you're about to touch |
| `depends_on` (front-matter) | Other specs this one relies on | The reading list you actually need (see §3) |
| **Invariants** (`INV-…`) | Numbered MUST/MUST NOT rules | The non-negotiables. This is the real content |
| **Contract** | Schemas, state machines, event/error shapes | The precise interface |
| **Targets** | Maps each invariant → file/symbol | Where the code lives; no blind searching |
| **Acceptance** (`AC-…`) | Given/When/Then, each citing an invariant | How "correct" is defined |
| **Verify** | Runnable commands | The definition of "done" |

IDs are stable and cross-referenced: `AC-AUTH-2` proves `INV-AUTH-3`; `ops/security`
cites `features/auth` invariants. You can follow these like links.

**Reading tip:** to understand a behavior, read its **Invariants** and **Acceptance**
first. Skip the prose. If a rule isn't an invariant, it isn't binding.

---

## 2. The map is the entry point — start at `overview` and the spec table

Don't read the repo top to bottom. Two starting points:

- [`README.md`](README.md) → the **Spec Map** table (what each spec is, by id).
- [`specs/overview.md`](specs/overview.md) → the **domain model**, **roles**, and
  **global invariants** (`INV-CORE-*`) that bind every other spec.

From there, jump straight to the spec whose `owns` globs match the files you care about.

---

## 3. Should you (or the agent) ingest the *whole* SDD at session start?

**No — and that's the entire point of the format.** Dumping all 16 specs into an agent's
context is the most common mistake. It's expensive, it buries the relevant rules in
noise, and it makes the agent more likely to "improve" things you didn't ask about.

The specs are **addressable** precisely so you can load a small, exact slice. Use this
loading recipe instead:

1. **Always load (small + global):**
   - `specs/overview.md` — domain model + `INV-CORE-*`
   - `CONTRIBUTING.md` — the spec format, so the agent edits specs correctly
2. **Load the target spec:** the one whose `owns` matches your feature
   (e.g. `specs/features/payments.md`).
3. **Load its dependency closure:** follow `depends_on` one or two hops.
   `payments` → `api/rest`, `api/errors`, `data/schema`, `features/notifications`.
4. **Stop there.** If the agent needs more, it will tell you (or cite a missing id).

> Rule of thumb: load `overview` + `CONTRIBUTING` + the target spec + its direct
> `depends_on`. That's usually 3–6 files, not 16.

**When *is* full ingestion useful?** Rarely, and deliberately:
- A **cross-cutting audit** ("check every spec's invariants are covered by a Verify
  command", "find contradictory rules across specs").
- A **global refactor of the spec format itself** (like the rewrite that produced these).
- Generating a **dependency/coverage report** across all specs.

For feature work, narrow beats broad every time.

---

## 4. Steering an agent to implement a feature

Point the agent at the spec, not at a vague description. A good kickoff prompt names the
spec id and the invariants in scope:

```
Implement features/notifications.

Load: specs/overview.md, CONTRIBUTING.md, specs/features/notifications.md, and its
depends_on (api/rest, api/websockets, data/schema).

Scope: INV-NOTIF-1 through INV-NOTIF-4 only. Implement at the files in the Targets
table. Do not touch billing or auth.

Done = the Verify commands in the spec pass. Show me the output.
```

Why this works:
- **Bounded context** — it tells the agent exactly what to read (§3).
- **Bounded scope** — invariant IDs are a precise fence. "Implement notifications" is
  vague; "satisfy `INV-NOTIF-1..4`" is testable.
- **Objective done-condition** — the `Verify` block, not the agent's opinion.

Good follow-up moves:
- "Map each change back to the invariant it satisfies." (catches scope creep)
- "Which acceptance criteria now pass? Run them." (`AC-…` are your test plan)
- "You modified `src/billing/*` — that's outside `features/notifications.owns`. Revert
  it or tell me which spec authorizes it." (the `owns` globs are your guardrail)

**Reviewing the agent's work:** check it against the spec, not just "does it look right."
Did every in-scope invariant get a Target change? Do the `Verify` commands pass? Did it
stay inside the spec's `owns`? If the diff touches files no in-scope spec owns, that's a
red flag — either the scope was wrong or a spec needs updating.

---

## 5. How and when to update the specs

The specs are living. Code that drifts from the spec is a bug; a *deliberate* behavior
change means **the spec changes first.** Decide which case you're in:

**Case A — Code is wrong, spec is right.** Fix the code. Don't touch the spec.

**Case B — The behavior should change (new requirement, better design).**
Update the spec *first*, then implement. Order matters: the spec is the brief.

When you (or an agent) edit a spec:

1. **Add/parameterize an invariant** rather than burying a rule in prose. New rule → new
   stable `INV-<AREA>-<n>`. Never reuse a retired ID; mark removed ones `(removed)`.
2. **Wire it up:** every invariant needs a row in **Targets** and at least one
   **Acceptance** criterion, and must be reachable by a **Verify** command. The review
   checklist in `CONTRIBUTING.md` enforces this.
3. **Keep `owns`/`depends_on` accurate.** If a rule now governs new files, widen `owns`.
4. **Bump `last_updated`.** For breaking changes (removed/renamed field, changed
   contract), label the PR `breaking-change` and add a `CHANGELOG.md` entry.
5. **Record the *why* in `DECISIONS.md`**, not the spec. Specs say *what/how*; ADRs say
   *why we chose this*. Append a new ADR; never edit an accepted one (supersede it).

**Telling an agent to update a spec** — make the spec edit an explicit, separate step:

```
We're changing behavior, so update the spec first.

In specs/features/auth.md: add an invariant that access tokens are revoked on
password change (not just reset). Give it a new INV-AUTH id, add it to the Targets
table and an AC-AUTH acceptance criterion, bump last_updated, and add a CHANGELOG
entry (this is breaking). Then stop — show me the spec diff before writing any code.
```

Then, in a second turn, "now implement it." Reviewing the *spec* diff before the *code*
diff is the single highest-leverage habit in SDD: it's cheaper to correct intent in a
markdown table than in a merged PR.

**When NOT to update a spec:**
- A pure bug fix that makes code match the existing spec (Case A).
- Implementation details with no observable contract (internal helper names, file
  splits) — unless they change what a `Targets` entry points to.
- One-off experiments. Land the spec change only when the behavior is real.

---

## 6. A 5-minute starting loop

1. Read `specs/overview.md` (domain + `INV-CORE-*`) and the README spec map.
2. Find the spec whose `owns` matches your task; read its Invariants + Acceptance.
3. Decide: am I fixing code to match the spec (Case A) or changing behavior (Case B)?
   If B, edit the spec first and get the spec diff reviewed.
4. Hand the agent: `overview` + `CONTRIBUTING` + target spec + its `depends_on`, with
   invariant IDs as the scope and the `Verify` block as the done-condition.
5. Review the diff *against the invariants and `owns`*, then run `Verify`.

That's SDD here: small, addressed context in; invariant-scoped, verifiable changes out.
