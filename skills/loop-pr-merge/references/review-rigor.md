# Review rigor — risk-tiered verification methodology

This is the depth behind SKILL.md §4. Read the section matching the PR's risk tier; always do the "recover project context" step (§0) and the common checks.

## §0 — Recover the project context first (do this before reading code)

A PR is never a stranger. In this workflow every change starts from a **spec**, and the spec carries the cause-and-effect the diff alone can't show: *why* this change exists, what root cause it targets, what the manifest of intended changes is, and what the acceptance criteria are. Review against that intent — not against a blank slate.

1. **Find the spec.** Check, in order: the PR body (it usually links `docs/superpowers/specs/<date>-<topic>-spec.md` and/or a `docs/problems/<date>-...md` diagnosis); the branch/worktree name (often mirrors the spec slug); recent specs in `docs/` by date. Read the spec's one-line action summary, change list (三/manifest), tests (四), acceptance criteria (五), and risk notes (六).
2. **Find the diagnosis / problem doc** if one is referenced — it tells you the *observed failure* (often a dogfood/prod trace) and the root-cause reasoning. This is what lets you judge "does the fix actually address the root cause, or just a symptom".
3. **Recover the local memory thread.** If you keep review memory, search it for this PR's area: prior PRs that touched the same files, lessons learned, known root causes, retracted hypotheses. A PR that contradicts an established root cause (or re-introduces a fixed bug) is a finding, not a pass.
4. **Place the PR in the sequence.** What merged just before it? Does it build on a recent PR (then verify that PR's pieces survive) or supersede one (then verify the old approach is cleanly replaced)?

**Why this matters:** without it you review a diff in a vacuum and miss the two failure modes that cost the most — (a) a fix that's locally plausible but wrong against the real root cause, and (b) a change that silently collides with or reverts something merged days ago. The spec + diagnosis + memory restore exactly that context.

**Completion criterion:** you can state the PR's root cause, its intended manifest, and how it relates to the last few merges — before judging a single line of the diff. Then verify the diff *against the spec's manifest and acceptance criteria* (does it do everything the spec claimed; did it quietly do more — scope creep).

## §receipt — the environment receipt: what you may trust, what you must still run

A PR produced by `/implement-spec` carries a `## 环境收据` block in its body: the exact `base` SHA it branched from, the `venv` isolation it tested under, how `config.yaml`/live tests were handled, and the suite result (`N passed, M skipped, K failed`). The point is to let the reviewer skip *re-deriving facts the implementer already knew* — **not** to skip verification. Split the fields by kind:

**Deterministic facts — trust after a 1-second check:**
- **base SHA.** Confirm against `gh pr view <N> --json baseRefName,headRefOid`. If it matches, take the receipt's stale-vs-clean judgement; you don't recompute the merge-base from scratch.
- **venv isolation.** If it says an isolated `uv sync --frozen` (or a `PYTHONPATH` override at the worktree's package source), trust that the reported run exercised the **PR** code, not the main repo through a shared editable `.pth`. That is exactly the false-green trap (worktree shares main venv) — a receipt that names the isolation resolves it without you re-standing-up the path yourself.
- **config.yaml / live handling.** The receipt states whether `config.yaml` was present (so `test_client_live.py` ran) or absent (module-skipped), and whether `requires_llm` ran. This tells you *why* the skipped count is what it is — so a skip in your own clean-worktree run (no `config.yaml`) isn't a discrepancy to chase.

**Self-reported result — downgrade scope, never trust as terminal:**
- The `N passed` count does **not** let you merge on the implementer's word. It downgrades your rerun from "full exploratory suite (is this thing even green?)" to "the incremental neighborhood seesaw aligned to **current staging HEAD** (what new red does merging introduce?)". That one authoritative rerun — clean venv + aligned to current HEAD + you watching it go red — is never skipped. This is the guard against reward-hacking and dual-track self-report masking a miscount: the gate reads ground truth (your run on disk), not the model's claim.
- **But "downgrade to neighborhood" means the neighborhood, not the whole suite.** The reviewer is **not** the full-suite oracle — CI and the receipt are. Re-running the entire repo suite for every self-authored PR does the author's job and, times N PRs, dominates the loop for no added signal. Instead: run the neighborhood, confirm CI green, and **re-derive each of the receipt's named failures yourself** — isolate each under the correct path and confirm it's the known env false-red it claims (not a real regression bucketed as noise; that mislabel has happened). Reserve a genuine full-suite rerun for shared-infra changes (dispatcher / parse / kernel / middleware chain / config), delete-heavy refactors, or a receipt whose failure-bucketing looks wrong. Caveat: if the repo's CI does **not** run the full suite (only a narrow guard like blocking-IO), there's no CI oracle for "full suite green" — tighten the per-failure recheck, but this still doesn't make blanket full-suite reruns the default.

**No receipt** (external contributor, pre-receipt PR): no downgrade — run the full flow.

**Unaffected by the receipt (always run in full):** red→green proof, protected-file byte-level head-to-head, assembly-chain runtime check, import-ring. A self-reported count is structurally incapable of standing in for any of these.

## Common checks (every code PR)

- **Read the full diff.** Cross-check each PR-body / spec claim against the code.
- **Scope.** Touched files ⊆ what the spec's change list authorizes. Flag scope creep.
- **Honesty of the body.** Acceptance items marked ✅ should be verifiable; items needing a live run (dogfood) should be marked ⏳ — confirm they really are deferred, not silently skipped.

## §stale-base — verify the 3-way merge result, not the PR branch

If the PR's base diverged from current `origin/dev`, the PR branch in isolation is *not* what will land. Verify the actual merge:

```bash
TREE=$(git merge-tree --write-tree origin/dev <PR_HEAD> | head -1)
git cat-file -t "$TREE"   # must say "tree" (clean). If it prints conflict text, inspect.
COMMIT=$(git commit-tree "$TREE" -p origin/dev -p <PR_HEAD> -m "test-merge")
git worktree add --detach /tmp/pr<N>-check "$COMMIT"
```

Then:
- List the dev commits the PR is missing (`git log <merge-base>..origin/dev --oneline`) and check whether any touch the **same files** as the PR — those are the merge hot-spots.
- Explicitly confirm recent merges survive in the merge result (e.g. grep for the latest tool/symbol/constant that landed just before this PR). Zero file-overlap is reassuring but still confirm the key artifacts are present.

## Worktree / venv setup (so tests exercise the PR, not main)

A repo's `.claude/worktrees/*` typically **share the main backend venv** via an editable `.pth` that points at the *main* repo source. Consequences:
- Plain `pytest` inside the worktree imports **main** code, giving false green/red.
- Override the package path: `PYTHONPATH=<worktree>/.../packages/harness pytest ...` so `import deerflow` resolves to the worktree source. Confirm with `assert "<worktree path>" in module.__file__`.
- If the worktree has its **own** venv (`cat` the `.pth` to check where it points), use that instead.
- For a second package (e.g. `ethoinsight`), shadow-load its source the same way: `PYTHONPATH=<worktree>/packages/ethoinsight`.
- Run from the directory where the app package resolves (e.g. `cd .../backend` so `app/` is importable).

## Import-ring (after touching harness core: middleware / executor / subagents / tools / agents)

Bare-import the production entrypoints; a partially-initialized-module error means a new top-level import closed a cycle.

```bash
PYTHONPATH=<wt-harness>:. python - <<'PY'
import app.gateway
from deerflow.agents import make_lead_agent
# + import the specific modules the PR changed
print("import-ring OK")
PY
```

Also grep for newly-introduced top-level `from <heavy-optional-dep> import` — those must stay lazy (in-function) so the harness imports without the optional dep installed.

## Red→green proof

- The PR's new tests must **fail against pre-change source** and **pass against the change**. If they don't fail on pre-change, they're tautological and prove nothing.
- To prove red: detached worktree at the merge-base, overlay only the new/changed test files, run them — expect failures (or collection errors if the test references a brand-new symbol).
- **Concurrency fixes**: red must be *stable* — run the no-lock/no-fix source across **multiple iterations** (use a `threading.Barrier` to align thread start and force the race); a single green run proves nothing. Green must hold under **stress** (run the fixed version many times).
- **Deterministic fixes** (clamp, sort, enumeration, exists()-check): a single red→green is enough — the behavior is determinate.

## Neighborhood seesaw + test-failure triage

Run the full test neighborhood around the change (the changed suites + their adjacent suites + import-cycle test). Then triage any failures against the known baseline:

- **standalone-red** (fails when that one file is run alone) = real pre-existing debt; not caused by this PR, but flag it.
- **full-suite-only-red** (passes alone, fails in the full run) = test-isolation pollution; compare the failing **file set** to the pristine-dev baseline, not the count.
- **PYTHONPATH-prefix-only-red** = namespace/editable-install artifact; not a real failure.

The merge is safe when the PR's failing-file set **equals** the pristine-dev baseline failing-file set (zero *new* failures), even if the raw count looks alarming.

## §sync — sync PRs are the highest-stakes (read the repo's sync rules in CLAUDE.md first)

A sync PR follows upstream and must preserve local customization. Apply *all* of the above, plus:

1. **3-way merge against current dev** (sync branches are usually stale) and **confirm every recent local PR survives** — a sync can silently overwrite a customization or revert a just-merged fix.
2. **Protected files: byte-level head-to-head, never grep-only.** For each protected file diff base-vs-PR-head and read it. The danger is **paraphrase-merge**: upstream rewrites a prompt and the constraint umbrella sentence is softened or a typo is introduced while the grep string still matches. Verify the customization survives **in the code/behavior**, not just in a comment or docstring.
3. **Distinguish "preserve" vs "follow".** Within surgical merges: *business customization* (local isolation logic, local prompt constraints, registry literals) must be **preserved**; *architectural style* (upstream refactors, perf rewrites, new middleware) should be **followed** — don't reject an upstream refactor just to keep old structure.
4. **Registry files** (collection literals / aggregate imports — e.g. `tools.py`, `builtins/__init__.py`) must be surgical-merged; confirm nothing was dropped.
5. **The protected-file list lags reality.** Diff *every* touched file, not just the declared protected list — local customization may live in files not on the list.
6. **Sync-state bump.** Confirm the recorded sync-state advances to the new upstream commit with an accurate manifest.
7. **Broad seesaw** must cover both the sync's own test suites **and** the recent-PR neighborhoods, to prove coexistence.

## Assembly chain (new tools)

A tool is not usable until it's wired in **every** required place. For this stack that's: the `@tool` definition + the package `__init__.py` export + the `tools.py` `BUILTIN_TOOLS` registry + the consuming agent's tool allow-list. Verify **at runtime**, not by grep:

```python
from deerflow.tools.tools import BUILTIN_TOOLS
assert "the_tool" in {t.name for t in BUILTIN_TOOLS}
from deerflow.subagents.builtins.<agent> import <AGENT>_CONFIG
assert "the_tool" in <AGENT>_CONFIG.tools
```

A unit test of the tool function itself does **not** exercise the assembly chain — a tool can be fully defined, exported, and unit-tested, yet absent from the runtime registry (100% production failure). Always dump the resolved tool set.

Related: deterministic batch-execution tools that pass **argv** to in-process scripts must **pre-resolve virtual paths in the argv** before handing them to the script — the script reads `args.output` raw and won't resolve sandbox/virtual paths itself. Test this with a runner that does **real file IO**, not a capture-only runner (a capture runner sees the unresolved path and passes, hiding the bug until a live run).

## Three-pathology self-check (agent-harness repos)

If the repo treats agent-harness quality as a first-class concern, sanity-check the change against:

- **Reward hacking** — can the model fake success? (claim outputs it didn't produce, self-declare state the gate trusts). Gates should read ground truth (disk, user text), not the model's self-report.
- **Catastrophic forgetting** — does the change drop a prior behavior/constraint? (3 instruction sources — system_prompt > dispatch prompt > SKILL.md — must stay aligned; data semantics preserved across refactors.)
- **Under-exploration / decision drift** — is something that should be a checkable rule left to LLM free-discretion? Prefer a deterministic gate over a new reminder-prompt (stacking reminders degrades, it doesn't fix).
