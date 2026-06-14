# Contributing to Relay Specs

How to write, update, and review specs in this project.

---

## Guiding Principles

1. **Spec before code.** No feature work begins without an accepted spec.
2. **One source of truth.** If the spec and code disagree, the spec wins — fix the code, or open a PR to update the spec first.
3. **Write for a new engineer.** Assume the reader knows software but not this product.
4. **Decisions live in DECISIONS.md.** The spec describes what and how; DECISIONS.md records why.

---

## Writing a New Spec

1. Create a markdown file in the appropriate `specs/` subdirectory.
2. Use the template below.
3. Open a PR. Request review from at least one engineer and one product stakeholder.
4. Merge only after approval. Implementation may begin after merge.

### Spec Template

```markdown
# Feature Name

**Last Updated:** YYYY-MM-DD
**Status:** Draft | Active | Deprecated

---

## Summary
One paragraph. What is this, and why does it exist?

## Goals
- Bullet list of what this spec achieves.

## Non-Goals
- Explicit list of what is out of scope.

## Requirements

### Functional
- [ ] Requirement 1
- [ ] Requirement 2

### Non-Functional
- Performance: ...
- Security: ...

## Behavior
Prose and/or diagrams describing normal flows and edge cases.

## Acceptance Criteria
- [ ] Given X, when Y, then Z.

## Open Questions
- Question (owner: @name, due: YYYY-MM-DD)
```

---

## Updating an Existing Spec

- Open a PR with a clear description of what changed and why.
- If the change is breaking (removes or renames a field, changes an API contract), label the PR `breaking-change` and add an entry to `CHANGELOG.md`.
- Update the `Last Updated` date at the top of the file.

---

## Review Checklist

- [ ] Is the motivation clear?
- [ ] Are edge cases addressed?
- [ ] Are acceptance criteria testable?
- [ ] Does this conflict with any other spec?
- [ ] Is CHANGELOG.md updated (for breaking changes)?
- [ ] For API changes: is `specs/api/errors.md` updated if new error codes are introduced?
