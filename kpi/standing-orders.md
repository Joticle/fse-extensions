# KPI Extension Standing Orders

One KPI Extension Standing Order. Projects that adopt this extension also adopt this rule. This is opt-in — it does not extend the Universal Standing Orders defined in Core.

## SO #KPI-1: KPI Capture is Mandatory at Session End

**Rule:** Every session that produces a commit in an adopting project must also produce a `.kpi/sessions/[session-id].json` file in the same commit. Drift moments (if any) append to `.kpi/drift_moments.json`.

**Why:** Sessions without metrics capture are invisible to portfolio rollup. Operators lose the ability to evaluate velocity, drift patterns, and feature density across projects. The cost of capture (~30 seconds at session end) is dwarfed by the cost of missing data.

**Consequence of violation:** Session-level metrics fall out of FCI / AAF / DR / Net AAF calculations. Drift incidents go un-logged, foreclosing root-cause analysis later. Cross-project rollup becomes unreliable.

**How to comply:**

1. Operator declares pre-session complexity estimate in the session prompt header (5 dimensions, total 0-15, see complexity-scoring.md)
2. CLI confirms or challenges the estimate before starting work; disagreement triggers brief alignment
3. CLI captures session metrics during work (files modified, LOC delta, test delta, claude_compute_seconds)
4. At session end, CLI writes `.kpi/sessions/[session-id].json` per session-schema.md
5. If drift occurred during the session, CLI appends to `.kpi/drift_moments.json` per drift-moments.md
6. Both files are part of the session's final commit

**Exceptions:**
- Pre-adoption sessions (no metrics required for sessions that predate adoption)
- Reverted sessions (capture the attempt; mark `outcome: "reverted"`)
- Emergency hotfixes where capture would delay shipping (capture retroactively within 24 hours)
