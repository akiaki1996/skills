---
name: implement-plan
description: Implement a structured plan document. Read the plan (the construction blueprint — ordered tasks with complete code), skim the linked spec for background, create a worktree, execute each task test-first, then push and open a PR.
disable-model-invocation: true
---

The plan path is the argument the user passed after the skill name (e.g. `/implement-plan docs/superpowers/plans/2026-06-22-foo-plan.md` → path is `docs/superpowers/plans/2026-06-22-foo-plan.md`). Resolve it relative to the current working directory if not absolute.

## 1. Read the plan (authority) + skim the spec (background)

**The plan is the construction blueprint and the authoritative source for what to do** — it has ordered Tasks, each with complete copy-pasteable code and exact verify commands. Read it in full:

- **Header** — `Goal` / `Architecture` / `Tech Stack`, and the **`对应 spec:`** field (a path to the design spec).
- **`Global Constraints`** — project-wide rules that apply to every Task; honour all of them.
- **Task 序列** — the ordered list of Tasks you will execute one by one in step 3. Each Task's 5 steps (write failing test → verify red → write implementation → verify green → commit) **are the tdd red-green-refactor discipline**.

Then open the spec named in `对应 spec:` and **skim it for background** — the "why" behind the design, the acceptance criteria (`4. 验收标准`), and the risks (`5. 风险与注意事项`). The spec is context, **not** a step list; the plan's Tasks are what you implement. If the plan has no `对应 spec:` field, proceed with the plan alone and note the missing reference in the PR body.

**Completion criterion:** you can restate the Goal in one sentence and list every Task before proceeding.

## 2. Create the worktree

Derive the branch/worktree name from the plan filename:

```
plan filename:  2026-06-22-metric-metadata-sidecar-plan.md
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

### 2.1. Backfill gitignored local config (common-pitfall guard)

A fresh `git worktree add` checks out only tracked files — **gitignored local config that tests/lint need at runtime is missing**, which surfaces as confusing failures the agent then spends a long time diagnosing. Backfill it now from the main worktree (the one you ran the command from), generically — **do not hardcode project paths**.

Default backfill list (extend per project as needed): `.env`, `config.yaml`, `config.yml`, and any `*.local.*` file.

```bash
MAIN=<main worktree root, e.g. the cwd you ran step 2 from>
WT=.claude/worktrees/worktree-<topic>
# Copy each file only if it exists in MAIN and is absent in the worktree — never overwrite.
for f in .env config.yaml config.yml; do
  [ -f "$MAIN/$f" ] && [ ! -f "$WT/$f" ] && cp "$MAIN/$f" "$WT/$f"
done
# Same for *.local.* (e.g. .env.local), recursively mirroring path under WT.
cd "$MAIN" && find . -type f -name '*.local.*' -not -path './.claude/worktrees/*' 2>/dev/null | while read f; do
  [ ! -f "$WT/$f" ] && mkdir -p "$WT/$(dirname "$f")" && cp "$f" "$WT/$f"
done
```

Rules:
- **venv is never copied** — reuse the main repo venv (it stays out of the worktree). To avoid false-green tests that run main-repo code via the editable `.pth`, follow the existing guidance: prefer an isolated `uv sync --frozen` in the worktree, or point `PYTHONPATH` at the worktree's own `packages/...` source.
- **Never `git add` these backfilled files** — they are gitignored because they carry secrets / are machine-local; copying them into the worktree is fine, committing them is not. The PR receipt (step 4.5) reports only *presence*, never contents.

**Completion criterion:** every file on the backfill list that exists in the main worktree is now present in the worktree (newly copied) or was already there (left untouched); `.venv` was not copied.

**Completion criterion:** `git worktree list` shows the new worktree; you are working inside it.

## 3. Implement — execute the plan's Tasks (tdd: red first, then green)

Work through the plan's **Task 序列 in order**. Each Task carries its own complete code and commands; **executing a Task's 5 steps IS the tdd red-green-refactor cycle**:

1. Write the failing test from the Task (run it, confirm it goes **red** for the right reason).
2. Write the minimal implementation from the Task (run the test, confirm **green**).
3. Commit as the Task specifies, then move to the next Task.

Honour every rule in the plan's **`Global Constraints`** as you code. When a constraint says "守 SSOT" or references a memory key, honour the design intent — don't shortcut it.

Run tests frequently. When the plan references file paths like `/mnt/user-data/workspace/...`, these are sandbox-internal paths; the actual source files are under `packages/`.

**When a test won't go green, or you hit unexpected behaviour, run the diagnosis loop before touching code again:**

1. **reproduce** — pin a stable repro inside a test (not a manual one-off run).
2. **minimise** — shrink to the smallest input/state that still triggers it.
3. **instrument** — add an assertion (or log) that *catches this specific bug*; confirm it goes red for the right reason.
4. **fix** — make the minimal change that turns it green.
5. **regression-test** — confirm the new assertion is green *and* the full suite still passes.

Diagnose the cause, then fix — rather than editing code and re-running the suite hoping for green.

Before you finish, run the **full suite once in a way the reviewer can reproduce** — record the exact command and how the venv resolved (shared main `.pth` vs. an isolated `uv sync --frozen`). This is what step 4.5 turns into the environment receipt, so the reviewer can trust your green instead of re-deriving it.

**Completion criterion:** every Task's tests pass; every Task is implemented and committed.

## 4. Verify — code-review dual-axis (inline, no skill call needed)

Run both axes before finishing:

**Spec fidelity axis** — check each item in the spec's **`4. 验收标准`** explicitly. If any criterion requires a live run (dogfood/smoke test), note it as a manual step and flag it in the PR body. Also confirm you didn't *miss* a plan Task or a spec constraint.

**Standards axis** — pass over the code once more: any bad smells (duplication, over-long functions, vague naming); any security concern; is it **surgical** — every changed line traces to a plan Task, with nothing done beyond what the spec/plan asked (see 用户 CLAUDE.md「Surgical Changes」).

**Completion criterion:** each acceptance criterion is confirmed green or explicitly deferred with a note, and both axes have no unresolved findings.

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

Never `git add -A` or `git add .` — they can sweep in secret-bearing gitignored files (the step-2.1 backfill: `.env`, `config.yaml`, `*.local.*`). Use `-p` only; those files never leave the worktree.

Commit message: a concise Chinese sentence describing the intent, not the mechanism.

**Completion criterion:** `git log --oneline -3` shows your commit; `git push` exits 0.

## 6. Open the PR

Target branch is the branch you branched from (captured in step 2 as `$BASE`).

```bash
gh pr create \
  --base $BASE \
  --title "<plan/spec title in one line>" \
  --body "$(cat <<'EOF'
## 背景

<paste the spec's one-line problem summary (from 「1. 问题定义与根因」) here>

## 改动内容

<list each plan Task in one line each>

## 验收

<list each item from the spec's 「4. 验收标准」, with status: ✅ confirmed / ⏳ needs manual run>

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
