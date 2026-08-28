# Design phase

Argument: `<issue>`, the number of the issue being worked.

Turn the research into a plan, autonomously. This is where implementation decisions get made — not deferred to a human unless the decision is genuinely theirs to make. Change no code directly.

## 1. Gather context

- `gh issue view <issue> --comments` — the issue and the most recent `<!-- agent:research -->` comment. No research comment found → stop and report `BLOCKED` with summary "no research found, explore phase must run first".
- Read `CONTEXT.md` and `docs/adr/` at the repo root — the approach must fit this project's existing vocabulary and decisions, not invent parallel ones.
- Read code as deep as the approach requires — research established what exists; design needs enough to judge the best approach.

## 2. Design

Favor the simplest approach that fits the existing domain model and ADRs over a more general one. Where two approaches are genuinely close, prefer the one that touches less of the codebase.

Consider alternatives briefly before settling — not an exhaustive survey, just enough to know the chosen approach isn't the first idea reached for without checking.

## 3. Post the design brief

```
<!-- agent:design -->
## Design

**Approach:** ...
**Key decisions:** ...           (decision → choice → why, one line each)
**Alternatives considered:** ... (what was rejected and why — omit if there was only one reasonable approach)
**Out of scope:** ...            (explicitly not doing, so implement doesn't scope-creep)
**Risks:** ...                   (what could go wrong, and how to mitigate)
```

The latest `<!-- agent:design -->` comment supersedes any earlier one on the same issue.

## 4. Block (judgment calls only)

If, after investigating, a decision is still open because it's a **product or business call this project can't answer from its own docs and history** — not a technical detail reasonably decidable from context — then:

1. Post a comment explaining exactly what's undecided and why it isn't a call to make alone.
2. Report `BLOCKED` as the outcome and stop.

Asking a human is the last resort, not the first: a costly technical call still gets decided and documented, not deferred.

## Outcome

Report `READY` or `BLOCKED` plus a one-line summary.
