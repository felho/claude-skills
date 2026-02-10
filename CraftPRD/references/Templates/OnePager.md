# One-Pager PRD Template

Use this template when creating a new one-pager PRD via the Draft workflow.

---

```markdown
# <Product/Feature Name> — One-Pager PRD

**Date:** <ISO date>
**Author:** <name>
**Status:** Draft | In Review | Approved

## Problem

<2-3 sentences describing the problem. Who has it? How painful is it? What happens if we don't solve it?>

## Solution Overview

<1 paragraph. What's the core insight or approach? What does the user experience look like?>

## Key Decisions

<Bulleted list of decisions already made, with brief rationale.>

- **<Decision>** — <rationale>
- **<Decision>** — <rationale>

## Scope

### In Scope (V1)
- <capability or feature>
- <capability or feature>

### Out of Scope
- <explicitly excluded item> — <why>
- <explicitly excluded item> — <why>

## Success Criteria

- <measurable criterion>
- <measurable criterion>

## Key Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| <risk> | High/Med/Low | High/Med/Low | <mitigation> |

## Open Questions

- <question that needs resolution before or during implementation>
- <question>

## References

- <link to exploration notes, if any>
- <link to related docs>
```

---

## Usage Notes

- **Keep it short.** If you can't fit it in ~1 page (~60-80 lines of content), you're including too much detail. Save technical depth for the Deepen phase.
- **No code blocks.** If you feel the need for TypeScript interfaces or SQL schemas, note them as "needs technical detail" in Open Questions.
- **Decisions are valuable.** Even early-stage decisions (e.g., "SQLite, not Postgres") are worth capturing with rationale. They prevent re-litigating later.
- **Out of scope is as important as in scope.** Being explicit about what's NOT included prevents scope creep and sets expectations.
