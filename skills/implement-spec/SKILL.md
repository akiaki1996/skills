---
name: implement-spec
description: Implement a structured spec document. Read the spec, create a worktree off the current branch, develop per the spec's change list and tests, then push and open a PR.
disable-model-invocation: true
---

The spec path is the argument the user passed after the skill name (e.g. `/implement-spec docs/superpowers/specs/2026-06-22-foo-spec.md` → path is `docs/superpowers/specs/2026-06-22-foo-spec.md`). Resolve it relative to the current working directory if not absolute.

## 1. Read the spec

Read the spec file in full. Your spec has a fixed structure — locate these sections and treat them as the authoritative source for what to do:

- **`〇、给实施 agent 的一句话`** — one-sentence action summary; read it first to orient yourself
- **`三、改动清单`** — the exhaustive list of files to change, with line references and code snippets
- **`四、测试`** — the TDD test list; these must go red before you write implementation code
- **`五、验收标准`** — the completion criteria; use these to judge when you're done
- **`六、风险与注意事项`** — constraints you must not violate during implementation

Skim 一、二 (root-cause and design rationale) for context. Don't let rationale sections expand into implementation steps — the 改动清单 is authoritative.

**Completion criterion:** you can restate 〇 in one sentence and list every numbered item in 三 before proceeding.

## 2. Create the worktree

Derive the branch/worktree name from the spec filename:

```
spec filename:  2026-06-22-metric-metadata-sidecar-spec.md
strip date + suffix → metric-metadata-sidecar
worktree name:  worktree-metric-metadata-sidecar
```

Get the current branch, then create the worktree off it:

```bash
BASE=$(git branch --show-current)
git fetch origin
git worktree add .claude/worktrees/worktree-<topic> -b worktree-<topic> origin/$BASE
```

Work inside the new worktree for all subsequent steps.

**Completion criterion:** `git worktree list` shows the new worktree; you are working inside it.

## 3. Implement — red first, then green

For each item in **四、测试**:

1. Write the test (it must fail).
2. Write the minimum implementation to make it pass.
3. Move to the next test.

Follow every constraint in **六、风险与注意事项** as you code. When a constraint says "守 SSOT" or references a memory key, honour the design intent — don't shortcut it.

Run tests frequently. When the spec references file paths like `/mnt/user-data/workspace/...`, these are sandbox-internal paths; the actual source files are under `packages/`.

**Completion criterion:** every test in 四 passes; every item in 三 is implemented.

## 4. Verify acceptance criteria

Check each item in **五、验收标准** explicitly. If any criterion requires a live run (dogfood/smoke test), note it as a manual step and flag it in the PR body.

**Completion criterion:** each criterion in 五 is either confirmed green or explicitly deferred with a note.

## 5. Commit and push

```bash
git add -p          # stage purposefully, not with -A
git commit -m "<Chinese summary of what changed and why>"
git push -u origin worktree-<topic>
```

Commit message: a concise Chinese sentence describing the intent, not the mechanism.

**Completion criterion:** `git log --oneline -3` shows your commit; `git push` exits 0.

## 6. Open the PR

Target branch is the branch you branched from (captured in step 2 as `$BASE`).

```bash
gh pr create \
  --base $BASE \
  --title "<spec title in one line>" \
  --body "$(cat <<'EOF'
## 背景

<paste 〇 here>

## 改动内容

<list each numbered item from 三 in one line each>

## 验收

<list each item from 五, with status: ✅ confirmed / ⏳ needs manual run>

🤖 Generated with [Claude Code](https://claude.ai/claude-code)
EOF
)"
```

**Completion criterion:** `gh pr view` shows the PR open against `$BASE`; the body covers 改动内容 and 验收.
