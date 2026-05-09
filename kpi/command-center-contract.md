# Command Center Contract

Reference contract for portfolio dashboard consumers that read KPI artifacts. Defines the read surface, expected behavior, and stability guarantees this extension exposes.

This document is descriptive, not prescriptive. The reference consumer is one possible implementation. Other dashboards (CLI tools, web UIs, metric-emitting agents, BI integrations) can consume the same artifacts following this contract.

## Read surface

A consumer of this extension reads:

1. **`.kpi/sessions/[session-id].json`** — one file per session. Read all of them, or filter by date / type / project.
2. **`.kpi/drift_moments.json`** — one file per project. Read whole; filter by date / category / session.
3. **`FSE_STATE.md`** (optional) — for project metadata, current adoption state, and human-readable session summaries that complement the JSON metrics.

Consumers SHOULD NOT:

- Read or depend on intermediate state in `.kpi/` (cache files, lockfiles, partial writes)
- Modify `.kpi/` files (read-only access from the consumer's perspective)
- Rely on filesystem timestamps — use `date_committed` from the JSON, not file mtime

## Discovery

A consumer discovers KPI-adopting projects by:

1. Scanning a known list of project repos (operator-supplied)
2. For each repo, checking for `.kpi/sessions/` directory existence
3. Within `.kpi/sessions/`, reading every `*.json` file
4. Reading `.kpi/drift_moments.json` if present

Projects that have not adopted this extension simply lack `.kpi/`. Consumers must handle missing-directory gracefully (filter from the list, do not error).

## Stability guarantees

What this extension guarantees to consumers within a major schema version (1.x):

- **Field additions are non-breaking.** New fields will be added to schema versions but optional; consumers that ignore unknown fields will continue to work.
- **Field removals are breaking.** Any field removal triggers a major version bump (1.x → 2.0). Consumers MUST check `fse_kpi_schema_version` and refuse to parse unknown majors.
- **Enum value additions are non-breaking.** New `category` values (in drift_moments.json) or `type` values (in sessions/*.json) may be added in minor versions. Consumers should treat unknown enum values as "Unknown" and surface them rather than dropping the record.
- **Enum value removals or renames are breaking.** Trigger major version bump.
- **JSON shape (objects, arrays, types) is stable within a major version.** A field that is `string` in 1.0 will still be `string` in 1.5.

What this extension does NOT guarantee:

- The order of array elements in `drift_moments.json` is append-only (chronological by write), but consumers should not depend on this for correctness — sort explicitly if order matters.
- The set of files in `.kpi/sessions/` may include legacy filenames during schema transitions; filter on `fse_kpi_schema_version` if needed.
- File encoding is UTF-8 without BOM. Other encodings are non-conformant; consumer behavior on non-UTF-8 files is undefined.

## Aggregation responsibility

Aggregation (FCI, AAF, DR, Net AAF, etc.) happens at the consumer, not at the source. The source (`.kpi/sessions/*.json`) provides flat per-session data; the consumer computes whatever rollups it needs over whatever windows it cares about.

This means consumers can:

- Define their own aggregation windows (last 30 days, last sprint, since last methodology change)
- Define their own filters (per-project, per-type, per-operator, per-stack)
- Define their own derived metrics (variance of FCI, drift category distribution, complexity bracket histogram)

The four aggregates defined in `aggregates.md` are the reference set. Consumers are encouraged to ship those plus whatever else fits their portfolio.

## Multi-project rollup

For a portfolio of N projects:

1. The consumer maintains a list of project repos (each one has its own `.kpi/`).
2. For each repo, the consumer pulls (`git pull` or equivalent) and reads `.kpi/sessions/*.json` and `.kpi/drift_moments.json`.
3. The consumer flattens the data: every session JSON becomes a row tagged with `project`, every drift moment becomes a row tagged with `project`.
4. Aggregations run over the flattened data, optionally grouped by `project`, `type`, `bracket`, etc.

Projects must use distinct values for the `project` field to avoid rollup collisions. The `project` field's stability across sessions in the same repo is the consumer's only reliable join key.

## Privacy and consent

Drift moments may contain references to operator behavior, missed expectations, and session-cost estimates. Before exposing aggregated data outside the team that produced it:

- Confirm operator consent for the data sharing scope
- Verify drift moment text contains no PII, credentials, or customer data (per `drift-moments.md` privacy section)
- Consider redacting `what_happened` / `root_cause` for inter-team or external display, while keeping `category`, `session_lost_minutes`, and `preventable_by` for trend analysis

The reference consumer assumes single-team use by default. Multi-team or external display requires explicit configuration.

## Failure modes

How a consumer should handle common failure modes:

| Failure | Consumer behavior |
|---------|-------------------|
| `.kpi/` missing in a project | Skip the project; do not error |
| `.kpi/sessions/[id].json` malformed (invalid JSON) | Skip that session; log a warning; continue with others |
| `fse_kpi_schema_version` is a future major | Refuse to parse; surface a "consumer update required" warning |
| `fse_kpi_schema_version` is a future minor | Parse with best effort; ignore unknown fields; continue |
| Drift moment references a `session` not in `sessions/` | Surface as "orphan drift moment"; do not crash |
| Two sessions in the same project share a `session` value | Surface as duplicate; use the most recent `date_committed`; flag for operator |
| Negative `session_lost_minutes` | Treat as data error; clamp to 0 for aggregation; flag for operator |

## Reference consumer

A reference portfolio dashboard implementing this contract is intended to be released alongside the extension. Until then, this contract serves as the spec for any team building their own dashboard.

Recommended minimum view for a reference consumer:

- **Per-project panel** — last 10 sessions, FCI trend, AAF trend, DR, drift category distribution
- **Portfolio panel** — aggregated FCI / AAF / Net AAF across all projects, drift heatmap by project × category
- **Session detail view** — full session JSON, linked to git SHA, drift moments associated with that session

Future consumers may add: ML-based drift categorization, automated standing-order generation from drift patterns, integration with external project management tools.
