# Implement phase

Argument: `<issue>`, optionally a list of blocking review findings (fix-mode, see step 4).

Implement the issue on its own branch and open (or update) a draft PR. Two modes, decided by whether blocking review findings were passed in.

## 0. Inputs

- `gh issue view <issue> --comments` — the issue plus the most recent `<!-- agent:research -->` comment. No research comment → stop and report a failure summary "no research found, explore phase must run first".
- The most recent `<!-- agent:design -->` comment — its approach and decisions are what you build. No design comment → stop and report "no design found, design phase must run first".
- If the repo already has established conventions (an existing stack, test setup, lint config), follow them. If this is the first code landing in the relevant area, the design brief's approach is the convention going forward — don't invent a second stack alongside an existing one.

## 1. Branch

Branch name: `agent/issue-<n>-<slug>` (slug from the issue title, kebab-case, ≤5 words). If the branch already exists locally or on the remote, check it out and continue on it — never create a duplicate.

## 2. Implement

Follow the design's approach. Honor its decisions rather than re-deciding them. Run typecheck/lint/tests as the repo provides them, as you go; run the full suite once before pushing. Every behavior change gets a test — no exceptions for "small" or "obvious" changes. Do not open or update the PR if the suite doesn't pass; fix it first, or if a failure is pre-existing and unrelated, say so explicitly in the PR summary rather than silently pushing past it. If the suite can't be made green, or the design's approach turns out to be blocked by something only a human can resolve, stop, comment on the issue explaining what's blocking it, and report `BLOCKED` (see Outcome) instead of opening a PR.

If mid-implementation the design's approach turns out to be wrong but still buildable, don't silently diverge: note the divergence and the reason in the PR summary.

## 3. Draft PR

Push the branch. If no open PR exists for it, create one:

- `gh pr create --draft --title "<issue title> (#<n>)"`
- Body: `Closes #<n>`, a link to the design comment, and a summary of the changes — what changed, why, how it was tested, any divergence from the brief.

If a PR already exists, push and add a comment summarizing what this iteration changed.

## 4. Fix-mode (invoked with blocking findings)

1. Address **every** blocking finding — fix it, or if you believe it's wrong, reply to the inline comment explaining why (with evidence); never silently skip one.
2. Reply to each addressed inline comment with a one-liner: what you changed.
3. Suggestions are optional — apply the cheap ones, skip the rest without comment-spam.
4. Run the full suite again, push, comment on the PR summarizing which findings were addressed.

## Outcome

Report the PR number/URL, a one-line summary of what changed, and test results — which tests were added, and confirmation the full suite passed. If something fails, say so honestly. End your final message with these trailer lines, verbatim:

If a PR was opened/updated (suite passing):

```
AGENT_PR: <pr number>
AGENT_OUTCOME: READY
```

If blocked before a PR was opened (suite couldn't be made green, or blocked on a human decision) — no `AGENT_PR` line, there's no PR to point at:

```
AGENT_OUTCOME: BLOCKED
```
