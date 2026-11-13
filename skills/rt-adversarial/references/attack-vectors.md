# Attack Vectors by Target Type

Vector library for the `rt-adversarial` skill. Load the section matching the target type classified in Step 0. These are prompts for the hunt, not a checklist to tick — a vector that produces nothing after honest effort is a dead end, not a finding.

---

## Code

### State and lifecycle
- What is the state before this runs, and who guarantees it? What if the guarantor is wrong?
- Can this run twice? Is it idempotent, or does the second run corrupt the first?
- Is there a partial-success path — three of five writes land, then it throws? What is left behind?
- Is cleanup guaranteed on every exit path, including the exception path and the early return?
- Is any state cached? What invalidates it, and what reads it after it goes stale?

### Concurrency
- Two callers at the same instant: read-modify-write, check-then-act, or lazy-init — any of these unguarded?
- Is a lock held across an await, an I/O call, or a callback?
- What is the ordering guarantee between related operations, and what depends on it?
- Retries: is the retried operation safe to repeat, or does it double-charge?

### Input
- Empty, single, exactly-at-limit, one-over-limit, absurdly large.
- Null vs. undefined vs. empty string vs. missing key vs. default — are they distinguished or silently merged?
- Unicode, control characters, RTL marks, emoji, four-byte characters in a three-byte column.
- Numbers: negative, zero, float where int expected, precision loss, overflow, `NaN`.
- Time: DST transitions, leap seconds, timezone-naive values, clock skew, a timestamp from the future.
- Where does untrusted data cross a boundary without validation?

### Failure modes
- Every external call: what happens on timeout, on 500, on 200-with-garbage, on a hung connection with no timeout set?
- Are errors swallowed? Grep for empty catch blocks and bare `except`.
- Does a failure here leave the system in a state a retry can recover from?
- Is there a failure that produces a *wrong answer* rather than an error? Escalate on sight.

### Resources
- Unbounded growth: a list, cache, queue, or map that only ever gets appended to.
- Connections, file handles, subscriptions, timers — opened where, closed where?
- N+1 queries; a loop with I/O inside it; a query with no LIMIT on a table that grows.
- What is the memory footprint at 100x the current data size?

### Tests as target
- Which branch would survive being deleted with the suite still green?
- Do the tests assert behavior, or do they assert the implementation back to itself?
- Are tests mocking the exact thing that would break in production?
- Is there a test that has never failed and never could?

---

## Idea / Strategy

### Demand
- Who has this problem *badly enough to pay*? Name them specifically, not as a segment.
- What are they doing instead today? "Nothing" usually means the problem is not painful.
- Is this a vitamin or a painkiller? Vitamins do not survive a budget cut.
- Is the buyer the same person as the user? If not, the pitch has two audiences and probably serves neither.

### Differentiation
- What stops the incumbent from shipping this as a checkbox feature in one quarter?
- If it is easy to build, why hasn't it been built? If it is hard, why is *this* team the one that can?
- What is the moat after copying — data, distribution, switching cost, network effect? "Execution" is not a moat.

### Economics
- What does one unit cost to serve, including the parts that are free today and won't be at scale?
- How is a customer acquired, and does the cost of acquiring one exceed what they are worth?
- Which cost line grows linearly with usage while revenue is flat-rate?

### Incentives
- Who inside the customer's organization loses if this succeeds? They will fight it.
- What does this ask a user to *stop* doing? Behavior change is the most-underpriced cost in any plan.
- Does anyone in the chain get paid for the current broken way?

### Pre-mortem
- It is twelve months out and this failed. Write the one-sentence obituary. Now trace back: what was the earliest visible symptom?

---

## Plan / Architecture

### Dependency and sequencing
- What is on the critical path, and what happens to the whole plan when it slips two weeks?
- Which step assumes another team, vendor, or approval delivers on time?
- Is there a step that cannot start until another finishes, that the plan runs in parallel anyway?
- What has no rollback? A one-way door disguised as a normal step is the finding.

### Blast radius
- When this component fails, what else stops? Draw the fan-out.
- Is there a single point of failure the plan treats as reliable infrastructure?
- Does a failure degrade gracefully, or does it cascade?

### Coupling
- What does this make permanently hard to change? Stored data formats and public interfaces are the usual culprits.
- How many components must change together to ship one feature? That number is the real cost of the design.
- Is there a shared mutable resource multiple owners write to?

### Estimation
- Which estimate assumes nothing goes wrong? All of them, usually — name the two most fragile.
- What work is not in the plan at all: migration, backfill, monitoring, docs, on-call, the deprecation of the old thing?
- Is there a step described in one line that is actually a month?

### Reversibility
- If this is wrong, when do we find out, and what does it cost to undo at that point?
- Is there a smaller, cheaper version that would surface the same information sooner?

---

## Writing

### Evidence
- Which claim carries the most weight and has the least support behind it?
- Is a citation actually saying what the sentence says it says?
- Are numbers sourced, dated, and comparable — or borrowed from a different context?
- Where does an anecdote stand in for data?

### Logic
- Where does the argument jump — a "therefore" that is really an "and also"?
- Is the strongest counterargument addressed, or the most convenient one?
- Does a definition shift between the beginning and the end?
- Does the conclusion actually follow, or does it merely feel earned by the buildup?

### Omission
- What would a knowledgeable reader expect here and not find? That absence is the finding.
- What is the risk, cost, or caveat that goes unmentioned?
- Who is affected by this and never appears in the text?

### Hostile reading
- Which single sentence gets quoted out of context to discredit the whole piece?
- Where would a skeptical reader stop reading, and why?
- What does this look like to the person it is implicitly criticizing?

### Craft
- Does the first paragraph earn the second? Does the reader know why they should care within the opening?
- Where does the prose hedge so heavily that it stops asserting anything?
- Is the structure serving the reader, or the author's drafting order?

---

## Claims / Data

### Measurement
- What exactly was measured, and is it the thing the claim is about — or a proxy for it?
- What is the denominator? Missing denominators are the most common lie in a metric.
- Over what window? Does a different window reverse the story?

### Sampling
- Who is missing from the sample, and would they behave differently? Survivors, churned users, non-responders.
- Is the sample self-selected?
- Is the effect size large enough to survive the noise, or is n small enough that one outlier drives it?

### Causality
- Is a correlation being narrated as a cause? What third factor drives both?
- What else changed at the same time as the intervention?
- Is the comparison group actually comparable?

### Presentation
- Is the axis truncated, the scale logarithmic, the color scheme steering the eye?
- Are percentages hiding tiny absolute numbers, or absolute numbers hiding a tiny rate?
- Was the metric chosen after the data was seen?

### Benchmarks specifically
- Was the benchmark run on a warm cache, a quiet machine, or a hand-picked dataset?
- Is the baseline configured as carefully as the thing being promoted?
- Does the measured workload resemble the real one?
