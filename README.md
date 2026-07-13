# aki-skills

Claude Code skills I use in my own workflow.

## Skills

### `/handoff`

When a session grows too long to continue effectively, generate a structured handoff document so the next agent can pick up exactly where you left off — with file paths, decisions made, and a concrete first step, not a vague summary.

### `/read-handoff`

The other half of `/handoff`: start a new session by reading a handoff document and figuring out where to actually pick up. Because a handoff is a snapshot that goes stale, this skill cross-checks every claim against the current git state — what shipped since the handoff's date, whether its "to-do" items were already done by later commits, whether HEAD has moved past it — and against the same-period spec/plan docs it references. It then reports the true pick-up point (handoff says X, git shows Y, here's what's actually left) and asks you what to do next. Read-only: it never touches git or code on its own.

Usage: `/read-handoff path/to/handoff.md` (omit the path to pick the latest)

### `/implement-plan`

Given a structured implementation plan (the construction blueprint — ordered tasks with complete code), create a worktree off the current branch, execute each task test-first, then push and open a PR. The plan is the authority; it references a design spec (`对应 spec:` field) which this skill skims for background. Tasks encode the tdd red-green-refactor cycle; failures trigger a diagnosis loop; a code-review dual-axis self-check runs before the PR.

Usage: `/implement-plan path/to/plan.md`

### `/loop-pr-merge`

Stand up a self-paced loop that watches a repo for new PRs and reviews-then-merges each one against *that repo's own* development and merge rules. Reads `CLAUDE.md` first, recovers each PR's spec + diagnosis for project context, then reviews with risk-scaled rigor (stale-base 3-way merge, protected-file byte checks, import-ring, red→green, neighborhood seesaw, sync discipline). On a clean pass it squash-merges; on a failure it either fixes the PR in place and pushes (when the branch is a local worktree you own) or posts a directed changes-requested review. Distilled from a real session that merged ~25 consecutive PRs.

Usage: `/loop-pr-merge [repo-path]`

### `/doc-sync`

Flow the facts and decisions that already happened (recorded in handoffs + new plans/specs) back into the project-level docs — `CLAUDE.md`, `docs/milestone/README.md` + each milestone, `docs/adr/` — and **delete or collapse the stale content while doing it**. The first principle is drift correction, not append: every run must surface what it deleted/collapsed (zero deletions needs a stated reason), and `CLAUDE.md` line count + milestone active-table row count are watched as health metrics — if both only grow, it raises an alert. Like `/read-handoff`, it trusts git/`gh` over handoff prose: a handoff claim that "X merged into dev" must be verified with `git log`/`gh` before it lands in `CLAUDE.md`. Changes are classified mechanical (pointer/status/hash swap — write directly), semantic (rewrite stale prose, move a finished track to the completed table, resolve a contradiction — write but highlight in the report), or new (new milestone/ADR — must pass a gating check and is quota-capped at ADR≤1, milestone≤2 per run to stop the skill itself from becoming an add-only machine). Writes everything, shows a full report, asks before committing.

Use on a project with a `docs/handoffs/` (by-month dirs), `docs/milestone/`, `CLAUDE.md` layout.

Usage: `/doc-sync [N]` (N = scan handoffs from the last N days, default 7)

### `/grill-with-fable`

Stress-test a plan, spec, or design decision by having Fable grill it — before you build. Two modes in one skill: **(A) pre-answer pipeline** — take the pile of one-at-a-time questions `/grilling` would ask, hand them to Fable to answer in one pass, then only the genuinely load-bearing decisions come back to you (Fable digests the noise, you rule on the few that matter); **(B) spec review** — generate a purely-abstract grill prompt that gets Fable to attack a spec/plan's soft spots and open assumptions, then return a verdict. Both modes enforce two hard rules: an **AUP self-check** (a grill prompt must never carry adversarial framing or security metaphors, or it trips content filters and gets refused — learned the hard way after two rejections) and **"grill's value is overturning your priors, not confirming them"** (questions must aim at what's marked tentative/leaning/candidate, not ask "is this right?"). Fable is never called directly — the skill always produces a prompt for you to forward manually.

Usage: say "grill fable", "have fable grill this design", "write me a grill prompt", or "let fable pre-answer the grill".

## Installation

```
/plugin install wangqiuyang/aki-skills
```
