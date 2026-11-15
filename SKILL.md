---
name: code-review-raccoon
description: Set up CodeRabbit-quality code review in any repo. Point this skill at a target repo and it interrogates the architecture, then writes a repo-specific RULES.md and a review SKILL.md into that repo. Trigger: code-raccoon, review, code review.
---

# code-review-raccoon — setup skill

You are setting up code review in a target repository. The goal is CodeRabbit-level review quality without the price: rules that reference that repo's real architecture, not platitudes. Everything you need is in this file — it is fully self-contained; do not load any external prompt, vault, or path.

Division of labour: this skill generates → `RULES.md` stores the rules (written into the target repo) → the repo's own review `SKILL.md` runs the review and emits the fixed report in the conversation, never to a file. Plain discussion sessions load neither.

## Input contract (do this FIRST)

- The target repo path must be given in the request. If it was not given, ask once: "Which repo should I set up code review for?" Never guess a repo path.
- Confirm the path exists and is a git repository before doing anything else.

## Step 0 — Conflict guards (stop and resolve with the user)

Check the target repo for BOTH of the following:

1. **A review skill already exists** — a `SKILL.md`, `skills/code-review/`, or similar carrying code-review procedure.
2. **A rules file already exists** — `RULES.md`, or an `AGENTS.md` that already carries code-review rules.

If either exists: **STOP. Do not touch it.** Report to the user and resolve together:

- "Review skill exists at `<path>` — covers: `<one-line summary>`."
- "RULES.md exists at `<path>` — covers: `<one-line summary>`."

Then ask the user how to proceed: leave it alone, revise the existing file (explicit request only), or abort. Never overwrite, never merge, never rewrite an existing file. An existing file is the owner's; your job was to create, not to replace. If a skill-name conflict with another installed review skill surfaces later (trigger clash), the same rule applies: stop and resolve with the user.

If neither exists: proceed.

## Step 1 — Interrogate the repo before writing anything

Read the codebase and answer, with evidence:

- Language(s) and version(s); framework(s)
- Dependency injection / wiring mechanism, if any (constructor injection, decorators, service locator, none)
- Module boundaries: what imports what; any circular imports
- Test runner(s) and how tests are invoked; are there any tests at all? — **discovery only. Tests and linters are NEVER executed during setup or review.** Record the command; running it is the fixer's post-fix offer, not your job.
- Existing lint/format/CI config (eslint, clippy, prettier, CI workflows)
- Any existing AGENTS.md, CONTRIBUTING.md, .cursorrules, or similar — note them, but do NOT put review rules in them; RULES.md is the dedicated rules file
- Call-graph shape: are deep chained calls (A() -> B() -> C() -> D()) common? Long if/else chains? Prop drilling?
- Merge shape: multi-branch / multi-agent workflows? Do conflict resolutions appear in recent history (`git log --merges`)? — this decides how prominent the conflict-audit rule must be
- Branch topology: inspect branches and upstreams for context. Never bake a base into generated files; the owner confirms the base or starting point for each review, regardless of topology.

If a Step-1 finding is uncertain, record it as an open question in the file rather than guessing.

## Step 2 — Write RULES.md

Location: target repo root, always named RULES.md. Structure it in three tiers — every rule carries a one-line WHY, because rules without rationale get cargo-culted:

### Tier 1 — Hard Rules (merge blockers)

SECURITY — list the OWASP Top 10 explicitly, one line each, phrased for this stack. Example: "A03 Injection: parameterized queries only — never string-built SQL/NoSQL/command strings." Do not summarise the OWASP list; name all ten.

Plus:
- No secrets in code or config — env vars or a secret manager, never committed
- Least privilege by default — every new endpoint/permission starts at minimum
- Dependency hygiene — minimal, pinned, license/security-checked before adding
- No swallowed errors — empty catch blocks are violations; handle, log, or propagate
- Fail fast, fail loud — validate at system boundaries, not ten frames deep
- No dead code, no commented-out code — git history is the archive

TRACE BEFORE YOU TOUCH (the core rule — make it prominent):
- Signature/constructor changes → find all callers and all DI/wiring bindings; update every call site in the same diff. An updated signature with stale callers is a broken build, not a review comment.
- Business-rule changes → find every consumer of the old rule; a changed rule in one place that contradicts a callee is a defect, not a refactor.
- Trace the line, or raise it. If you cannot trace the full dependency chain with confidence, say so explicitly — "unverified: callers of X not traced" — rather than passing silently. A flagged uncertainty is a pass; an unnoticed break is not.

CONFLICT RESOLUTION INTEGRITY (multi-agent rule — a different agent will audit this):
- Every resolved conflict must preserve BOTH parents' intent, or consciously drop one side WITH a stated reason. Compare the resolution against each parent (`git show <parent>:<file>`) — never against memory.
- Violations, all merge blockers: picking a parent wholesale without comparing intent; a parent's change silently vanishing; lines deleted just to clear conflict markers; "resolved" claimed but never verified against both parents; a resolution that compiles clean but contradicts one parent's intent.
- WHY: a resolution that only looks right is how silent regressions enter history. A clean diff is not evidence of a correct resolution — the agent that resolved may have cut corners to get a clean read.

### Tier 2 — Guidelines (prefer; exceptions allowed with a comment)

- DRY, SOLID, ACID, separation of concerns — name the specific violations you see in THIS repo, not the theory
- Zen of Python, adapted (never verbatim — it is a joke list): readability counts; errors should never pass silently; there should be one obvious way to do it; now beats never; if the implementation is hard to explain, redesign it
- Intention-revealing names — no `data2`, no `handleStuff()`; consistent vocabulary across the codebase
- Comments explain WHY, not WHAT — if a comment narrates what the code does, rename instead
- Every change ships with tests; test the failure path, not just the happy path; no flaky tests
- Small commits, conventional messages — a diff should be reviewable in one sitting

### Tier 3 — Banned Patterns (zero exceptions)

- Long if/else chains where a switch/match works — guarded by the max-nesting rule below
- Max nesting depth: 2 — guard clauses + early return + switch, mechanically enforced
- Prop drilling — no passing props/state through N components/modules to reach a leaf; restructure (context, module boundary, param object)
- Deep call pyramids — A() -> B() -> C() -> D() and back again; flatten, invert, or redesign
- Shared mutable state — root cause of most concurrency pain
- Circular imports between modules — dependency direction is one-way
- Reaching into another module's internals — modules expose interfaces, not guts

## Step 3 — Prune and localise

- Delete any rule that does not apply to this repo's stack (e.g. "prop drilling" if there is no UI layer) — dead rules teach agents to ignore the file
- Replace generic paths with the repo's real ones: name the actual DI wiring file, the actual test command, the actual modules with known boundaries
- Where Step 1 found an existing config conflict, note it in an "Open questions" section instead of silently overriding

## Step 4 — Keep the generated files out of git

Before either generated file is created (including Step 2), carry out the destination and exclusion checks below. The step numbering does not grant permission to write first.

RULES.md and the review SKILL.md (Step 5) are local-only agent tooling: never stage, commit or touch the index for them. Before writing, check each exact destination is not already tracked or occupied; if it is, stop and resolve with the owner. Do not hide or overwrite a tracked file.

Prefer the repo's local Git exclude file (`git rev-parse --git-path info/exclude`) instead of changing `.gitignore`. Preserve existing entries and append exact, anchored, properly escaped repo-root-relative patterns for the generated paths only when needed. A Git exclude file can be shared between worktrees; report its actual path. For each path, verify the effective exclusion with `git check-ignore -v -- <path>` (ensure the matching rule is not a negation) and confirm it is not tracked with `git ls-files --error-unmatch -- <path>` (exit 1 means untracked; other errors are not proof). After writing, verify neither appears in `git status --porcelain --untracked-files=all`. Absence from status alone does not prove a file is ignored. If another rule overrides the exclusion, or Git configuration or paths prevent safe verification, stop and report what could not be verified; do not silently edit `.gitignore` or claim success. Report any other visible setup changes honestly; do not claim the entire working tree is clean.

## Step 5 — Write the target repo's review SKILL.md

Write a second file alongside RULES.md: the review skill for THAT repo. It must be self-contained — no references back to any vault, prompt, or external path. Exact content to write:

````markdown
---
name: code-review
description: Review changes in this repo against its RULES.md. Trigger: `review` or `!code-review`.
---

# Code Review (this repo)

You are doing code review work in <repo>. RULES.md at the repo root is the authority — you supply the report form, it supplies the rules.

## Procedure

1. **Confirm scope, every review** — support a branch (including before a PR exists), a PR, or an explicit commit range. Ask the owner to confirm the base branch for a branch or PR, or the starting and ending commits and inclusion semantics for a range, even if the request names them or topology seems obvious. Do not infer or default the base. If confirmation or required refs are unavailable, stop and ask; do not silently change scope. For a branch review, also ask whether staged and unstaged working-tree changes and untracked files are included; review each confirmed category and report any that cannot be inspected. A PR or reviewer comments are not required for a branch review.
2. **Build the comparison** — for a confirmed branch or PR, compare its confirmed head against the merge-base with its confirmed base, e.g. `git diff "$(git merge-base <base> <head>)" <head>`. For a branch, inspect confirmed staged changes with `git diff --cached`, unstaged changes with `git diff`, and confirmed untracked files separately; do not treat these commands as interchangeable or include an unconfirmed category. Working-tree changes belong to the checked-out worktree: if its HEAD does not match the confirmed branch head, stop and ask instead of silently mixing scopes or checking out a branch. For a commit range, honor the confirmed endpoints and inclusion semantics rather than replacing them with a merge-base. If a PR or its refs are unavailable, report the limit and ask for an accessible scope; do not pretend to have reviewed it.
3. **Locate rules and evidence** — take the repo path from the request, or find `RULES.md` at the repo root. If it is missing, stop and reply "No RULES.md in this repo — run the code-review-raccoon setup skill first." Read RULES.md in full, the confirmed diff, and other material necessary to trace callers and consumers, inspect merge parents, and verify ticket intent. Never review from memory; do not claim evidence you could not inspect.
4. **Review against the rules** — every finding names its rule in the Rule column, cites `File:Line` as a range, and carries one line of evidence. No rule, no finding: a ruleless complaint goes to Open Questions. Acceptance-criterion findings may cite the criterion as their rule.
5. **Dependency trace** — prove trace-before-you-touch ran: signatures changed → callers found/updated; business-rule changes → consumers checked. Anything not verified goes in the ⚠️ Unverified line explicitly — silence is not an option.
6. **Conflict audit** — for every merge/commit that resolved conflicts, re-examine each conflicted region against BOTH parents (`git show <parent>:<file>`). Assume the resolver may have cut corners: a clean diff proves nothing. Flag as suspect — weird, not quite right, or straight-up bullshit: one parent's changes vanished without a stated reason; a side taken wholesale without comparing intent; lines deleted just to clear markers; resolution claimed but never verified; result compiles but contradicts a parent's intent. Every suspect resolution is a finding citing the Conflict resolution integrity rule. Regions you could not compare go under ⚠️ Unverified — never silently pass them.
7. **Business-rule adherence (BR pass)** — intent gathering is provider-agnostic: use the team's available forge, tracker, tools and authorization; do not assume a specific provider or credential. Always read commit messages for the confirmed review scope as context. Use ticket or acceptance-criteria content already supplied or available; otherwise, when a ticket can be retrieved using available tools and authorization, retrieve it. If no ticket or criteria are available, ask once: "Want to provide the ticket or acceptance criteria?" Declining is valid; proceed using commit-message intent only and report that limit. Compare implementation against stated acceptance criteria, citing evidence for unmet criteria; passing tests alone does not prove acceptance. Do not invent criteria or claim full acceptance-criteria coverage when none were available. Number these findings `BR-1`, `BR-2`, …, quote the criterion in the Rule column, and identify the intent source in the report.
8. **Report** — emit the template below in the conversation only, never to a file. Sort findings by severity and number each ID series in that order.
9. **Fix round** — when a report is handed back:
   - First act: re-state your understanding of each finding (one line each) before editing anything.
   - **Approved steps are exhaustive.** If a prescribed edit turns out wrong or impossible (applied migration, moved file, step already done), or the fix shape is ambiguous (index vs. query change), STOP before creating or editing anything: report why the step fails, propose options, wait for approval. Never substitute your own mechanism, however reasonable — improvisation outside the approved file table is exactly what this rule prevents. Stopping to ask is always allowed; proceeding on an unapproved deviation is not.
   - Work findings by ID, re-trace anything you changed, reply per-finding: FIXED / SKIPPED+why. When every finding is handled, OFFER — never auto-run — tests and lint: ask the user whether to run <test command> and the linters; run nothing until they say yes.
   - **Open Questions belong to the owner** — do not act on them, do not fix them opportunistically while doing other findings.
   - **Exclusions are the owner's call** — the fix prompt's "Do not touch" list may only hold owner-confirmed items; anything else you would skip gets surfaced as "Exclusions proposed — your call" in your reply, never silently dropped.
   - Do not expand scope beyond the listed findings.

## Report template

```markdown
# Code Review Report — <repo> · <branch/PR/commit-range>
Reviewed: <date> · Rules: RULES.md <version/commit>
Scope: <confirmed head and base, or confirmed range and endpoint semantics; included/excluded working-tree categories>
Intent context: <ticket and source | pasted acceptance criteria | commit messages only>

## Verdict: BLOCK / REQUEST CHANGES / APPROVE
🔴 X blockers · 🟠 Y majors · 🔵 Z minors · ⚪ N nits

## Findings
| ID | File:Line | Sev | Rule | Why (evidence) |
|----|-----------|-----|------|----------------|
| F1 | src/auth.py:88-94 | 🔴 | OWASP A01 | checkAdmin() skipped on DELETE route |
| F2 | src/api.ts:12-12 | 🟠 | Trace-before-you-touch | Signature changed; 2 callers stale |
| BR-1 | src/billing/refund.py:51-58 | 🟠 | AC: refunds within 30 days | Code caps refunds at 14 days; AC says 30 |

Severity scale (defined once, used every review):
🔴 Blocker — merge blocked (security, broken build, data loss)
🟠 Major — fix before merge (logic bug, missed trace, untested failure path)
🔵 Minor — fix soon (naming, dead code, nesting)
⚪ Nit — optional (style, wording)

Exclusions proposed — your call: <F# or BR-# (<reason>)> — the reviewer proposes, you dispose; confirmed exclusions become "Do not touch" items on the next round.

## Dependency Trace
- Signatures changed: → callers found: N, updated: N ✓
- Business-rule changes: → consumers checked: …
- ⚠️ Unverified: <explicit list — "callers of X not traced" is allowed here, silence is not>

## Open Questions
- <things needing your decision, not code fixes>

## Fix Prompt (copy-paste)
> In <repo>, fix findings F1, F2, BR-1 from the code review report in this conversation.
> If the report is not available in this conversation, ask me to paste it; never
> invent findings or assume a report file exists.
> Re-state your understanding of each finding before editing. If a fix shape is
> ambiguous or a prescribed step is impossible, STOP — report why, propose
> options, wait for approval; never substitute your own mechanism.
> Per finding: apply fix, then RE-TRACE callers of anything you changed,
> reply per-finding: FIXED / SKIPPED+why, then ask whether to run <test
> command> and lint — never run them without an explicit yes.
> Do not touch F3 (owner-confirmed accepted risk). Open Questions are for the
> owner — do not act on them. Do not expand scope beyond findings.
```

Filling rules:
- `File:Line` must be a line range — a path without lines forces the fixer to re-search. If a missing implementation has no precise changed line, cite the nearest relevant existing code and explain the absence; if no relevant location can be identified, record it as unverified instead of inventing a location.
- A finding that can't name its rule is an Open Question, not a finding.
- "Why" is one line with evidence; paragraphs go to Open Questions or split into two findings.
- Wontfix/accepted-risk findings stay in the report, struck or excluded via the Fix Prompt — never silently deleted.
- Exclusion provenance: the fix prompt's "Do not touch" list may only contain owner-confirmed exclusions. Everything else the reviewer would skip is surfaced as `Exclusions proposed — your call: F4 (reason)` — the reviewer proposes, the owner disposes; confirmed exclusions become "Do not touch" items on the next round.
- The fix prompt always carries these guards: re-trace, test-offer (ask, never auto-run), scope fence, stop-before-impossible-or-ambiguous, Open Questions fence.

## Guards
- Review is read-only against the repo. Fixes happen only when a Fix Prompt is handed back.
- Tests and linters never run before or during review. The fixer offers them only after fixes are applied; the user decides.
- Stopping to ask is always allowed — ambiguity, an impossible step, an unexpected discovery mid-fix: report and wait. Proceeding on an unapproved deviation is never allowed.
- Never modify RULES.md — it is the owner's. A rules change is a separate, explicit request.
- Never commit or `git add` RULES.md or this skill; both stay git-ignored.
- The report goes to the conversation only, never to a file.
````

Replace `<repo>` and `<test command>` with the target repo's real name and actual test command during Step 3 localisation. Do not bake any branch name into this file — scope and base or range are confirmed per review (Procedure step 1). If the target repo already has a review skill (Step 0 guard), do not write this file — stop and resolve with the user as Step 0 prescribes.

## Step 6 — Wire the loading

Both generated files load only during review work, via the target repo's SKILL.md (Step 5): the skill takes the repo path, asks the owner to confirm the review scope and base or range each time, reads RULES.md, runs the review, and emits the report template it carries itself in the conversation. They MUST NOT be injected into plain discussion sessions. Confirm in your reply how a chat-only session avoids paying for it.

## Report back

Report to the user:

- Both file paths (RULES.md and the review SKILL.md)
- How many rules survived Step 3 pruning
- Any open questions from Step 1
- Effective ignore and tracking checks for each file, the Git exclude file path, status evidence, and any other visible setup changes; report any verification that failed
- If Step 0 hit an existing file or a skill conflict — your stop-and-resolve report instead of a rewrite
- Confirmation: no tests or linters were run during setup; test command recorded as `<command>` for the fixer's post-fix offer only
