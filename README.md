# aki-skills

Claude Code skills I use in my own workflow.

## Skills

### `/handoff`

When a session grows too long to continue effectively, generate a structured handoff document so the next agent can pick up exactly where you left off — with file paths, decisions made, and a concrete first step, not a vague summary.

### `/read-handoff`

The other half of `/handoff`: start a new session by reading a handoff document and figuring out where to actually pick up. Because a handoff is a snapshot that goes stale, this skill cross-checks every claim against the current git state — what shipped since the handoff's date, whether its "to-do" items were already done by later commits, whether HEAD has moved past it — and against the same-period spec/plan docs it references. It then reports the true pick-up point (handoff says X, git shows Y, here's what's actually left) and asks you what to do next. Read-only: it never touches git or code on its own.

Usage: `/read-handoff path/to/handoff.md` (omit the path to pick the latest)

### `/implement-spec`

Given a structured spec document, create a worktree off the current branch, implement the change list test-first, then push and open a PR. My specs follow a fixed format: a one-sentence action summary, a change list, TDD tests, acceptance criteria, and risk notes — this skill reads all of them.

Usage: `/implement-spec path/to/spec.md`

### `/loop-pr-merge`

Stand up a self-paced loop that watches a repo for new PRs and reviews-then-merges each one against *that repo's own* development and merge rules. Reads `CLAUDE.md` first, recovers each PR's spec + diagnosis for project context, then reviews with risk-scaled rigor (stale-base 3-way merge, protected-file byte checks, import-ring, red→green, neighborhood seesaw, sync discipline). On a clean pass it squash-merges; on a failure it either fixes the PR in place and pushes (when the branch is a local worktree you own) or posts a directed changes-requested review. Distilled from a real session that merged ~25 consecutive PRs.

Usage: `/loop-pr-merge [repo-path]`

## Installation

```
/plugin install wangqiuyang/aki-skills
```
