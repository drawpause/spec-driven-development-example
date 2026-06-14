# Contributing to Relay Specs

These specs are written to be **read and executed by coding agents**, not only humans.
An agent should be able to open any spec and learn exactly which files it governs, what
must always be true, and how to prove the implementation conforms — without guessing.

---

## Guiding Principles

1. **Spec before code.** No feature work begins without an accepted spec.
2. **One source of truth.** If the spec and code disagree, the spec wins. Fix the code, or open a PR to update the spec first.
3. **Machine-checkable over prose.** Prefer a numbered invariant or a runnable command over a paragraph. If a human would have to "use judgement," make the rule explicit.
4. **Addressable.** Every spec declares the file globs it `owns` and maps each invariant to a concrete file/symbol. An agent must never have to search blindly.
5. **Decisions live in DECISIONS.md.** The spec describes what and how; DECISIONS.md records why.

---

## ID Conventions

Stable IDs let agents (and other specs) reference rules precisely. Never renumber; mark removed IDs `(removed)` and add new ones.

| Kind | Format | Example |
|---|---|---|
| Spec | lowercase, matches path under `specs/` | `features/auth` |
| Invariant | `INV-<AREA>-<n>` | `INV-AUTH-3` |
| Acceptance criterion | `AC-<AREA>-<n>` | `AC-AUTH-3` |
| Error code | `SCREAMING_SNAKE` | `EMAIL_NOT_VERIFIED` |

Each acceptance criterion cites the invariant(s) it proves.

---

## Spec Template

Every spec begins with a YAML front-matter block and uses these sections in order.

````markdown
# <Spec Name>

```yaml
spec: features/<name>
status: draft | active | deprecated
last_updated: YYYY-MM-DD
owns:                      # globs this spec is authoritative for
  - src/<area>/**
depends_on:                # other spec ids this relies on
  - api/errors
```

## Summary
One sentence: what this is and why it exists.

## Invariants
Numbered, stable, machine-checkable. Use MUST / MUST NOT (RFC 2119).

- **INV-<AREA>-1:** <a single, testable rule>.

## Contract
The precise interface: request/response schemas, state machines, event shapes.
Deterministic — no "should usually."

## Targets
Where an agent implements each invariant. Globs allowed; symbols when known.

| Invariant | File | Symbol |
|---|---|---|
| INV-<AREA>-1 | `src/<area>/<file>.ts` | `functionName` |

## Acceptance
Given / When / Then, each citing the invariant(s) it proves.

- **AC-<AREA>-1** (INV-<AREA>-1): GIVEN <state> WHEN <action> THEN <observable result>.

## Verify
Commands an agent runs to prove conformance. Must run headless and exit non-zero on failure.

```bash
npm test -- <area>
```
````

---

## Writing or Updating a Spec

1. Create/edit the markdown file in the appropriate `specs/` subdirectory using the template.
2. Keep `owns` globs accurate — they are the contract for which spec governs a file.
3. Add or update invariants with new stable IDs; never reuse a retired ID.
4. Ensure every invariant has at least one acceptance criterion and is reachable by a `Verify` command.
5. Update `last_updated`. For breaking changes (removed/renamed field, changed contract), label the PR `breaking-change` and add a `CHANGELOG.md` entry.

---

## Review Checklist (Agent-Runnable)

- [ ] Front-matter present; `owns` and `depends_on` resolve.
- [ ] Every invariant uses MUST/MUST NOT and is individually testable.
- [ ] Every invariant appears in the `Targets` table.
- [ ] Every acceptance criterion cites at least one invariant ID.
- [ ] `Verify` commands exist and exit non-zero on failure.
- [ ] No prose rule that lacks a corresponding invariant.
- [ ] New error codes are reflected in `specs/api/errors.md`.
- [ ] `CHANGELOG.md` updated for breaking changes.
