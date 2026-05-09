# Aggregate Metrics

Four primary aggregates computed from `.kpi/sessions/*.json` and `.kpi/drift_moments.json`. These are the metrics consumer dashboards display and operators evaluate trends against.

All four aggregates accept a window parameter (sessions in date range, sessions matching a project filter, etc.). Definitions below assume a fixed window of N sessions for clarity.

## FCI — Feature Complexity Index

**What it measures:** Complexity-weighted feature work. Sessions that ship feature work count more than polish; harder feature work counts more than easy feature work.

**Formula:**

```
FCI = Σ (post_session.complexity.total)  for sessions where is_feature == true
```

**Why it's not just "feature count":** A 4-week heavy infrastructure overhaul that finally unblocks a feature module should not show up the same as a 30-minute polish pass on already-shipped feature copy. FCI weights by complexity to reflect that.

**Why post-session, not pre-session:** Pre-session estimates drift. Post-session captures what actually shipped. If estimation drift is significant, that's its own signal (see "Estimation Drift" below).

**Interpretation:**
- FCI trending up over time = feature velocity increasing
- FCI flat with rising session count = sessions shifting toward polish/infra (not necessarily bad — late-stage projects do this)
- FCI dropping = either feature work has wound down or sessions are being mis-typed

**Worth noting:** Polish and infrastructure sessions intentionally contribute zero to FCI even if they are highly complex. That's the design — FCI tracks one specific thing (complexity-weighted feature throughput), not overall engineering throughput.

## AAF — AI Acceleration Factor

**What it measures:** How much faster AI-assisted sessions are than traditional estimates.

**Formula:**

```
AAF = (Σ operator_estimated_traditional_hours * 60) / (Σ actual_claude_powered_minutes)
```

Both sums computed over the same window of sessions.

**Interpretation:**
- AAF = 1.0 — no acceleration; AI sessions match traditional estimates
- AAF = 5.0 — five times faster than traditional estimates
- AAF = 10.0+ — extremely fast (verify operator estimates aren't gamed)

**Calibration drift:** If operator estimates are consistently too high, AAF inflates over time. If too low, AAF deflates. The dashboard should chart AAF alongside `operator_estimated_traditional_hours / actual_claude_powered_minutes` per session to surface estimation drift early.

**Trend signal:** AAF tends to rise during a project's middle phase (operator and CLI develop shorthand), then can fall if the codebase complexity outpaces the team's pattern library. A falling AAF in a mature project is a "patterns library is stale" signal.

## DR — Drift Rate

**What it measures:** How often sessions involve drift incidents.

**Formula:**

```
DR = (count of sessions where drift_session == true) / (count of sessions in window)
```

**Interpretation:**
- DR = 0.00 — no drift in window. Either the window is short, or operator/CLI alignment is excellent.
- DR = 0.10 — 1 in 10 sessions had drift. Healthy.
- DR = 0.25+ — 1 in 4. Investigate. Check drift category distribution; the dominant category points at the systemic gap.
- DR = 0.50+ — 1 in 2. Methodology or operator/CLI alignment needs intervention before more sessions accumulate.

**Category-weighted variant (optional):** Some categories (like `constraint_violation` or `hallucinated_dependency`) are more expensive than others. A weighted DR can be defined as:

```
DR_weighted = Σ (category_weight * count_in_category) / count_in_window
```

with weights set per organization. Default weights are `1.0` for all categories.

## Net AAF — Acceleration Adjusted for Drift Cost

**What it measures:** AAF after subtracting time lost to drift incidents. Answers "how fast were we, *really*?"

**Formula:**

```
total_drift_minutes = Σ session_lost_minutes  (over moments in window)
actual_minutes_minus_drift = Σ actual_claude_powered_minutes - total_drift_minutes
Net AAF = (Σ operator_estimated_traditional_hours * 60) / actual_minutes_minus_drift
```

Note `actual_minutes_minus_drift` could go negative if drift cost exceeds actual minutes; in that case, Net AAF is undefined for the window (consumer should display "—" not a negative number).

**Why this matters:** If a project shows AAF = 8.0 but Net AAF = 5.0, the difference is the drift cost. That gap is the headroom for methodology improvement — the maximum AAF achievable if drift went to zero.

**Interpretation:**
- Net AAF ≈ AAF — drift cost is negligible. Methodology is well-tuned.
- Net AAF noticeably below AAF — drift cost is real. Investigate dominant drift categories.
- Net AAF undefined (negative denominator) — drift incidents in the window cost more time than was actually spent. Either the window is very small, or this is a "lost session" period worth a methodology retro.

## Estimation Drift (secondary signal)

Not a primary aggregate; a session-level signal worth surfacing on dashboards.

**Per session:**
```
estimation_drift = post_session.complexity.total - pre_session.complexity.total
```

**Aggregate:**
```
mean_estimation_drift = Σ estimation_drift / count_in_window
```

**Interpretation:**
- Mean ≈ 0 — operator and CLI agree on session sizing
- Mean > +1 — sessions are consistently harder than estimated (operator under-scopes)
- Mean < -1 — sessions are consistently easier than estimated (operator over-scopes)
- Variance is also signal — high variance with mean ≈ 0 means estimates are noisy in both directions, which is its own problem

## Worked example

A hypothetical project window of 5 sessions:

| Session | type | is_feature | post total | est hours | actual min | drift session | drift min |
|---------|------|------------|------------|-----------|------------|---------------|-----------|
| S1 | Feature | true | 13 | 80 | 540 | false | 0 |
| S2 | Feature | true | 8 | 24 | 180 | true | 25 |
| S3 | Polish | false | 5 | 6 | 49 | false | 0 |
| S4 | Infrastructure | false | 11 | 32 | 220 | true | 45 |
| S5 | Feature | true | 6 | 12 | 95 | false | 0 |

**FCI** = 13 + 8 + 6 = **27** (S3 and S4 excluded; not feature)

**AAF** = ((80+24+6+32+12) × 60) / (540+180+49+220+95) = (154 × 60) / 1084 = 9240 / 1084 ≈ **8.52**

**DR** = 2 / 5 = **0.40** (S2 and S4)

**Net AAF** = (154 × 60) / (1084 − 70) = 9240 / 1014 ≈ **9.11**

**Mean estimation drift** — not calculated above (would need pre_session totals); illustrative only.

**Read:** This window shipped substantial feature work (FCI = 27 across three feature sessions). Sessions completed at ~8.5× traditional estimates. Two of five sessions involved drift, costing ~70 minutes total. If the team eliminated drift, they'd reach Net AAF ≈ 9.1 — a 0.6× improvement. Worth investigating the two drift categories.

## Implementation note

These aggregates are computed by consumer dashboards, not by the JSON files themselves. The JSON files are flat data; aggregation happens at read time. See `command-center-contract.md` for the consumer interface.
