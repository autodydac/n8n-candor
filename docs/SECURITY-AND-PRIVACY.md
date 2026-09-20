# Security & Privacy Design

## Candidate data

Every resume, job description, and candidate record used anywhere in this project's development and testing is **entirely fictional** — synthetic test fixtures, explicitly labeled as such inside the fixture text itself ("SYNTHETIC TEST FIXTURE — NOT A REAL PERSON... Any resemblance to a real person, employer, school, or certification body is coincidental"). No real candidate, real employer, or real job posting appears anywhere in this repository, either interface, or any of the underlying test data.

## No live connection from this repository

Neither the Candidate Console nor the Ops HUD connects to n8n, an API, or any other backend. Both interfaces embed a frozen, explicitly labeled, point-in-time snapshot of real (synthetic) pipeline output, generated once and frozen into the page. Opening either file makes zero network requests beyond loading its own fonts. This is a deliberate design choice, not an oversight — a live-connected demo would mean exposing real infrastructure (an n8n instance, API credentials, a database) to the public internet, which this project's own governance rules out.

## Credential handling

- Real deployment credentials (Anthropic API key, form Basic-Auth password) are read from environment configuration at runtime, never hardcoded into workflow definitions or scripts.
- `/workflows` contains 3 real, sanitized workflow exports (see the README for which and why). Credential IDs and the builder's real name were stripped from each before publication; no credential *value*, internal container port, or internal file path is included in any of them, or anywhere else in this repository — those are internal-infrastructure details with no public-demo value, and their absence is a deliberate sanitization decision, not an accident.
- A real secret scan (grep-based, checking for credential-shaped strings, API key patterns, and JWT patterns) was run against every file selected for this repository before publication — including a dedicated pass against `/workflows` specifically, since exported workflow JSON is exactly the kind of file that can carry a stray credential ID or internal reference — cross-checked specifically against the two credential names known to have been exposed once, earlier in this project's history, in an unrelated chat transcript (since flagged for rotation independently of this publication). The same scan was re-run against the actually-pushed repository after publication, not only the pre-push staged content.

## Scope limits as a security control

CANDOR's pipeline computes scores, evidence, and diagnostics — it does not, anywhere, execute a hiring action. No module can reject, advance, or contact a candidate. That's not just a policy statement; it's an architectural fact enforced by what the pipeline's own code is capable of doing. The human-approval gateway *records* a decision to a durable, append-only log — it has no code path that acts on that decision.

## Regulated-domain awareness

Employment screening is a regulated, high-risk category under most jurisdictions' AI/algorithmic-decision governance frameworks. This project was built with that awareness from the start: deterministic (not model-invented) scoring, a documented and enforced list of permitted scoring inputs with protected characteristics never collected at all, full evidence traceability on every claim, and a human decision required before any real-world action — the same posture a real regulated deployment would need, demonstrated here on synthetic data.

## What a real production deployment would still need

This is a portfolio/demonstration build. A real production deployment handling actual candidate data would additionally need: encryption at rest for persisted records, a real authentication/authorization layer for who can view candidate data, a data-retention and deletion policy, and jurisdiction-specific legal review of the scoring methodology — none of which are in scope for a synthetic-data demonstration project, and none of which are claimed to be solved here.
