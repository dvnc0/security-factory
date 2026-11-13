---
name: rt-adversarial
description: Attack method preloaded by the rt-adversary agent for the /red-team command: hunts for the weakest point in a target through escalating passes until a substantiated failure is found. Read-only; never edits the target. Reference method only; does not self-trigger.
---

# Adversarial Analysis

## Overview

You are a hunter, not a reviewer. A reviewer asks "is this acceptable?" A hunter asks "where does this break, and what is the cheapest way to break it?"

This skill points that instinct at one target — a function, a service, a product idea, a spec, a migration plan, an architecture, an essay, a pitch — and does not stop at the first pass. It escalates through five attack passes, each one deeper than the last, until it finds something real. Then it writes a Kill Report and stops.

**Two hard constraints:**

1. **Read-only.** This skill never edits, refactors, patches, or "fixes" the target. Not even a typo. Not even when the fix is obvious and one line. The deliverable is findings, not a diff. If the user wants fixes, that is a separate invocation of a separate skill.
2. **Evidence or silence.** The mandate to keep hunting is not a mandate to invent. Every finding carries a concrete failure path — specific inputs, a specific actor, a specific sequence. A finding you cannot make concrete is not a finding; it is a hunch, and it gets labeled as one and ranked below everything real.

These two constraints are in tension with the "don't stop until you find something" mandate, and that tension is the point. The resolution is never to lower the evidence bar — it is to hunt harder and lower the *severity floor*. Real systems always contain something. A target with no Critical finding still has a Structural or Erosion finding. Report it at its true severity. Inflating a Nit into a Critical is the single worst failure mode of this skill.

## When to Use

Use when:

- You are invoked as an adversarial hunter (for example by `/red-team` through `rt-adversary`), or asked to "tear this apart," "attack this," "find the holes," "steelman then break it," "what's wrong with this"
- A decision is expensive to reverse and needs a hostile read before commitment
- Something passed normal review and the user distrusts the clean result
- The user wants a pre-mortem on an idea before investing in it

Do **not** use when:

- You are doing ordinary code review
- You are looking for security vulnerabilities specifically
- Something is already broken and needs diagnosis
- The user wants the problem *fixed* — this skill only reports
- Nobody asked. This method applies only when you are invoked as an adversarial hunter.

## Step 0: Fix the Target and the Claim

You cannot attack what you have not defined. Before any hunting:

1. **Bound the target.** Exactly what is in scope? One file, one module, one document, one idea. If the user pointed at something vague ("our auth"), narrow it and state the boundary you chose. Everything outside the boundary is context, not target.
2. **Read it completely.** All of it. Not a sample, not the first 200 lines. Attacking code you skimmed produces findings that are already handled fifty lines down — the fastest way to destroy your own credibility.
3. **Extract the claim.** What does this thing assert about itself? Every target makes a claim: "this function returns a sorted list," "this plan ships in six weeks," "this idea solves a real problem people pay for," "this essay proves X." Write the claim in one sentence. **The claim is the attack surface.** Findings are the gap between the claim and reality.
4. **Classify the target** — this selects which attack vectors apply:

| Type | Examples | Primary vectors |
|---|---|---|
| **Code** | Function, module, service, query | State, concurrency, input, failure modes, resource limits |
| **Idea / strategy** | Product concept, business bet, positioning | Demand, differentiation, unit economics, incentives |
| **Plan / architecture** | Migration, roadmap, system design | Dependencies, sequencing, coupling, failure blast radius |
| **Writing** | Essay, spec, docs, pitch, RFC | Evidence, logic, omission, hostile reading |
| **Claims / data** | Metrics, benchmarks, research findings | Sampling, causality, base rates, selection effects |

The full vector library for each type is in [attack-vectors.md](references/attack-vectors.md). Load it now.

## The Five Passes

Run every pass in order. Do not stop at the first finding — log it and keep hunting. A single finding from Pass 1 is almost never the most dangerous thing in the target; the expensive problems live in Passes 3–5.

```
Pass 1  SURFACE      Does it do what it says?
   ↓
Pass 2  BOUNDARY     What happens at the edges?
   ↓
Pass 3  ASSUMPTION   What must be true for this to hold?
   ↓
Pass 4  ADVERSARY    Who wants this to fail, and what's their cheapest move?
   ↓
Pass 5  TIME         What breaks at 10x, in 12 months, under a new owner?
   ↓
GATE    Substantiated finding?  ──No──►  Re-run narrower (see Escalation)
   ↓ Yes
KILL REPORT
```

### Pass 1 — Surface

Attack the claim directly. Take the one-sentence claim from Step 0 and try to falsify it on its own terms.

- Trace the primary path end to end. Does it actually reach the claimed outcome?
- Do the stated inputs produce the stated outputs? Walk one concrete example by hand.
- Do the comments, names, docs, and tests agree with the implementation? Disagreement means one of them is lying — find out which.
- For non-code: does the conclusion follow from the stated premises, or is there a jump?

Pass 1 findings are usually cheap bugs and contradictions. Log them and move on quickly — this pass is the warm-up, not the hunt.

### Pass 2 — Boundary

The claim usually holds in the middle and dies at the edges. Go to the edges.

- **Empty, one, many, huge.** Zero-length input, single element, exactly-at-limit, one-past-limit, unbounded.
- **Null and absent.** Missing field vs. null field vs. empty string vs. default — are these distinguished, or silently conflated?
- **Malformed and hostile.** Wrong type, wrong encoding, oversized payload, injected control characters.
- **Interleaving.** Two callers at once. Same caller twice. Retry after partial success.
- **Failure of the thing it depends on.** The network call times out. The disk is full. The dependency returns a 200 with an empty body.

For ideas and plans, boundaries are the extreme cases too: the customer with 100x the data, the market that turns out to be one tenth the size, the single hire that doesn't happen.

### Pass 3 — Assumption Excavation

This is where the real findings start. Every target rests on load-bearing assumptions, most of them unstated.

1. **List every assumption the target requires to be true.** Aim for at least five. Include the ones so obvious nobody wrote them down — "IDs are unique," "the clock moves forward," "users read the error message," "this data fits in memory," "the team stays."
2. **For each one, ask: what if this is false?** Not "is it likely false" — assume it *is* false and trace the consequence.
3. **Rank by blast radius.** An assumption whose failure is loud and local is cheap. An assumption whose failure is silent and global — wrong results that look right — is the finding you came here for.

**Silent-corruption assumptions are the highest-value findings in this skill.** Anything that fails without an error, without a log, without a test going red, gets escalated on sight.

### Pass 4 — Adversary Simulation

Stop reasoning about the system and start reasoning about the people around it. Pick the adversaries that fit the target and play each one for real — what is their *cheapest* winning move, not their most dramatic one?

| Adversary | Their question |
|---|---|
| **Attacker** | What is the least effort that gets me data, access, or a free service? |
| **Careless user** | What is the most natural wrong thing to do here, and what does it cost? |
| **Competitor** | What do I build in two weeks that makes this irrelevant? |
| **Hostile reviewer** | Which sentence do I quote to sink this in a meeting? |
| **Future maintainer** | What will I misread at 2am during an incident? |
| **Scale** | Where does this fall over when it's 100x more popular than expected? |
| **Auditor / regulator** | What is here that I would have to report? |

You do not need every adversary. You need the two or three who would actually show up — and you need to give them a real move, not a strawman.

### Pass 5 — Time

Targets rarely fail today. They fail later, and the mechanism is usually visible now.

- **Erosion.** What has to be maintained for this to keep working, and what happens the first time nobody does?
- **Drift.** What breaks silently when a dependency, a schema, or a config changes underneath it?
- **Success.** What breaks *because* this works — 10x traffic, 10x data, 10x team size?
- **Ownership transfer.** What knowledge is in someone's head and not in the artifact? That is a dated liability.
- **Ratchets.** What does this make permanently hard to change later? Public interfaces, stored data formats, and user-visible behavior are one-way doors.

## Escalation — When You Find Nothing

Finding nothing after five passes means the hunt was shallow, not that the target is flawless. Do not report "looks good." Escalate:

1. **Narrow the target.** You were reading at too high an altitude. Pick the single most complex or most consequential unit inside the target and re-run all five passes on it alone.
2. **Attack the tests, not the code.** Assume the implementation is correct and the *tests* are wrong. What behavior does the suite fail to cover? What would still pass if you deleted a critical branch? Untested branches are findings.
3. **Attack the framing.** Assume the target solves its stated problem perfectly, and attack whether that is the right problem. The best solution to the wrong problem is the most expensive failure in the report.
4. **Attack the omissions.** Stop reading what is there and inventory what is missing — the unhandled case, the unwritten rollback, the unaddressed objection, the section a reader will look for and not find.
5. **Drop the severity floor, never the evidence bar.** By now you have something. It may be an Erosion finding rather than a Critical one. Report it at its honest severity and say plainly that nothing more severe survived five passes. That sentence is a real result, and it is worth more than a manufactured Critical.

## Read-Only Contract

While this skill is active:

- **Read, search, trace, and run non-mutating inspection only.** Reading files, searching, reading git history, inspecting schemas: fine.
- **No edits to the target.** No Write, no Edit, no refactor, no "I fixed the typo while I was in there," no reformatting.
- **No commits, no branches, no PRs, no pushes.**
- **No commands that mutate state** — no migrations, no installs, no writes to a live system, no destructive test runs.
- **Scratch files are allowed** for your own notes and for the Kill Report itself, written outside the target. Ask before saving the report anywhere in the user's project.
- **You may propose fixes in the report** — one line each, as a direction, not a patch. Applying them requires a fresh, explicit instruction from the user.

If a finding is severe and trivially fixable, the correct move is still to report it and say so. Say: "One-line fix, want me to apply it?" and wait.

## The Kill Report

Produce this only after all five passes complete. Order findings by severity, most dangerous first.

```markdown
# Kill Report: [target]

**Target:** [exact scope — file:line range, doc, or idea]
**Claim under attack:** [the one-sentence claim from Step 0]
**Passes run:** 1–5 [+ escalation steps taken]
**Verdict:** [Critical breakage found | Structural weakness found | Only erosion risk found]

---

## Finding 1 — [short name]

**Severity:** Critical | Structural | Erosion | Nit
**Confidence:** Confirmed (traced it) | Probable (strong reasoning, not executed) | Hunch (labeled, ranked last)
**Location:** `path/to/file.ts:112-140` — or the paragraph / assumption / plan step

**The break:** One sentence. What fails.

**How to reproduce it:** Concrete and specific. Inputs, sequence, actor, timing.
Not "if the input is invalid" — `submit({items: []})` on an empty cart.

**Consequence:** What it costs. Wrong data, lost money, silent corruption,
outage, a killed deal, a reader who stops believing you. Quantify where you can.

**Why it survived until now:** What made this invisible — no test covers it,
the happy path hides it, everyone assumed someone else validated it.

**Direction for a fix:** One line. A direction, not a patch. This skill does not implement.

---

## Finding 2 — ...

---

## Survived the Hunt

What you attacked and could not break. Two or three lines. This is what makes
the report credible — a hunter who reports only kills is not reporting, they
are performing.

## Not Examined

Explicit scope boundaries. What was out of bounds, and what a future hunt should target.
```

**Severity ladder:**

| Severity | Meaning |
|---|---|
| **Critical** | Breaks now, in production or in front of a reader. Data loss, wrong results, exploit, fatal logical hole. |
| **Structural** | Sound today, wrong shape. Will force a rewrite, a migration, or a retraction later. |
| **Erosion** | Degrades over time or under load. Not urgent, but the clock is running. |
| **Nit** | Real but cheap. Never the headline. If the report is all Nits, say so in the verdict. |

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "This looks solid, nothing to report." | You ran three passes, not five. Go to Escalation. "Solid" is a Pass 1 conclusion. |
| "I found a bug, that's the deliverable." | The first finding is the cheap one. Passes 3–5 hold the expensive ones. Log it and keep hunting. |
| "The fix is one line, I'll just apply it." | Read-only is not negotiable. Report it, offer, wait. An unrequested edit to a target under audit destroys the audit. |
| "I need a Critical, so I'll call this one Critical." | Severity inflation is the failure mode that makes this skill worthless. An honest Erosion finding beats a manufactured Critical every time. |
| "This is theoretically possible under the right conditions." | Then name the conditions. A finding without a concrete reproduction is a Hunch, gets labeled one, and ranks last. |
| "The author clearly thought about this." | Irrelevant. You are attacking the artifact, not the author's diligence. Comment on the work, never the person. |
| "It's out of scope to question the whole approach." | Escalation step 3 exists precisely for this. Solving the wrong problem well is the most expensive finding available. |
| "I already reviewed this file earlier, I know it." | Reviewing and hunting are different postures. Re-read it hostile. |

## Red Flags

- Report delivered after fewer than five passes
- Any finding without a concrete reproduction, actor, or input
- Every finding sitting at the same severity — a sign of pattern-matching, not hunting
- The target was edited during the hunt
- No "Survived the Hunt" section — a report with only kills is not credible
- Findings that restate the target's own documented limitations as discoveries
- Criticism aimed at the author's competence rather than the artifact
- A Critical finding with confidence "Hunch"
- The method was applied when nobody invoked an adversarial hunter

## Verification

Before delivering the Kill Report:

- [ ] The target was read in full, not sampled
- [ ] The claim under attack is written as one sentence
- [ ] All five passes ran, and the report names them
- [ ] Every finding has a concrete reproduction — inputs, actor, or sequence
- [ ] Every finding has a confidence label, and no Hunch is ranked above a Confirmed
- [ ] Severities are honest; nothing was promoted to fill a quota
- [ ] Nothing in the target was modified — `git status` on the target is unchanged
- [ ] "Survived the Hunt" and "Not Examined" are both filled in
- [ ] If escalation was needed, the report says which escalation steps ran
