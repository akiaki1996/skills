# aki-skills

Claude Code skills I use in my own workflow.

## Skills

### `/handoff`

When a session grows too long to continue effectively, generate a structured handoff document so the next agent can pick up exactly where you left off — with file paths, decisions made, and a concrete first step, not a vague summary.

### `/implement-spec`

Given a structured spec document, create a worktree off the current branch, implement the change list test-first, then push and open a PR. My specs follow a fixed format: a one-sentence action summary, a change list, TDD tests, acceptance criteria, and risk notes — this skill reads all of them.

Usage: `/implement-spec path/to/spec.md`

## Installation

```
/plugin install wangqiuyang/aki-skills
```
