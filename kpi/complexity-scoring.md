# Complexity Scoring Framework

Five-dimension 0-15 scoring system for session sizing. Each dimension scored 0-3 independently. Total = sum of all five.

## Dimensions

### 1. Surface Area (Files Touched)

How much of the codebase the session modifies.

| Score | Definition |
|-------|------------|
| 0 | 0-1 files |
| 1 | 2-5 files |
| 2 | 6-15 files |
| 3 | 16+ files |

### 2. Cognitive Depth

How much architectural reasoning the session requires.

| Score | Definition |
|-------|------------|
| 0 | Trivial — config tweak, copy change, single-line fix |
| 1 | Routine — single-component change, well-trod pattern |
| 2 | Substantial — multi-component coordination, judgment calls, pattern decisions |
| 3 | Foundational — architecture decisions, new patterns, trade-off analysis with long-term consequences |

### 3. Test/Quality Scope

What test surface the session affects.

| Score | Definition |
|-------|------------|
| 0 | No test impact |
| 1 | Existing tests cover, no new tests needed |
| 2 | New tests written for new behavior |
| 3 | Test infrastructure changes, integration test surface, architecture tests |

### 4. Risk Surface

What can break.

| Score | Definition |
|-------|------------|
| 0 | Zero blast radius — docs, comments, formatting |
| 1 | Contained — single feature, easy rollback |
| 2 | Cross-cutting — affects multiple modules or users |
| 3 | Foundational — affects deploy, auth, data integrity, or multiple downstream sessions |

### 5. Spec Ambiguity

How much judgment the operator's spec leaves to CLI.

| Score | Definition |
|-------|------------|
| 0 | Spec is exhaustive, zero judgment calls |
| 1 | Minor decisions within established patterns |
| 2 | Meaningful design decisions documented as deviations |
| 3 | Novel territory, multiple defensible approaches, operator validation needed |

## Brackets

| Total | Bracket |
|-------|---------|
| 0-2 | Trivial |
| 3-5 | Light |
| 6-9 | Moderate |
| 10-12 | Heavy |
| 13-15 | Major |

## Pre-session estimation protocol

1. Operator scores all 5 dimensions in the session prompt header before sending to CLI
2. CLI re-scores upon prompt parsing as part of "session understood" handshake
3. If CLI's score deviates from operator's by >2 points total, CLI surfaces the disagreement before starting work
4. Both pre-session estimates land in `.kpi/sessions/[session-id].json`

## Post-session capture

After session ends, CLI re-scores all 5 dimensions based on actual work performed. The post-session score reflects what shipped, not what was estimated. Both pre- and post-session scores are recorded; estimation drift > 2 points is itself a tracked signal.

## Worked examples

Drawn from anonymized real-world session work across the FSE portfolio:

**Trivial session (1 total):** "Replace external brand reference with project name on marketing surface" — single-file scope, zero cognitive depth, zero test impact, zero risk, exhaustive spec.

**Light session (5 total):** "Replace email provider package; update IEmailService implementation" — 2-5 files, routine cognitive depth, existing tests cover, contained risk, minor decisions within established patterns.

**Moderate session (8 total):** "Visual polish — hero gradient, card character, line-icons, section rhythm, substantial CTAs" — 6-15 files, substantial cognitive depth (visual hierarchy decisions), existing tests, contained risk, meaningful design decisions.

**Heavy session (10-11 total):** "Customer-facing surface part 1 — landing, registration wizard, legal pages" — 6-15 files, substantial cognitive depth (multi-component coordination), new tests for new behavior, cross-cutting risk, meaningful design decisions documented as deviations.

**Major session (13-14 total):** "Tenant registration foundations — claims factory, saga, email abstractions" — 16+ files, foundational architecture decisions, new test coverage, foundational risk affecting deploy/auth/data integrity, novel territory with multiple defensible approaches.
