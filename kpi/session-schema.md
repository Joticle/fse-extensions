# Session Schema

JSON schema for `.kpi/sessions/[session-id].json`. One file per session in adopting projects. Written by CLI at session end per Standing Order #KPI-1.

## File naming

`.kpi/sessions/[session-id].json` where `[session-id]` matches the project's session identifier verbatim (flat or hierarchical per the project's numbering convention defined in FSE.md).

Examples:
- `.kpi/sessions/SESSION_56.json` (flat)
- `.kpi/sessions/Session-7D.json` (hierarchical)
- `.kpi/sessions/Session-2B.1.2.json` (hierarchical with sub-decimals)

Filenames replace `.` with `-` because some filesystems and CLI tools treat dots specially. The `session` field inside the JSON preserves the original notation including dots.

## Schema (version 1.0)

```json
{
  "fse_kpi_schema_version": "1.0",
  "project": "string",
  "session": "string",
  "date_started": "ISO8601 UTC",
  "date_committed": "ISO8601 UTC",
  "type": "Feature | Polish | Infrastructure | Refactor | Bugfix | Methodology | Docs | Security",
  "is_feature": "boolean",
  "sha": "string (7-char abbrev or full)",
  "scope_summary": "string (one-line summary, ≤140 chars)",

  "complexity": {
    "pre_session": {
      "surface_area": "0..3",
      "cognitive_depth": "0..3",
      "test_scope": "0..3",
      "risk_surface": "0..3",
      "spec_ambiguity": "0..3",
      "total": "0..15",
      "bracket": "Trivial | Light | Moderate | Heavy | Major"
    },
    "post_session": {
      "surface_area": "0..3",
      "cognitive_depth": "0..3",
      "test_scope": "0..3",
      "risk_surface": "0..3",
      "spec_ambiguity": "0..3",
      "total": "0..15",
      "bracket": "Trivial | Light | Moderate | Heavy | Major"
    },
    "estimation_drift": "integer (post_session.total - pre_session.total)"
  },

  "metrics": {
    "files_modified": "integer",
    "loc_added": "integer",
    "loc_removed": "integer",
    "test_count_before": "integer",
    "test_count_after": "integer",
    "test_delta": "integer (after - before)",
    "build_state": "string ('errors/warnings' format, e.g. '0/0')",
    "claude_compute_seconds": "integer (cumulative wall-clock spent in CLI)",
    "operator_estimated_traditional_hours": "number (operator's estimate of how long this session would have taken without AI)",
    "actual_claude_powered_minutes": "number (wall-clock minutes from session_started to session_committed)"
  },

  "outcomes": {
    "hitl_phases": "array of strings (e.g. ['VERIFY', 'EXECUTE-step-3'])",
    "deviations_documented": "integer (count of decisions logged as deviations from spec)",
    "outcome": "clean | partial | reverted | rolled_forward",
    "drift_session": "boolean (true if any drift moment was logged for this session)",
    "operator_satisfied": "boolean (operator's post-session signal)"
  },

  "drift_moments": "array of drift_id strings (foreign keys into drift_moments.json)"
}
```

## Field semantics

### Top-level

- **`fse_kpi_schema_version`** — schema version this file follows. Required for forward-compatibility. Current: `"1.0"`.
- **`project`** — stable project identifier matching the value used across the portfolio. Should not change between sessions in the same project.
- **`session`** — the session identifier exactly as written in commit messages and FSE_STATE.md (`SESSION_56`, `Session 7D`, `Session 2B.1.2`).
- **`date_started`** / **`date_committed`** — ISO 8601 UTC timestamps. `date_started` is when CLI began the session (first prompt), `date_committed` is when the session's final commit landed.
- **`type`** — categorical session type. Feature work, Polish (visual/UX refinement), Infrastructure, Refactor, Bugfix, Methodology (FSE adoption / standing orders), Docs, Security. Pick the dominant type if multiple apply.
- **`is_feature`** — boolean shortcut: `true` only if `type == "Feature"`. Convenience field for FCI rollups (which weight feature work specifically).
- **`sha`** — git SHA of the session's final commit (the one that landed `.kpi/sessions/[session-id].json` itself). 7-char abbreviation is acceptable.
- **`scope_summary`** — one-line summary, suitable for table display. Should match (or closely paraphrase) the commit-message subject line.

### Complexity block

- **`pre_session`** — operator's estimate before session begins. Captured from the session prompt header.
- **`post_session`** — CLI's re-score at session end based on actual work performed.
- **`estimation_drift`** — `post_session.total - pre_session.total`. Negative means the session was easier than estimated; positive means harder. `|drift| > 2` is itself a tracked signal in aggregates.

Each scoring block contains the five dimensions plus computed `total` and `bracket`. See `complexity-scoring.md` for dimension definitions.

### Metrics block

- **`files_modified`** — count of files in the session's final diff (additions + modifications + deletions, excluding pure renames if the rename is unaccompanied by content change).
- **`loc_added`** / **`loc_removed`** — git diffstat numbers from the session's final commit.
- **`test_count_before`** / **`test_count_after`** / **`test_delta`** — test count from the project's test runner. Delta is convenience; consumer should not trust it without verifying `after - before`.
- **`build_state`** — `"errors/warnings"` format. `"0/0"` is the goal; `"0/3"` means 0 errors and 3 pre-existing or session-introduced warnings.
- **`claude_compute_seconds`** — cumulative seconds spent in CLI tool calls (or equivalent for non-Claude AI assistants). Distinct from wall-clock minutes because of HITL pause time.
- **`operator_estimated_traditional_hours`** — operator's honest estimate of how long this exact scope would have taken without AI assistance. Used in AAF calculations; calibration drift over time is itself a useful signal.
- **`actual_claude_powered_minutes`** — wall-clock minutes from session start to session commit, including HITL pauses. The denominator in AAF calculations.

### Outcomes block

- **`hitl_phases`** — array of FSE phase tags where Human-in-the-Loop intervention occurred. Examples: `"VERIFY"`, `"PLAN"`, `"EXECUTE-step-3"`, `"VALIDATE"`. Empty array if the session ran fully autonomous after kickoff.
- **`deviations_documented`** — count of decisions explicitly flagged as deviations from the spec during the session. Drives the "how often does CLI take unsupervised judgment" signal.
- **`outcome`** —
  - `"clean"` — all spec items shipped, build green, no rework
  - `"partial"` — some spec items shipped, others halted (interruption, scope reduction, or operator-acknowledged deferred items)
  - `"reverted"` — session attempted but the commit was reverted before merge
  - `"rolled_forward"` — session shipped but a follow-up session was required to fix issues introduced
- **`drift_session`** — true if any drift moment is associated with this session (cross-checked against `drift_moments` field).
- **`operator_satisfied`** — operator's binary post-session signal. Captured as part of the session-end HITL handshake. Distinct from `outcome` — a session can ship "clean" but leave the operator unsatisfied (premature completion, scope misread, etc.).

### drift_moments

Array of `drift_id` strings that reference entries in `.kpi/drift_moments.json`. Empty array if the session ran without drift. See `drift-moments.md` for the drift moment schema.

## Atomicity

The session JSON file is written as part of the same git commit as the session's actual work. CLI must not commit the session work without the JSON, and must not commit the JSON without the work — they ship together or not at all. This is enforceable as a pre-commit hook by adopting projects.

## Worked example

See `example-populated/example-session-01.json` for a fully populated example.
