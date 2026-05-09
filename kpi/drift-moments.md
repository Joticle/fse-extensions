# Drift Moments Schema

JSON schema for `.kpi/drift_moments.json`. One file per project. Append-only. Records every incident where AI behavior deviated from spec or operator intent in a way that cost time, reputation, or quality.

## File location

`.kpi/drift_moments.json` at repo root. Created at adoption with an empty `moments` array. Subsequent sessions append; entries are never deleted or rewritten.

## Schema (version 1.0)

```json
{
  "fse_kpi_schema_version": "1.0",
  "project": "string",
  "moments": [
    {
      "drift_id": "string (per format below)",
      "session": "string (matches the session field in sessions/[session-id].json)",
      "timestamp": "ISO8601 UTC (when the drift was identified)",
      "category": "scope_misread | pattern_divergence | over_engineering | premature_completion | spec_invention | constraint_violation | hallucinated_dependency | regression_introduction",
      "what_happened": "string (factual description, blame-neutral, ≤500 chars)",
      "root_cause": "string (why CLI did the wrong thing — ambiguity, missing pattern reference, conservative bias, etc.)",
      "correction_required": "string (what was done to recover)",
      "session_lost_minutes": "integer (rough wall-clock cost of the drift, including correction)",
      "preventable_by": "string (what guardrail, standing order, or prompt change would have prevented this)"
    }
  ]
}
```

## drift_id format

`{project-prefix}-{YYYY-MM-DD}-{NNN}` where:
- `project-prefix` — short slug for the project (e.g., `bp` for BoomerPath, `csp` for CollegeWCProd, `lex` for LexShield). Stable across the project's lifetime.
- `YYYY-MM-DD` — UTC date the drift was identified (not the date the underlying mistake was made, if those differ).
- `NNN` — three-digit zero-padded sequence number for that project on that date. Reset each day. Example: `bp-2026-05-03-001`.

The example file uses `exp-` (for "ExampleProject") as its prefix.

## Drift category enum

Eight categories, each capturing a distinct failure mode. CLI picks the closest match; if multiple apply, pick the one that started the chain.

| Category | Definition | Typical preventable_by |
|----------|------------|------------------------|
| `scope_misread` | CLI interpreted the prompt's scope narrower or wider than operator intended. | More explicit scope boundaries in spec; CLI re-confirming scope on partial authorization. |
| `pattern_divergence` | CLI invented an approach inconsistent with established patterns elsewhere in the project or portfolio. | Standing order requiring cross-project pattern check before infra decisions. |
| `over_engineering` | CLI built more rigor than the use case required (premature optimization, premature immutability, premature abstraction). | Operator review of architectural decisions before implementation; lifecycle-constraint clarification. |
| `premature_completion` | CLI declared session shipped without verifying the customer-facing surface, end-to-end flows, or critical paths. | Smoke-test gate before session-complete signal; mandatory walkthrough of customer-facing flows. |
| `spec_invention` | CLI added features, sections, or behavior not asked for. Distinct from `over_engineering` (which is about rigor on what WAS asked). | Strict-spec-adherence prompts; explicit "do not invent scope" reminders. |
| `constraint_violation` | CLI violated a known constraint — committed credentials, used a banned package, broke a Standing Order, etc. | Pre-commit hooks; CLI re-reading FSE_POLICE.md before high-risk operations. |
| `hallucinated_dependency` | CLI referenced a package, API, function, or file that does not exist in the project or its dependencies. | Mandatory existence-check before referencing; CLI grep-before-call for cross-module symbols. |
| `regression_introduction` | CLI broke previously-working functionality while making changes elsewhere. | Test-before-commit gate; running the full suite (not just affected tests) before declaring a session complete. |

The category list is closed for v1.0. New categories require schema bump (1.0 → 1.1) and an aggregates.md update.

## Append-only contract

Drift moments are an immutable historical record. Once written, an entry is never:

- **Deleted** — if a moment was wrongly logged, append a new moment with `category: "scope_misread"` (or another applicable category) referencing the original `drift_id` in `what_happened` rather than rewriting the original.
- **Rewritten** — typo fixes are acceptable in the originating session's commit only (within the same hour); after that, immutable.
- **Reordered** — entries appear in the order they were written. Do not sort by date or session; chronological-write order is itself signal.

Consumer dashboards must treat the file as append-only and reject any commit that decreases the entry count.

## Privacy considerations

Drift moments will be read by:

- The operator (always)
- Future CLI sessions reading the file to learn from past mistakes
- Portfolio dashboard consumers (potentially across an organization)
- Any reviewer of the repo (collaborators, auditors)

Therefore:

- **No PII in `what_happened`, `root_cause`, or `correction_required`** — describe systems and behaviors, not people.
- **Operator names and CLI model identifiers are acceptable** — they are agentic identifiers, not personally-identifying.
- **Customer data, credentials, internal URLs, secrets** — must never appear. Redact before logging. If a drift moment was specifically a credential exposure, describe the category of credential, not the value.
- **Tone is blame-neutral** — drift moments are diagnostic artifacts, not accusations. "CLI interpreted X as Y when operator intended Z" beats "CLI was wrong about X."

## Worked example

See `example-populated/example-drift-moments.json` for a fully populated example with four drift incidents demonstrating different categories.
