# Review phase

Argument: `<pr>`, the number of the PR being reviewed.

Review a PR and render a verdict on the PR itself.

## 1. Determine the iteration

No stored counter — count it from the PR's own history: `gh pr view <pr> --json reviews -q '[.reviews[] | select(.state == "CHANGES_REQUESTED")] | length'`. This run's iteration is that count plus 1. The cap is 3; if this iteration exceeds the cap, still run the review (don't skip it) but say so plainly in the outcome so the caller can route to a human instead of looping back to implement.

## 2. Inputs

- `gh pr view <pr>` + `gh pr diff <pr>` — the diff is the review surface.
- The linked issue, its `<!-- agent:research -->` comment, and its `<!-- agent:design -->` comment — review against what was actually asked and designed, not against taste.
- Existing review threads on the PR — don't re-raise a finding already addressed or already answered with a justified pushback.

## 3. Review

Two axes:

- **Spec** — does the change do what the issue asked and the design brief described? Flag scope creep and missed requirements alike.
- **Standards** — does it fit the repo's existing conventions (or, if this is the first code in the area, is it internally consistent and reasonable)? Nothing that looks unsafe (injection, secrets, obviously missing validation at a real trust boundary)?

**Tests, checked explicitly, every review**: every behavior change in the diff has a test covering it, and the implement phase's reported suite run actually passed (don't take "tests pass" on faith — re-run the suite yourself if the repo makes that cheap). A behavior change with no test, or a claimed-passing suite that doesn't actually pass, is `[blocking]` by default — not a judgment call like the rest of Standards.

Severity: `[blocking]` only for something that must change before merge — verify each one has a concrete failure scenario (specific input/state → wrong output or crash) before posting it; drop anything you can't substantiate. Everything else is `[suggestion]`. When in doubt, suggestion.

## 4. Post the verdict

**Blocking findings exist** — post one PR review (event `COMMENT`, via `gh api repos/{owner}/{repo}/pulls/<pr>/reviews`) with inline comments for each finding, plus a body summarizing them. PR stays draft.

**Zero blocking findings** — post suggestions (if any) the same way, then a summary comment:

```
<!-- agent:verdict -->
Approved after <iteration> iteration(s).
```

Never flip the PR to ready and never approve it (`gh pr review --approve` — GitHub rejects self-approval anyway) — leave it in draft. The comment above is the signal; flipping draft→ready is a human act, always.

## Outcome

Report `APPROVED` or `CHANGES_REQUESTED`, the iteration number and whether it's at or past the cap, plus the list of blocking findings (file:line + one-liner each). End your final message with these trailer lines, verbatim:

```
AGENT_PR: <pr>
AGENT_OUTCOME: APPROVED
```

(substitute `CHANGES_REQUESTED`, or — if this iteration is at or past the cap — `CHANGES_REQUESTED_CAPPED`, for `APPROVED`)
