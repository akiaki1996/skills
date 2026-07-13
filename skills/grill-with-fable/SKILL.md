---
name: grill-with-fable
description: 用 Fable 做 grill（压力测试一个 plan/spec/设计决策）。两个场景合一——(A) 把 matt 的 grilling 生成的一大堆逐条拷问交给 Fable 先预答一轮，你只收束最关键的决策；(B) 生成一份纯抽象的 grill prompt 让 Fable 评审一份 spec/plan。凡用户说「grill fable / 让 fable 拷问这个设计 / 帮我写个 grill prompt / fable 预答一下 grill / 用 fable 压测这个 plan」，或在设计 session 里想在动手前请 Fable 挑战一份 spec/plan/决策，就用这个 skill。它会守住 AUP 自检（grill prompt 绝不能带对抗词/隐喻，否则触发拦截）和「grill 价值在推翻预判不在确认」两条硬约束。Fable 不直连——skill 一律生成 prompt 交你手动转发。
---

# grill-with-fable

把「用 Fable 做 grill」这个高频动作固化下来。Fable 判断力强，适合在动手前狠狠拷问一份设计——但每次手写 grill prompt 有两个坑反复踩：**AUP 拦截**（对抗性措辞命中网络安全类目，被拒答）和**grill 沦为确认**（写成让 Fable 点头，白 grill）。这个 skill 把这两条守住，并提供两种编排。

## 先判场景

问自己（或问用户）：现在是哪种？

- **场景 A — grill-me 预答流水线**：你有一份 plan/设计，想被逐条拷问，但**不想一次回答一大堆**。让 matt 的 `grilling` 先生成全部问题，Fable 替你预答一轮，你只收束最关键的几个决策。
  - 触发信号："grill 我但问题太多""让 fable 先答一遍 grill""grill-me 预答"。
- **场景 B — Fable 评审 spec/plan**：你有一份写好的 spec/plan/设计文档，想让 Fable 当外部评审狠挑，输出裁决。
  - 触发信号："写个 grill prompt 让 fable 评审这个 spec""用 fable 压测这份 plan"。

不确定就问用户一句。两个场景都以**生成一段交用户手动转发给 Fable 的 prompt** 为核心——**skill 不直连 Fable**（用户没接 Fable API）。

---

## 贯穿两场景的两条硬约束（不可违反）

### 1. AUP 自检 — 生成任何要转给 Fable 的 prompt 后必做

grill prompt 若带**对抗性措辞或安全类隐喻**，会字面命中网络安全类目、触发 Fable 拒答（血泪：某批 grill prompt 因此被拦两次，纯抽象重写第三版才过）。

生成 prompt 后，扫一遍 `references/aup-wordlist.md` 的禁用词。命中即改写成纯抽象中性词（系统/单元/计算/判断/分离/约束/校验独立性）。**用正向抽象描述系统与问题，不用隐喻、不用"防X""抗X""绕过X"这类框架。** 详见 `references/aup-wordlist.md`。

自检可用：`grep -noE '<禁用词alternation>' <prompt文件>`，应全 0（背景段也要干净，防整段被贴）。

### 2. grill 的价值在推翻预判，不在确认

grill 不是让 Fable 附和你已经想好的答案。prompt 必须逼 Fable **攻击设计里的隐含假设、软决策、与既定原则的张力**——而不是问"这样对不对"。判据：如果一个问题的"预期答案"就是 spec 现在的写法，它是坏问题；好问题瞄准 spec 里标了"待定/倾向/候选"的地方，和你没意识到的前提。（memory `feedback_grill_handoff_must_be_verified`：grill 的价值恰在推翻预判。）

---

## 场景 A：grill-me 预答流水线

### A1. 生成拷问问题（借 matt 的 grilling）

读 `~/.agents/skills/grilling/SKILL.md`（或 `~/.claude/skills/grilling/SKILL.md`）的精神：走设计树、逐条拷问、每个决策一个问题、fact 自己查代码不问人。

**但不要真进入"一次一问、等用户答"的交互**——那正是用户想避开的。改为：把该问的问题**一次性全列出来**（走设计树、解依赖、每题附你的推荐答案），作为待预答的清单。fact 类先自己 grep 代码补上，只留 decision 类。

### A2. 生成"请 Fable 预答"的 prompt（交用户手动转发）

把问题清单包成一段给 Fable 的 prompt。结构：

```
你在评审一个软件系统的设计决策。下面是一组逐条决策问题，每题请给出：
(a) 你的推荐答案 + 一句判据；(b) 若答案取决于别的决策，标出依赖；
(c) 这题的重要度（关键/次要）——关键 = 答错会导致返工或违反核心原则。

[系统设定：抽象描述被评审的系统，纯抽象，见 references/grill-prompt-skeleton.md]

问题：
1. ...
2. ...
（逐条）

输出：逐题给 (a)(b)(c)，最后列出你标为"关键"的题号。
```

**过 AUP 自检**（硬约束 1），然后交用户：「把下面整段转给 Fable，把它的回答贴回来。」

### A3. 收束 —— 只把关键决策拎给用户

用户贴回 Fable 的预答后：

1. 按 Fable 标的重要度 + 你的判断，把问题**分两档**：
   - **需人拍板**：关键决策（答错返工/违反核心原则）、Fable 答案你存疑、或涉及用户偏好（偏好归人，见 memory）。
   - **Fable 已答够**：次要的、Fable 答案合理且无争议的——直接采纳，列出来告诉用户"这些按 Fable 答案定了"，不逐个问。
2. 只把"需人拍板"的用 **AskUserQuestion** 拎出来问用户（一次问几个关键的，不要把全部倒回去）。
3. 用户定完，汇总成决策记录（若是大设计，仿 `docs/superpowers/plans/` 下的 decision-record 格式落档）。

核心：**Fable 消化杂问题、你只碰关键偏好**——这就是这个场景的全部价值。

---

## 场景 B：Fable 评审 spec/plan

### B1. 读被评审对象 + 找软肋

读要 grill 的 spec/plan。**grill 的靶子不是全文，是它的软肋**：

- spec 里自己标了"待定/倾向/候选/评估"的决策点；
- 与既定原则/裁决有**潜在张力**的地方（例：新增结构是否违反"消灭预设"、状态存哪是否违反"账本是唯一载体"）；
- 隐含的、没被论证的前提（"用户模型 X"是不是又一层预设？）。

好 grill 打这些，不打已经论证充分的部分。

### B2. 按固定骨架写 grill prompt

用 `references/grill-prompt-skeleton.md` 的骨架（已被验证既狠又躲得过 AUP）：

- **背景（给用户，不贴给 Fable）** + **PROMPT 正文（贴给 Fable）** 两段分明；
- 正文 = 系统设定（固定、纯抽象）→ 逐条要评的问题（每条瞄一个软肋，要"结论+判据"，标依赖链）→ 输出要求（含收敛句模板）；
- 全程纯抽象：系统/单元/内核/边缘/账本/状态通道，**零业务名词、零对抗词**。

### B3. AUP 自检 + 交付

过硬约束 1 的 AUP 自检（grep 禁用词全 0）。写进 `docs/superpowers/reports/YYYY-MM/YYYY-MM-DD-<topic>-grill-prompt.md`（若项目有此惯例）或用户指定处。告诉用户：「贴 PROMPT 正文整段给 Fable，背景段不贴。」

### B4.（可选）Fable 裁决回来后

用户把 Fable 裁决贴回来时，若是大设计，仿四批裁决那样落 decision-record（逐条裁决 + 收敛 + 对预判的颠覆），并据裁决修 spec。

---

## 产出

- 场景 A：一段"请 Fable 预答"的 prompt（交用户转发）→ 用户贴回 → 关键决策经 AskUserQuestion 收束 → decision-record。
- 场景 B：一份 grill prompt 文档（背景不贴 + PROMPT 正文）→ 用户转发 → Fable 裁决 →（可选）decision-record + 改 spec。

两者都**必过 AUP 自检**、都**以推翻预判为目标**。
