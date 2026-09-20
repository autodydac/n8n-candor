# Load & Concurrency Testing

This pipeline had, until this test, only ever been proven correct one synthetic candidate at a time. This is a condensed, public-safe summary of the real load and concurrency testing that closed that gap — every number below was read directly from the live pipeline's own execution records and observability log after the fact, not estimated or reasoned about from code alone.

## Intake pipeline: 86 real submissions, up to 50 concurrent

**Method.** Three real batches fired as concurrent, authenticated HTTP submissions against the live production form endpoint (the same entry point real candidates use) — not hand-constructed payloads.

| Batch | Size | Purpose |
|---|---|---|
| 1 | 28 | Stress the dedup gate under real concurrency, including several *simultaneous, byte-identical* submissions — a real concurrent-duplicate race |
| 2 | 8 | Stress the append-only write path under real concurrent *new* writes |
| 3 | 50 | Push past the initial 36 to find a more informative real ceiling |

**Result: zero failures, zero corruption, zero circuit-breaker trips.** All 86 intake executions and all 86 scoring executions completed successfully. No batch showed calls queueing or serializing as concurrency rose from 8 to 28 to 50 — the 50-candidate batch's full extraction fan-out completed in under 6 seconds wall-clock.

**Real cost and latency**, measured directly from the observability log:

| Batch | Candidates | Total cost | Cost/candidate | Latency (min/mean/max) |
|---|---|---|---|---|
| 1 | 28 | $0.3049 | $0.01089 | 2.9s / 3.7s / 5.7s |
| 2 | 8 | $0.0750 | $0.00938 | 2.9s / 3.0s / 3.1s |
| 3 | 50 | $0.5331 | $0.01066 | 3.0s / 3.8s / 9.0s |
| **Total** | **86** | **$0.9131** | **$0.01062** | **2.9s / 3.7s / 9.0s** |

Honest linear extrapolation from this single real data point (not a second measurement at these sizes): ≈$1.06 per 100 candidates, ≈$10.62 per 1,000, ≈$106 per 10,000 — against one job posting, at the current (unoptimized) architecture.

**A real, quantified inefficiency, found and documented rather than fixed under time pressure:** the job posting gets fully re-extracted by Claude on every single submission, even when 78 of the 86 submissions in this test ran against the exact same, unchanged posting as a prior submission. That redundant re-extraction accounted for 43% of this test's total spend. A concrete fix (content-hash-based caching, mirroring the resume dedup layer already built) was identified but deliberately not built in this pass — named here as an open, scoped item rather than silently left for someone to rediscover.

## Human-approval gateway: 15 real concurrent reviews

A separate real concurrent test targeted the human-approval workflow itself — the part of the system where a person reviews a briefing and records a decision — since the intake load test above didn't exercise it.

**A real platform-behavior finding, surfaced and classified, not silently worked around:** the review workflow's session mechanism is not a cookie or a hidden field — submitting page 1 returns a single-use, signed URL, and that URL *is* the session. The first concurrent-test attempt held one connection open per candidate waiting on that URL and every single one failed to receive a response within a 90-second timeout — yet the pipeline's own execution records showed all 15 underlying briefing-generation calls had actually succeeded in 6-9 seconds each. Reproduced at smaller scale with verbose tracing; confirmed a *fresh* request to the same URL after the result was ready returned instantly. **Classified as a test-methodology issue, not a defect in the pipeline itself** — every real execution's own output was correct throughout. Fixed by polling with short, repeated, fresh connections instead of one held-open request.

**Result, after the fix:** 15 of 15 concurrent reviews succeeded, 30.9 seconds wall-clock for the full batch. The decision log grew by exactly 15 real records, every one valid, every candidate ID distinct and correct — zero corruption or duplication despite 15 concurrent appends to the same file. Real cost: $0.1191 total, $0.0079/candidate, under the estimate stated before running. Real latency: 6.5-8.7 seconds per call, all 15 landing within a 2.2-second window of each other — no evidence of serialization or contention under true concurrency.

**No code change was needed in either the intake pipeline or the approval gateway as a result of this testing** — both held correctly under real, unoptimized load; the findings above are a redundant-cost inefficiency and a client-side test-methodology issue, both documented rather than hidden.

## Honest test-coverage gaps, stated plainly (as of the original test)

- ~~The circuit breaker's *tripping* behavior under concurrent *failures* was not re-tested at volume~~ **Closed in a follow-up pass** — see below: the full open→cooldown→recover lifecycle was proven end to end, not just its clean-closed state.
- ~~Whether a real browser client shares the long-held-connection behavior this test's script hit was not independently confirmed~~ **Closed in a follow-up pass** — see below: a real browser does not share it.
- These are still single-data-point measurements against one job posting and one synthetic candidate pool — not a statistically powered benchmark across varied job types or candidate volumes.

## Follow-up pass: circuit-breaker lifecycle, a real concurrency ceiling, and a real-browser confirmation

A later session closed both gaps named above, with real evidence, plus pushed concurrency further.

**Circuit-breaker full lifecycle**, proven with a local mock server (never the live provider) standing in for the Anthropic API, swapped in via a temporary credential and swapped back out afterward: 3 real threshold failures tripped the breaker open (its cooldown timer set to exactly the coded value); a 4th call was blocked in 31 milliseconds — the execution log shows the breaker's own check throwing before the API-calling node ever ran, and the mock server's own request log confirms it never received that call; after the real cooldown period elapsed, a recovery probe succeeded and reset the breaker to closed; normal traffic through the real credential was confirmed flowing again immediately after. Zero real provider cost for the failure-injection stages.

**A real concurrency ceiling, found and reported honestly.** Pushed to 60 concurrent submissions (above the original 50-concurrent baseline): all 60 succeeded with zero data loss or corruption, but per-execution completion time within that one submission burst ranged from 10.7 seconds to 49.1 seconds — a clear, real queueing signal, not noise. A subsequent attempt at 100 concurrent produced an inconclusive result: the test harness itself (many parallel local processes) didn't complete within its own timeout, and the live execution log confirms zero new work was even created server-side — a client-tooling limit, reported as exactly that rather than as a confirmed system ceiling.

**Real-browser confirmation of the human-approval gateway.** A full review was driven end to end in an actual browser (not a script) against a real, already-persisted candidate. The browser's own tab correctly sat on the multi-page form's long-poll URL and resolved on its own once the underlying work finished — no reload, no workaround. This directly answers the open question above: the long-held-connection behavior found earlier was specific to that session's scripted test client, not something a real reviewer using a real browser would ever encounter.
