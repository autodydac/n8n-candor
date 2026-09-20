# CANDOR — Architecture

Condensed, public-safe summary of the pipeline's 10 modules. This distills the original internal engineering logs (session-by-session build notes, several hundred entries across per-module files) down to what each module does, how it's built, and what's actually verified — internal-only detail (workflow IDs, credential IDs, container ports, raw session-by-session narrative) is intentionally omitted.

## Pipeline overview

```
                    ┌─────────────────────────────────────────┐
                    │           CANDOR-02: Source Adapters      │
                    │  file-upload · application-form · ATS-export │
                    └───────────────────┬───────────────────────┘
                                        ▼
                    ┌─────────────────────────────────────────┐
                    │   CANDOR-01/03: Intake & Claim Extraction  │
                    │   (Claude Haiku — extract, don't guess)    │
                    └───────────────────┬───────────────────────┘
                                        ▼
                    ┌─────────────────────────────────────────┐
                    │  CANDOR-04/05: Deterministic Scoring       │
                    │  + Bias/Fairness Audit Record              │
                    │  (code-computed, 7 permitted factors)      │
                    └───────────────────┬───────────────────────┘
                                        ▼
          ┌─────────────────────────────┼─────────────────────────────┐
          ▼                             ▼                             ▼
┌──────────────────┐      ┌──────────────────────┐      ┌──────────────────────┐
│ CANDOR-08:         │      │ CANDOR-06:             │      │ CANDOR-09:             │
│ Persistence + Dedup│      │ Pool Statistics,       │      │ Alerts / Reporting     │
│ (append-only JSONL)│      │ Bias Audit, Clustering │      │ (Python digest)        │
└──────────────────┘      └──────────────────────┘      └──────────────────────┘
          │
          ▼
┌──────────────────────┐      ┌──────────────────────────────┐
│ CANDOR-07:             │ ───▶ │ CANDOR-10: Human Approval &    │
│ Recruiter Briefing     │      │ Action Gateway                 │
│ (Claude Sonnet + QA)   │      │ (records decisions, executes   │
│                        │      │  nothing — Section 13)         │
└──────────────────────┘      └──────────────────────────────┘
```

## Module summary

| # | Module | What it does | How it's verified |
|---|---|---|---|
| 01 | Intake & Parsing Core | Claude (Haiku) extracts structured candidate claims from raw resume text — instructed to report "insufficient information" rather than fabricate, and to quote the source sentence backing each claim. | Behind a circuit breaker, retry logic, request-level observability logging, and a dead-letter queue for failed extractions. |
| 02 | Source Adapters | Three real intake paths into the same pipeline: direct file upload, a structured application-submission form (skips LLM extraction — nothing to extract from already-structured fields), and a batch ATS-export file (mixed record shapes in one batch, per-record error isolation, in-batch dedup). | Each adapter authenticated (HTTP Basic Auth); zero-regression checks re-run on every commit via a git pre-commit hook. |
| 03 | Claim Extraction & Evidence Verification | Cross-checks each extracted claim against the source resume text before it's trusted downstream. | Same workflow as module 01, by design — not a separate system. |
| 04 | Requirement Matching & Deterministic Scoring | Computes Match Score (0-100) from up to 7 permitted factors: years of experience, required-skill overlap, role-tenure stability, career trajectory, education/certification match, portfolio/work-sample match — each domain-gated so a factor is only scored when the job posting actually requires it, and each factor's contribution is code-computed, never left to an LLM to invent. A separate Evidence Confidence score (0-100) measures how much of the record rests on independently-verified claims. Also caches job-posting extractions by content hash, so a repeat submission against an unchanged posting reuses the prior extraction instead of paying for a new one — a real fix for a real inefficiency found in load testing (43% of one test's spend was redundant re-extraction of the same posting). | 7-factor build verified against a real candidate pool at each step; real bugs (an OR-logic scoring error, a matched-skills under-reporting bug) were found and fixed during this build, not assumed correct. The extraction cache was verified 3 ways on the live instance: byte-identical scores on a cache hit vs. the original miss, a real 7-submission/2-distinct-posting test showing only 2 real extraction calls, and a genuinely different posting still triggering a fresh extraction. The real node chain (`workflows/CANDOR-04.json` in this repo) is the same one that ran that test. |
| 05 | Bias & Fairness Audit Layer | Attaches a full audit record to every scored candidate: which fields were used in scoring, which protected/personal fields were present but excluded, and an empirical check for whether an excluded field's value leaked into what was actually scored. | Same workflow as module 04 — the audit record is built immediately after the score, from the same run. |
| 06 | Comparative Ranking & Clustering | Pool-level statistics (score distributions, outlier detection via Tukey's IQR fences), a qualification-profile clustering pass (explicitly not a black-box leaderboard — threshold-bucketed on real scoring axes, with its own coarseness limitations stated), and a bias-audit report on required-skill match rates across the pool. | Plain Python, no dependencies; re-run against real pool data, output included in this repo's own audit doc. |
| 07 | Recruiter Briefing Generator | One candidate's audit record → one plain-English briefing (Claude Sonnet — the one step where phrasing nuance justifies the cost over Haiku). 5 hard, non-negotiable constraints: no recommendation however indirect, both scores always shown together, the reliability caveat is prominent when evidence confidence is low, every claim traceable to the record. | A second, independent Claude call reviews the generated briefing against the same 5 constraints before it reaches a human — verified against both a compliant and a deliberately-violating test case. |
| 08 | Database / Historical Intelligence | Append-only JSONL persistence of every scored candidate record — never overwritten, never mutated. Exact-match dedup (job + resume-content-hash) gates duplicate writes; a separate near-duplicate fuzzy-match layer flags close-but-not-identical resubmissions. | Verified under real concurrency up to 50 simultaneous submissions, including deliberate exact-duplicate races — zero corruption, correct dedup math every time. |
| 09 | Alerts / Reporting | A digest script flags low required-skill match rates (possible miscalibrated job requirement) and candidates sitting in the "insufficient information" bucket with no recorded human decision yet. | Plain Python, re-run on demand against real pipeline state. |
| 10 | Human Approval & Action Gateway | A two-page form: page 1 takes a candidate ID, page 2 shows that candidate's real, freshly-generated briefing and records a human decision (ADVANCE / REJECT / HOLD — MORE INFO NEEDED) plus decider identity and notes. Never executes the decision — only records it, append-only. | Verified under real 15-way concurrent load: correct session isolation, zero corrupted or duplicated decision records. |

## Design principles enforced throughout

- **Deterministic scoring, LLM-assisted extraction.** The Match Score itself is always code-computed from structured inputs — Claude's job is limited to extracting and verifying claims from unstructured text, never to inventing a score.
- **Evidence Confidence is structurally separate from Match Score, everywhere.** Never merged into one number, in any record, any script output, or either interface.
- **A documented exclusion list, enforced and checked.** Only a fixed set of fields (years of experience, key skills, education, certifications) are permitted scoring inputs. Every scored record includes an empirical check for whether an excluded field's value (name, contact info, prior title, prior employer) ever leaked into the actual scoring computation.
- **Nothing executes.** No module in this pipeline is capable of rejecting, advancing, or contacting a candidate — that decision belongs to a human, every time, recorded not automated.
