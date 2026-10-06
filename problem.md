# CreditLens — Problem Statement

**Version:** v1 (locked 6 Oct 2026, agreed by both team members)
**Change policy:** any change to this document is recorded in `docs/decisions.md` with a reason. The parameters in §12 are set in M03 (24 Oct) and do not reopen the problem.

---

## 1. Summary

CreditLens is a **systematic credit relative-value strategy for USD investment-grade (IG) corporate bonds**:

- **Signal:** every month, a machine-learning signal ranks bonds by whether their credit spreads look cheap or expensive compared with similar bonds.
- **Index:** a published, rules-based index turns those rankings into portfolio weights.
- **Evaluation:** we test the index walk-forward on history, net of trading costs, and measure its risk under stress.
- **Product:** we package it as a capital-protected note (zero-coupon bond + call option on the index), priced with its Greeks and hedge behaviour.
- **Platform:** the system runs on AWS.
- **Data is a historical snapshot, not a live feed:** Bloomberg is extracted once from the Terminal to a laptop (history up to the 12 Oct 2026 pull). "Live" monthly operation is demonstrated by replaying history month by month, as if each month were today.
- **Research layer:** an AI research layer (RAG + a read-only agent) explains the numbers from company filings. It never makes the investment decision.

---

## 2. Context

- A corporate bond pays a **credit spread**: extra yield over a government bond of the same maturity, as compensation for default and liquidity risk. In IG credit the standard measure is the **option-adjusted spread (OAS)**, quoted in basis points (1 bp = 0.01%).
- Spreads of similar bonds (same rating, sector and maturity) often differ:
  - **Part of the gap is justified**, by leverage, earnings trend, liquidity or event risk.
  - **Part of the gap is mispricing.** It comes from forced selling, index flows, rating-agency lag and investor inattention, and tends to correct over the following months.
- Credit investors care about **relative value (RV)**: which bonds will outperform their peers, regardless of where the whole market goes. This is how credit RV desks and systematic credit funds work. It is a *cross-sectional* problem: rank bonds against each other at each point in time.
- **Spread duration** turns spread moves into returns: a bond with spread duration 6 gains about 0.6% if its spread tightens by 10 bp. Performance is measured as **excess return**, the bond's return minus the return of duration-matched Treasuries. That isolates the credit component.

## 3. Problem

> **Each month, which USD IG corporate bonds should be overweighted or underweighted relative to the benchmark, based on whether their spreads are cheap or expensive versus comparable bonds — and can that view be delivered as a transparent, investable, risk-managed product?**

This breaks into four sub-problems:

1. **Measurement.** Can we build a point-in-time-correct dataset where every historical decision uses only information available on that date?
2. **Prediction.** Can a model rank bonds by future excess spread change better than simple rules (raw spread, carry, spread z-score within rating bucket)?
3. **Implementation.** Does the ranking survive realistic constraints once turned into an index: issuer and sector caps, turnover, and trading costs?
4. **Productisation and risk.** What is the index's risk profile under stress? How would a bank structure, price and hedge a capital-protected note on it?

## 4. Why it matters

- **Investors and index providers:**
  - Systematic credit and fixed-income factor indices are a growing area.
  - Credit is harder than equities: bonds trade over-the-counter, data is patchy, and costs are high.
  - A disciplined, cost-aware approach is valuable precisely because naive backtests in credit are usually misleading.
- **Banks:**
  - The same pipeline covers research (signal), structuring (note pricing and hedging) and risk (exposure and stress).
  - These are the three groups that would touch a product like this in practice.
- **Governance:** credit models feed real capital decisions. Reproducibility, point-in-time data, auditability and keeping AI out of the decision loop are requirements, not extras.

## 5. Hypothesis

Bond spreads contain a mispricing component relative to peers that partially corrects over the following one to three months. A model using peer-relative spread, spread momentum, carry, issuer fundamentals and their trends, rating drift and the macro regime can rank bonds by future excess spread change better than single-variable rules.

**Falsifiable:** if rank IC is not consistently above the baselines out of sample, or the edge disappears after costs, the hypothesis is rejected and we report that.

## 6. Objectives

| # | Objective |
|---|---|
| O1 | Build a point-in-time bond-month dataset from Bloomberg, FRED and SEC EDGAR, with validation and lineage |
| O2 | Build and evaluate a cross-sectional ML relative-value signal with walk-forward testing |
| O3 | Define and implement a rules-based index (eligibility, weighting, caps, rebalance, turnover buffer) |
| O4 | Backtest the index net of costs against a benchmark and simple baselines, with attribution |
| O5 | Measure portfolio risk (exposures, DTS, VaR/ES) and stress-scenario P&L at each rebalance |
| O6 | Structure, price (Black-Scholes and Monte Carlo) and hedge a capital-protected note on the index |
| O7 | Run the pipeline on AWS (scheduled backtest, read-only research API, monitoring) |
| O8 | Provide a RAG + agent research layer over SEC filings that explains issuers with citations, evaluated for accuracy |

## 7. Users

| User | Question | What CreditLens provides |
|---|---|---|
| Credit PM / RV analyst | "Where is the value this month, and why?" | Ranked signal, drivers per bond, cited issuer memo |
| Index desk / structurer | "Can we publish and sell a product on this?" | Rulebook, holdings, turnover, costs; note term sheet, price, participation rate, Greeks, hedge simulation |
| Portfolio risk | "What can go wrong and how much can we lose?" | Exposures by rating/sector, DTS, VaR/ES, stress scenarios on index and note |

## 8. Solution design

```
Bloomberg (bonds, spreads, ratings, fundamentals) + FRED (rates, VIX, IG/HY OAS) + SEC EDGAR (filings)
   │
[1] Data layer       bronze → silver → gold (Parquet + DuckDB); Pandera validation; point-in-time joins
   │
[2] ML signal        features → ridge / gradient boosting → monthly cross-sectional ranking; walk-forward
   │
[3] Strategy & index rulebook converts ranks into weights within caps; monthly rebalance; turnover buffer
   │
[4] Backtest         month-by-month replay net of costs; benchmark + baselines; attribution; bootstrap CIs
   │
[5] Risk & stress    exposures, DTS, VaR/ES, scenario P&L for index and note
   │
[6] Structured note  zero-coupon bond + call on index; Black-Scholes + Monte Carlo; Greeks; delta-hedge simulation

[7] AWS platform     S3 data lake · ECS Fargate + EventBridge (monthly run) · API Gateway + Lambda (read-only API) · CloudWatch
[8] Research layer   RAG over 10-K/10-Q (as-of filtered, cited, refuses when unsupported) + read-only tool-calling agent → issuer memo
```

### Layer details and the reasoning behind them

**[1] Data**

- **What:**
  - Bond-month panel: OAS, price, spread duration, rating, sector, amount outstanding, maturity.
  - Issuer fundamentals with their report dates.
  - Macro series from FRED; filing text from EDGAR.
- **Why point-in-time:** look-ahead is the most common way credit backtests overstate results. A fundamental is usable only after its report date.
- **Why keep matured and downgraded bonds:** dropping them causes survivorship bias. The survivors look better than the real investable universe was.
- **Why Parquet + DuckDB:** the data is analytical and fits on one machine. A columnar file plus an in-process SQL engine is fast and simple, with no server to run.

**[2] ML signal**

- **Target:** forward excess spread change or excess return, *ranked within each month*.
- **Why rank within each month:** that removes the market-wide move. We are predicting relative, not absolute, performance.
- **Features:**
  - spread vs peers (rating, sector and maturity bucket);
  - spread momentum and carry;
  - leverage and coverage levels and trends;
  - rating drift;
  - macro regime interactions.
- **Models:** ridge regression (interpretable baseline) and gradient boosting (captures non-linearity and interactions).
- **Evaluation:**
  - **Walk-forward:** train on the past, test on the next period, roll forward, with a 3-month embargo so overlapping targets can't leak.
  - **Metrics:** rank IC and hit rate, which match how the signal is used (ranking) better than RMSE.

**[3] Strategy and index**

- **Construction:** benchmark weights tilted by signal rank, subject to issuer and sector caps and a tracking-error budget, with a turnover buffer and a monthly rebalance.
- **Why rules-based:** anyone with the data and the rulebook reproduces the same holdings. That makes it auditable and investable, and keeps judgement (human or AI) out of the monthly decision.

**[4] Backtest**

- **Mechanics:** replay each month, charge estimated transaction costs (bid-ask by rating and size), and compare against the benchmark and the simple baselines.
- **Reported:** net excess return, information ratio with a bootstrap confidence interval, drawdown, tracking error, turnover, and attribution (sector, rating, spread-duration and selection effects).
- **Overfitting control:** we record how many variants were tried, since multiple testing inflates results.

**[5] Risk and stress**

- **Measures:** exposures, DTS (duration × spread), and historical/parametric VaR and Expected Shortfall at each rebalance.
- **Scenarios:** 2020-style spread blow-out, BBB downgrade wave, and parallel rate shifts, applied to both the index and the note.
- **Why DTS:** it scales spread risk by spread level. Wider-spread bonds move more in sell-offs, which duration alone misses.

**[6] Structured note**

- **Structure:**
  - Zero-coupon bond = 100% capital protection at maturity.
  - The remaining budget buys a call on the index.
  - **Participation rate** = option budget ÷ option price. It rises with interest rates and falls with volatility.
- **Pricing:** Black-Scholes, checked against Monte Carlo and put-call parity.
- **Hedging:** compute Greeks (delta, gamma, vega, theta), then simulate delta hedging on the backtested index path to see where hedge P&L leaks (gamma, volatility mismatch, rebalancing frequency).
- **Known limitation:** Black-Scholes assumes lognormal, constant-volatility dynamics. We test this and report it.

**[7] AWS**

- **Storage:** S3 for processed data. Raw Bloomberg data goes there only if the licence permits.
- **Compute:** a containerised backtest + rebalance job on ECS Fargate. EventBridge triggers it on a schedule to show how it would run in production; because the data is a static snapshot, each run replays the next historical month ("as-of replay") rather than pulling new prices. Public FRED and EDGAR data can refresh for real.
- **API and monitoring:** a read-only research API on API Gateway + Lambda, and CloudWatch logs, metrics and alarms. Everything is defined in IaC with budget alarms.
- **Why Fargate, not Lambda, for the backtest:** a long, memory-heavy batch job in a container fits Fargate. The API's short request/response fits Lambda.

**[8] Research layer (RAG + agent)**

- **RAG:**
  - Searches 10-K/10-Q filings with section-aware chunking and hybrid retrieval (BM25 + vectors + reranking).
  - Filters to filings published before the as-of date and cites sources. If the filings don't support an answer, it refuses.
- **Agent:**
  - Calls read-only tools: issuer snapshot, signal rank, RAG search, risk report, scenario run, note price.
  - Produces a structured, validated issuer memo.
- **Evaluation:**
  - **RAG:** recall@k, MRR, faithfulness.
  - **Agent:** tool-call correctness, numeric accuracy vs tool outputs, cost and latency.
  - **Security:** prompt-injection tests.

## 9. Guardrails

1. **RAG never generates trading signals.** Text explains; it does not decide.
2. **The agent never trades or rebalances.** All its tools are read-only.
3. **Rules are deterministic and versioned.** Same data + same rulebook = same index.
4. **No look-ahead, no survivorship.** Point-in-time joins, embargoed walk-forward, dead bonds retained, enforced by tests.
5. **Licence compliance.** Raw Bloomberg data is never committed or published. The repo holds derived data, code and provenance only.
6. **No invented numbers.** No target return or IC set in advance, and no performance claims before the backtest.

## 10. Success criteria

All are measured **out of sample, net of costs**.

| # | Criterion | Measure |
|---|---|---|
| S1 | The signal beats simple baselines | Rank IC and hit rate vs baselines, with spread across folds |
| S2 | Index performance is reported honestly | Net excess return, IR with bootstrap CI, drawdown, tracking error, turnover, attribution |
| S3 | No look-ahead or survivorship bias | PIT and leakage tests pass; embargo applied; dead bonds included |
| S4 | Risk is reproducible | VaR/ES/DTS and scenarios regenerate from versioned data each rebalance |
| S5 | The note is priced consistently | Black-Scholes ≈ Monte Carlo within tolerance; parity holds; hedge P&L explained |
| S6 | The AI layer is trustworthy | RAG recall@k / faithfulness / refusal; agent tool and numeric accuracy; cost per memo |
| S7 | It runs as a system | Scheduled AWS replay run reproduces local results; CI green; monitoring and runbook in place |

**A finding of "no edge after costs" is an acceptable, reportable outcome.**

## 11. Data, constraints, assumptions and risks

**Data sources**

| Source | Content | Notes |
|---|---|---|
| Bloomberg (Excel Add-In) | Bond and issuer data | Historical snapshot, no API or live feed (Terminal only). Main pull 12 Oct 2026; top-up at the start of November if needed. Credits are limited and renew monthly; no Terminal access during winter break. Raw data never committed. |
| FRED | Treasury yields, VIX, ICE BofA IG/HY OAS indices | Public |
| SEC EDGAR | 10-K / 10-Q filings | Public; used by the RAG layer and as a fundamentals fallback |

**Assumptions**

- Monthly data is enough for a monthly rebalance strategy.
- A bid-ask cost model by rating and size is a reasonable proxy for IG trading costs.
- About 7 years of history gives enough walk-forward folds. This will be confirmed in M02.
- The last ~12 months of the snapshot are held out untouched until the final evaluation, standing in for "live" performance.

**Risks and mitigations**

| Risk | Mitigation |
|---|---|
| A field is missing or has poor coverage in the pull | Top-up pull at the start of November; FRED/EDGAR fallback; otherwise documented as a limitation |
| Look-ahead or leakage | PIT joins with report dates, embargo, automated leakage tests in CI |
| Overfitting | Walk-forward, simple baselines, record of variants tried, bootstrap CIs |
| Costs wipe out the edge | Costs modelled from the start; turnover buffer; report net results only |
| No live data after the pull | Month-by-month replay demonstrates operation; final ~12 months kept as an untouched holdout; stated clearly as a limitation |
| Licence restricts cloud storage | Admin answer pending; if not allowed, only processed or public data goes to S3 |
| The LLM hallucinates | As-of retrieval, citations, refusal, read-only tools, evaluation sets, injection tests |

## 12. Scope

- **In scope:**
  - USD IG corporate bonds, monthly frequency, about the last 7 years of history;
  - one index and one capital-protected note design;
  - AWS deployment;
  - the RAG + agent research layer.
- **Out of scope:**
  - high yield, non-USD and CDS;
  - daily or intraday trading;
  - live execution;
  - LLM-generated signals;
  - claims of production alpha.
- **Parameters set in M03 (24 Oct):** benchmark index; universe rules (minimum size, rating floor, maturity range, bond vs issuer level); target horizon (1 vs 3 months). The S3 licence answer is due from the Terminal admin.

## 13. Ownership

| Diya | Teammate |
|---|---|
| Data engineering, PIT warehouse, validation | Credit/market research, relative-value framework |
| ML signal, walk-forward evaluation | Strategy design, index rulebook |
| AWS platform, MLOps, CI | Note structuring, pricing review, Greeks interpretation |
| RAG + agent and their evaluations | Economic review of signal drivers |
| Portfolio risk analytics (DTS, VaR/ES, stress) | Stress scenario design (with Diya) |

**Both members must be able to explain and defend every layer.**

## 14. Timeline

| Period | Milestones |
|---|---|
| Oct 2026 | M01 problem locked ✓ · M02 Bloomberg pull + feasibility (12–19 Oct) · M03 target, universe, PIT rules (24 Oct) · M04 RV framework · M05 ingestion |
| Nov 2026 | M06–M09 validation, warehouse, EDA + baselines, ML signal v1 · winter-data check (23 Nov) |
| 1–12 Dec | Exams, no project work |
| 13 Dec – 8 Jan | M10–M18 MVP backtest, decision gate, scale-up, ML v2, index rulebook, full backtest, risk, stress, note pricing |
| Jan 2027 | M19–M26 Greeks/hedge, filings corpus, RAG + eval, agent + eval, governance, AWS compute |
| Feb 2027 | M27–M30 research API + monitoring, dashboard, demo + report, defence |

## 15. Key concepts

| Term | Meaning |
|---|---|
| IG / HY | Investment grade (BBB- and above) / high yield (below BBB-) |
| OAS | Option-adjusted spread: spread over the risk-free curve after removing embedded-option value |
| Excess return | Bond return minus duration-matched Treasury return |
| Spread duration | Approximate % price change per 100 bp spread move |
| DTS | Duration × spread; scales spread risk by spread level |
| Relative value | Performance vs comparable bonds, not market direction |
| Rank IC | Spearman correlation between predicted and realised cross-sectional ranks |
| Walk-forward / embargo | Repeated train-past/test-next evaluation / gap preventing overlap leakage |
| Point-in-time (PIT) | Using only information known on the decision date |
| Survivorship bias | Inflated results from excluding bonds that matured, defaulted or were downgraded |
| Information ratio | Excess return ÷ tracking error |
| VaR / ES | Loss threshold at X% confidence / average loss beyond that threshold |
| Participation rate | Share of index upside paid to the note holder |
| Delta / gamma / vega / theta | Option sensitivity to index level / to delta's change / to volatility / to time |
| RAG | Retrieval-augmented generation: LLM answers grounded in retrieved, cited documents |
| Agent | LLM that calls tools in a loop to complete a task |

---

## 16. Explaining CreditLens

### 30-second version

> "CreditLens is a systematic credit strategy for US investment-grade bonds. Each month an ML model ranks bonds by whether their spreads are cheap or expensive versus similar bonds; a rules-based index turns that into portfolio weights. We backtest it walk-forward with point-in-time data and trading costs, measure its risk under stress, and package it as a capital-protected note we price and hedge. It runs on AWS, and an AI research layer explains each issuer from its SEC filings — but it never makes the investment decision."

### 2-minute structure

1. **Problem:** similar bonds trade at different spreads; part of the gap is mispricing that corrects.
2. **Data:** Bloomberg + FRED + EDGAR; point-in-time joins; dead bonds kept.
3. **Signal:** cross-sectional ranking; ridge vs gradient boosting; walk-forward with an embargo; rank IC vs simple baselines.
4. **Index:** rulebook with caps and a turnover buffer; net-of-cost backtest; attribution.
5. **Risk and product:** DTS, VaR/ES, stress; capital-protected note with Greeks and hedge simulation.
6. **Platform and AI:** AWS scheduled pipeline + API; RAG/agent for research only, evaluated.
7. **Result + lesson:** the honest net result, the main limitation, and what we'd do next.

### Questions to expect

| Question | Core of the answer |
|---|---|
| Why relative value, not predicting spreads? | Market direction is mostly macro and hard to forecast. Ranking within a month removes it and is testable with rank IC. |
| Why not just buy the widest bonds? | That's one of our baselines. ML must beat it, otherwise it adds nothing. |
| How do you avoid look-ahead? | Report-date PIT joins, an embargoed walk-forward, and tests that fail if future data is used. |
| Why rank IC and not RMSE? | The signal is used as a ranking, so the metric should evaluate ranking. |
| What if costs kill the edge? | We model costs from day one and report net results. "No edge" is a valid finding. |
| Why can't the AI make decisions? | Unreliable, hard to audit, and open to prompt injection. Decisions stay in deterministic rules; the AI explains. |
| How does the note work? | Zero-coupon bond for protection plus a call on the index for upside. Participation depends on rates and volatility. |
| Why Fargate for the backtest and Lambda for the API? | Batch job vs request/response. Duration, memory and cost profile differ. |
| What would change at a bank? | Licensed data feeds, daily risk, model risk management review, entitlements, a production-grade cost model. |
