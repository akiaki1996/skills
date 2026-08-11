---
name: write-plan
description: 将已确认的 spec 转成详细实施 plan（施工图：有序 Task + 完整代码 + 红绿 commit），写入 docs/plans/。给 implement-plan 直接照落地。在 spec 已确认、准备派实施时触发。
model-invoked: true
---

# write-plan — 实施 plan 写入

将一份已确认的 spec 转成实现 agent 可以直接执行的详细施工图 plan，写入 `docs/plans/YYYY-MM-DD-<slug>-plan.md`。

## plan 是什么

- **plan = 施工图**：回答"怎么做、按什么顺序"。把 spec 拆成有序 Task，每 Task 含完整可复制代码 + 精确命令 + 期望输出。
- **plan 是实施权威**：implement-plan 照 plan 的 Task 逐步落地；spec 只作背景理解。
- **SSOT**：完整实现代码只在 plan（不在 spec 重复）。

## 触发条件

- 对应的 spec 已被用户确认
- 用户说"写 plan""出实施计划""准备派实施"或类似表述

## 落笔前：读一遍项目惯例

- **对应 spec**：读它的设计方案、验收标准、风险（plan 的 Task 从这里推导）
- **项目约定**：`CLAUDE.md` / `README` / `Makefile` / `package.json` / `pyproject.toml`——**本项目怎么跑测试、lint、构建**（Task 的运行命令必须是真实命令，不能瞎编）
- **editable-install 假绿陷阱**：若项目是 uv workspace 成员（`pyproject.toml` 有 `[tool.uv.workspace]` / `workspace = true` 源），读它时确认**测试命令在 worktree 里怎么用独立 venv 跑**——主仓 venv 的 editable `.pth` 是绝对路径、指向主仓源码，implement-plan 复用它会假绿（测试跑在主仓代码上）。Task 的测试命令必须发 isolated-venv 的真实命令（如从 worktree 的 workspace 根 `UV_PROJECT_ENVIRONMENT=<wt>/packages/<pkg>/.venv uv sync --frozen` 后 `<wt>/.../.venv/bin/python -m pytest`），不能只写裸 `pytest` 或共享主仓 venv 的命令
- **plan 存放惯例**：项目里已有 plan 目录吗？沿用它（否则默认 `docs/plans/`）
- **测试目录 + 命名**：新测试放哪、怎么命名（照项目惯例）

## Plan 文档的固定结构

创建文件 `docs/plans/YYYY-MM-DD-<slug>-plan.md`。

### 标题 + 元信息段

```markdown
# <短描述> Implementation Plan

> **For agentic workers:** 每个 Task 的 5 步（写红测试 → 验证红 → 写实现 → 验证绿 → commit）
> **就是 tdd 的 red-green-refactor 纪律**——逐步执行即在做 TDD，不必额外调 skill。
> 全部 Task 完成后，收尾 Task 做 code-review 双轴自检（Standards + Spec fidelity，见收尾 Task）。
> Steps use checkbox (`- [ ]`) syntax for tracking.

**对应 spec:** `docs/specs/YYYY-MM-DD-<slug>-spec.md`
**Goal:** <一句话目标>
**Architecture:** <3-5 句架构决策总结>
**Tech Stack:** <语言 + 框架 + 测试工具。标注：改动面（前端/后端/跨层）>
```

> **`对应 spec` 字段是必填的**——implement-plan 靠它顺藤读到 spec 建立背景理解。plan 是实施权威（含完整代码），spec 是"为什么"的背景（功能说明）。

### Global Constraints 段

在 plan 顶部列出所有全局约束。这些约束对每个 task 都生效。从 spec 的风险段 + 项目 CLAUDE.md 提取真实约束，示例：

```markdown
## Global Constraints

- **行为不变**：重构类改动必须对账（逐字节 / 快照 / 等价性，如有）
- **测试位置**：新测试放 `<项目测试目录>`，命名 `<项目命名惯例>`
- **TDD 强制**：每个 task 先红后绿
- **改核心模块后验证生产入口**：改动被广泛 import 的基础模块后，除跑测试外，用本项目的启动/导入方式确认生产入口不炸（命令见项目 CLAUDE.md；测试可能因 mock 假绿，裸启动/裸导入才抓得到导入环）
- **惰性 import**：改动涉及被广泛依赖的核心模块时，新 import 不放模块顶层（防导入环）
- **防 vacuous**：对账/快照测试必须能在故意破坏时变红
- **遇失败走 diagnose 纪律**：任何测试从红转绿失败、或出现非预期行为时，走诊断循环再修——reproduce（测试里稳定复现）→ minimise（缩到最小触发用例）→ instrument（加一个能抓到该 bug 的断言，确认它红）→ fix → regression-test（新断言绿且不回归旧测试）。诊断清因再改，不盲改重跑
- **受保护/上游代码 surgical**：触及 vendored/fork/受保护区域时，逐处编辑不整文件覆盖（若 spec 标注了）
```

### Task 序列

每个 task 按固定格式：

```markdown
### Task N: <任务名>

**Files:**
- Create: `<新文件路径>`
- Modify: `<改动的文件路径>`
- Test: `<新增测试文件路径>`

**Interfaces:**
- Consumes: <这个 task 依赖哪些已有函数/模块>
- Produces: <这个 task 产出什么接口/函数/模块>

- [ ] **Step 1: Write the failing test**

<完整测试代码，可直接复制运行>

- [ ] **Step 2: Run test to verify it fails**

Run: `<本项目的测试命令>`
Expected: FAIL with `<期望的失败信息>`

- [ ] **Step 3: Write minimal implementation**

<完整实现代码>

- [ ] **Step 4: Run test to verify it passes**

Run: `<本项目的测试命令>`
Expected: PASS (N tests)

- [ ] **Step 5: Commit**

```bash
git add <文件列表>
git commit -m "<commit message>"
```
```

### Task 间依赖

如果一个 task 依赖前面的 task，在 task 开头标注：

```markdown
**Depends on:** Task 1 (<前置产物> 已存在)
```

### 最后一个 Task：收尾验证

```markdown
### Task N: 收尾与 code-review 自检

**Files:** None (验证 task)

- [ ] **Step 1: 全量测试**
  Run: `<本项目全量测试命令>`
  Expected: 全绿（含旧测试 + 本次新增）

- [ ] **Step 2: lint / 格式检查**
  Run: `<本项目 lint 命令>`
  Expected: 零 warning

- [ ] **Step 3: 生产入口验证（如果改了核心模块）**
  Run: `<本项目的启动/裸导入命令，从 CLAUDE.md 找>`
  Expected: 生产入口 0 退出、正常启动（抓导入环 / 启动期错误）

- [ ] **Step 4: code-review 双轴自检（内联，不必调 skill）**
  逐项过两轴：
  - **Standards 轴**：代码质量——有无 bad smells（重复、过长函数、命名含糊）；有无安全隐患；是否 surgical（每处改动都能追溯到某个 Task，没多做 spec 没要求的）
  - **Spec fidelity 轴**：对照 spec 的「4. 验收标准」逐条核——
     - 有没有漏掉 spec 的约束或验收项？
     - 有没有多做了 spec 没要求的？
  Expected: 两轴均无未解决发现

- [ ] **Step 5: 领域/端到端验证（如果项目有 golden 数据集或 e2e 手段）**
  <具体验证步骤或标注 N/A>
```

## 必做的交叉检查（写 plan 时自动执行）

### 1. 确保每个 task 的粒度可在一个实现 session 内完成

- 每个 task 对应 1-3 个文件
- 每个 task 的代码量适中（约 ≤100 行 new + modify）
- Task 总数与 spec 复杂度匹配（简单重构 3-5 task，复杂功能 8-15 task）

### 2. 确保 task 之间没有循环依赖

- Task 依赖链必须是 DAG
- 如有互不依赖的 task，标注 `可并行`

### 3. 确保 test 代码和实现代码在同一个 task 里

- 不让实现 agent 在 "先写全部测试" 和 "先写全部实现" 之间切换
- 每个 task 是完整的垂直切片

### 4. 确保每个 task 有明确的验收标准

- 测试通过 = 验收通过
- 没有 "手动验证" 或 "目视检查" 类模糊验收

### 4.1 确保每个 Run 命令都指向 worktree 独立 venv（editable-install 假绿陷阱）

- 若项目是 uv workspace 成员，`Run:` 里的测试命令必须用 **worktree 内独立 venv**：如 `cd <wt>/packages/agent/backend && UV_PROJECT_ENVIRONMENT=<wt>/packages/<pkg>/.venv uv sync --frozen` 后再 `<wt>/packages/<pkg>/.venv/bin/python -m pytest`。
- 不能写裸 `pytest` / 共享主仓 venv 的命令——implement-plan 复用主仓 editable `.pth` 时测试跑在主仓代码上，会假绿（见「落笔前」的 editable-install 假绿陷阱）。

### 5. 确保收尾 task 包含本项目特有的验证步骤

- 全量测试、lint（用真实命令）
- 生产入口验证（如改核心模块）
- code-review 双轴自检
- 领域/e2e 验证（如项目有此手段）

## 产出

生成的 plan 文件路径：

```
docs/plans/YYYY-MM-DD-<slug>-plan.md
```

其中 `<slug>` 匹配对应 spec 的 slug。写完后提示用户：plan 已就绪，新 session 用 `/implement-plan <plan路径>` 落地。
