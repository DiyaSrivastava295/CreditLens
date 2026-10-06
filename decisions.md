# CreditLens — Decision Log

## 2026-10-06 · Agent built in code, not Flowise

- **Decided:** The research agent is a Python Claude tool-use loop with typed, read-only tools and a pydantic memo schema. It is not built in Flowise.
- **Trade-off:** We give up a visual flow builder that's quick to demo. In return, the agent can be unit-tested and gated in CI, its read-only guarantee is visible in code review, and prompts and models are versioned in git.

## 2026-10-06 · Problem statement locked (M01 done)

- **Decided:** `01_problem.md` v1 is locked, and both team members agree. The separate 13 Oct project-lock call is no longer needed. The JD map and ownership table are in the problem statement (sections 8–9).
- **Still open, deliberately:** benchmark, universe rules, horizon and S3 licence. These are set in M03 on 24 Oct and don't change the problem.

## 2026-10-05 · Problem framing (M01 v0)

- **Decided:** CreditLens supports one monthly decision: which USD IG bonds to over- or underweight. That view drives a rules-based index, and a capital-protected note is built on the index.
- **Decided:** Success means beating simple baselines out of sample, net of costs. No target return or IC is set in advance.
- **Decided:** RAG and the agent form a read-only research layer. They never generate signals, trade or rebalance.
- **Decided:** We do the main Bloomberg pull on 12 Oct, broad, and filter it later. Missing fields can be pulled in a short top-up at the start of a later month, when credits renew (before 30 Nov; there is no Terminal in winter).
- **Trade-off:** We chose a cross-sectional relative-value signal over forecasting the direction of spreads. A ranking within the universe is easier to evaluate honestly (rank IC, out of sample) and maps directly onto index weights. The cost is that it says nothing about overall market direction, so the index carries market beta relative to cash.
