# code-review-raccoon

Unlike the other code rodent, this does not cost you $90 per month per member...

## What is this?

A self-contained agent skill that sets up evidence-backed code review: tracing changes through their callers, checking security and business rules, and backing findings with evidence. Point your agent at `SKILL.md`, name a target repo, and it interrogates the repo's real architecture and teaches your agent to produce two outputs:

- **`RULES.md`** — written into the target repo's root (the one repo-specific artifact): review rules (OWASP Top 10, trace-before-you-touch, conflict-resolution integrity, idempotency, TOCTOU, observability), pruned and localised to the stack
- **A global `code-review` skill** — built by the agent in its own skill system, in its own format. Nothing is copied from this file: the agent's skills live wherever its architecture puts them, and we can't cater to every agent's skill format, so this SKILL.md teaches what to write instead of shipping a fixed file. Installed once, it reviews any repo against that repo's `RULES.md`.

MIT licensed. Review is fully static — tests and linters are never run before or during setup or review; the fixer offers them only after fixes are applied, and only on your say-so. It also checks business-rule adherence against available acceptance criteria, supplied by you or retrieved with the team's available tools and authorization. If none are available and you decline to provide them, the review proceeds using commit-message intent only and reports that limit; it does not invent acceptance criteria.

Reports end with a **Suggested Improvements** section, placed after the Fix Prompt: things the review noticed that could be done better, anywhere it actually looked. They are separate from findings, never counted in the verdict, never included in the Fix Prompt, and not fixed in a fix round unless you ask for one.

## How to use

1. Clone or point your agent at this repo's `SKILL.md`.
2. Tell it: "Set up code review for `<target repo path>`."
3. It checks for existing rules and skills (it never overwrites; a rules conflict stops and resolves with you), writes `RULES.md` into the target repo — keeping it local, preferring Git's `info/exclude` to changing `.gitignore`, verifying its exclusion — and builds its global `code-review` skill in its own skill system, reporting where it installed it.

Trigger a review later with `review` or `!code-review` from anywhere — the skill is global and takes the repo path per request. Reviews support a branch, PR or explicit commit range. The agent asks you to confirm the base or range every time; for branches it also asks whether to include staged, unstaged and untracked changes. A branch can be reviewed before a PR exists, with no reviewer comments. The report appears in the conversation only, never in a file.
