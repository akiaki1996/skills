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

Before you finish, run the **full suite once in a way the reviewer can reproduce** — record the exact command and how the venv resolved (shared main `.pth` vs. an isolated `uv sync --frozen`). This is what step 4.5 turns into the environment receipt, so the reviewer can trust your green instead of re-deriving it.

**Completion criterion:** every test in 四 passes; every item in 三 is implemented.

## 4. Verify acceptance criteria

Check each item in **五、验收标准** explicitly. If any criterion requires a live run (dogfood/smoke test), note it as a manual step and flag it in the PR body.

**Completion criterion:** each criterion in 五 is either confirmed green or explicitly deferred with a note.

## 4.5. Record the environment receipt

The reviewer's slowest step is re-deriving facts you already know from having just run the suite. Collect them now so the PR carries them (they go into the PR body in step 6). Gather exactly four things:

- **base** — the exact SHA you branched from, not just the branch name: `git rev-parse origin/$BASE`, plus your worktree HEAD (`git rev-parse HEAD`). This lets the reviewer judge stale-base without guessing.
- **venv isolation** — did the worktree use the shared main venv or an isolated one? `cat` the worktree's `.pth` (or check whether it has its own `.venv`): a `.pth` pointing at the *main* repo means plain `pytest` tests **main** code, not the PR — so record whether you ran with an isolated `uv sync --frozen` venv, or overrode `PYTHONPATH=<worktree>/packages/...`. State the actual command you used.
- **config.yaml / live tests** — is `config.yaml` present in the worktree? `tests/test_client_live.py` skips the whole module when `config.yaml` is absent (or `CI` is set), and `requires_llm` tests skip without `OPENAI_API_KEY`. Say in one line which happened: live ran, or live skipped (and why).
- **result** — the suite's `N passed, M skipped, K failed`. If anything failed, list the failing **file set** (not just the count) and say whether it equals the known pre-existing baseline or is new.

This is a receipt the reviewer can spot-check, not a substitute for their own run — so report it honestly: the exact command, the real counts, the real skip reason.

**Completion criterion:** you have all four facts written down, ready to paste into the PR body.

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

## 环境收据

<paste the four facts from step 4.5, one per line — keep the field names so the reviewer can grep them>
- base: origin/$BASE @ <sha12>  (worktree HEAD <sha12>)
- venv: <独立 uv sync --frozen | 共享主仓 .pth,测试用 PYTHONPATH=<...>>
- config.yaml: <存在,live 已跑 | 缺席,live 模块 skip>
- 测试: <确切命令,如 `make test`> → <N passed, M skipped, K failed>
- 失败文件集(若 K>0): <[...] == 已知 baseline / 新增>

🤖 Generated with [Claude Code](https://claude.ai/claude-code)
EOF
)"
```

**Completion criterion:** `gh pr view` shows the PR open against `$BASE`; the body covers 改动内容, 验收, and 环境收据.
