# KPI Extension

Methodology extension for FSE projects that want to capture session-level metrics tracking work output, drift incidents, and AI-accelerated velocity. Opt-in. Designed for portfolio-wide rollup via consumer dashboards.

## When to adopt

Adopt this extension when you want to:
- Track which sessions ship feature work vs. polish vs. infrastructure
- Measure how much faster sessions ship with FSE methodology vs. traditional estimation
- Surface drift incidents (where AI behavior deviated from spec) for retrospective analysis
- Roll up metrics across multiple projects in a single dashboard

Do NOT adopt this extension if:
- You're early in a project and metrics noise outweighs signal
- You're not committed to operator-supplied complexity estimation per session
- Your team doesn't have a downstream consumer for the JSON files

## What it provides

- Five-dimension complexity scoring framework (0-15 total)
- JSON schema for per-session metrics files
- JSON schema for drift moment incident logging
- Aggregate metric definitions (FCI, AAF, DR, Net AAF)
- One KPI Extension Standing Order: KPI capture at session end (opt-in)
- Reference consumption contract for portfolio dashboards

## Adoption mechanics

A project adopting this extension:

1. Creates `.kpi/sessions/` directory and `.kpi/drift_moments.json` (with empty `moments` array initially) at repo root
2. Tracks `.kpi/` in git (these files ARE the KPI artifacts; they should be committed)
3. Adopts the KPI Extension Standing Order (mandatory capture at session end for adopting projects)
4. Updates project's FSE_STATE.md "FSE Adoption State" block to reference this extension with adoption date

Going forward, every session ends with `.kpi/sessions/[session-id].json` written and committed alongside the session work. Drift moments append to `.kpi/drift_moments.json`.

## Schema versioning

Every JSON file produced by this extension carries a `fse_kpi_schema_version` field (e.g., "1.0"). Schema evolution is backward-compatible within major versions. Consumer dashboards must handle multiple schema versions during transition periods.

## Files in this extension

| File | Purpose |
|------|---------|
| README.md | This file |
| standing-orders.md | KPI Extension Standing Order for KPI capture |
| complexity-scoring.md | 5-dimension 0-15 scoring framework |
| session-schema.md | JSON schema for `.kpi/sessions/[session-id].json` |
| drift-moments.md | JSON schema for `.kpi/drift_moments.json` plus drift category enum |
| aggregates.md | FCI, AAF, DR, Net AAF formulas with worked examples |
| command-center-contract.md | Reference consumption contract for portfolio dashboards |
| example-populated/ | Anonymized real-world examples from a contributing project |

## Provenance

Authoring source: empirical session work demonstrated need for complexity-weighted session metrics; abstracted upstream to FSE for portfolio-wide use. The example-populated/ files use anonymized data from real session history rather than hypothetical examples.
