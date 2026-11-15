# code-review-raccoon

Unlike the other code rodent, this does not cost you $90 per month per member...

## What is this?

A single self-contained agent skill that sets up CodeRabbit-quality code review in any repository. Point your agent at `SKILL.md`, name a target repo, and it interrogates the repo's real architecture, then writes two files into that repo:

- **`RULES.md`** — repo-specific review rules (OWASP Top 10, trace-before-you-touch, conflict-resolution integrity), pruned and localised to the stack
- **`SKILL.md`** — the review skill that runs against those rules and emits a fixed report template

MIT licensed. Review is fully static — tests and linters are never run before or during setup or review; the fixer offers them only after fixes are applied, and only on your say-so. It also checks business-rule adherence against available acceptance criteria, supplied by you or retrieved with the team's available tools and authorization. If none are available and you decline to provide them, the review proceeds using commit-message intent only and reports that limit; it does not invent acceptance criteria.

Reports end with a **Suggested Improvements** section, placed after the Fix Prompt: things the review noticed that could be done better, anywhere it actually looked. They are separate from findings, never counted in the verdict, never included in the Fix Prompt, and not fixed in a fix round unless you ask for one.

## How to use

1. Clone or point your agent at this repo's `SKILL.md`.
2. Tell it: "Set up code review for `<target repo path>`."
3. It checks for existing review files and stops on conflicts instead of overwriting. It keeps the generated files local, preferring Git's `info/exclude` to changing `.gitignore`, verifies their exclusions, and reports what it wrote.

Trigger a review later with `review` or `!code-review` inside the target repo. Reviews support a branch, PR or explicit commit range. The agent asks you to confirm the base or range every time; for branches it also asks whether to include staged, unstaged and untracked changes. A branch can be reviewed before a PR exists, with no reviewer comments. The report appears in the conversation only, never in a file.
