# Bias & Fairness Audit

CANDOR's approach to fairness auditing is scoped deliberately narrow, and that scope is stated explicitly rather than implied to be broader than it is.

## What this system can check — and does

CANDOR never collects race, gender, age, disability status, or any other protected characteristic at any point in the pipeline. That's a permitted-fields design decision, not an omission: only years of experience, key skills, education, and certifications are ever passed into scoring, and every scored record carries an empirical check confirming that a fixed list of excluded fields (candidate name, contact info, prior job title, prior employer) never leaked into what was actually computed.

Because that data is never collected, **disparate-impact analysis by protected characteristic is structurally impossible with this system's own data — not deferred, not a future roadmap item, impossible by the design itself.** That's stated plainly here rather than glossed over.

What the system *can* check, and does:

**Real per-required-skill match-rate analysis across the pool** — run against a real 6-candidate synthetic pool:

| Required skill | Matched by | Rate |
|---|---|---|
| HubSpot | 1 of 6 | 17% |
| Salesforce | 2 of 6 | 33% |

A low match rate like this is flagged as worth a human look — it may indicate a miscalibrated job requirement (asking for a specific tool rather than the underlying skill), not necessarily a weak candidate pool. The system reports the pattern; it does not draw the conclusion.

**Factor-component distribution analysis** — checking whether the two most-populated scoring factors (years-of-experience sufficiency, skill overlap) show a meaningfully different distribution across the pool. On the real 6-candidate pool, the observed gap was disclosed honestly as **not statistically meaningful at this sample size** — reported for visibility, not presented as a finding.

**Qualification-profile clustering** — a real 6-candidate pool groups into 4 distinct profiles (matched+sufficient, matched+partial, not-matched+insufficient, and a 3-member not-matched+unknown cluster). The clustering method's own real limitation is stated alongside the result: the largest cluster groups candidates whose only proven similarity is "no verified required skill and no extractable years claim" — it does not mean their underlying resumes are actually similar to each other, and the clustering script's own documentation says so.

## An honest disclosure about sample size

Every number above comes from a 6-candidate synthetic pool — small enough that no statistical claim should be drawn from it with confidence. That limitation is stated here the same way it's stated in the underlying audit scripts' own output, not smoothed over for a cleaner-looking report.

## An honest disclosure about a self-check false positive

CANDOR's excluded-field leak check is an empirical, runtime check — it searches each excluded field's real value against what was actually passed into scoring on that run. During this build, that check produced what appears to be a false positive on a small number of real records: a legitimately-quoted piece of verification evidence (a source-text quote proving a years-of-experience claim) happened to contain words that also appeared in an excluded field (the candidate's job title), triggering the leak flag even though the excluded field's *value* was never itself used as a scoring input. This is disclosed transparently in the Candidate Console interface — the flag is shown, not hidden, on the one curated demo candidate where it occurred — rather than quietly filtered out of the public demo pool to look cleaner than the system actually is.

## What this section is not

This audit is a diagnostic aid, not a certification. Nothing produced by CANDOR's bias-audit layer constitutes a legal determination of compliance with employment-discrimination law, and no combination of these numbers authorizes any hiring decision. That's the same language stated on every individual candidate's own audit record, repeated here at the pool level.
