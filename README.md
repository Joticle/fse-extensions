# FSE Extensions

Stack-specific patterns, standing orders, and anti-patterns for projects built using [FlowState Engineering](https://github.com/Joticle/fse-core).

The core repository defines the methodology — how sessions are run, how documents are structured, how teams (or solo developers) ship without breaking themselves. This repository is where the methodology meets the metal. One folder per stack. Each folder owns its own conventions, its own pain points, and its own rules.

## What lives here

Every stack folder follows the same structure:

- `README.md` — what this stack covers, who maintains it, current status
- `standing-orders.md` — the absolute rules for this stack (the things that, when violated, cause real damage)
- `patterns.md` — proven patterns that work in this stack
- `anti-patterns.md` — things that look reasonable but consistently break

A stack folder is "complete" when all four files exist and have been validated against at least one real production project. Until then, it's a `WANTED.md` placeholder asking for a maintainer.

## Stacks

| # | Stack | Status | Maintainer |
|---|---|---|---|
| 1 | [dotnet](./dotnet) | Reference implementation | Scott Michael Wilson |
| 2 | [react](./react) | Open for contribution | — |
| 3 | [python](./python) | Open for contribution | — |
| 4 | [nodejs](./nodejs) | Open for contribution | — |
| 5 | [rails](./rails) | Open for contribution | — |
| 6 | [go](./go) | Open for contribution | — |
| 7 | [java-spring](./java-spring) | Open for contribution | — |
| 8 | [php-laravel](./php-laravel) | Open for contribution | — |
| 9 | [flutter](./flutter) | Open for contribution | — |
| 10 | [swift-ios](./swift-ios) | Open for contribution | — |

.NET is first because the creator has worked with it for 25 years and it's the stack everything was originally pressure-tested in. That is the entire reason. There is no deeper architectural meaning to the ordering. The other nine are listed in roughly the order they tend to come up in the kind of projects FSE was built for.

## Methodology Extensions

Stack extensions cover language-specific patterns. Methodology extensions cover cross-cutting practices that any stack can adopt.

| Extension | Purpose |
|---|---|
| [kpi](./kpi) | Methodology extension for session-level metrics tracking. Five-dimension complexity scoring, JSON schemas for sessions and drift moments, FCI/AAF/DR/Net AAF aggregates. Opt-in. |

Methodology extensions are opt-in. Adopting one is a per-project decision, recorded in the project's `FSE_STATE.md` adoption block.

## How to contribute a new stack

Pick an open stack. Read the root `CONTRIBUTING.md`. Match the structure and quality bar of the dotnet folder. Submit a pull request.

The first contributor whose extension passes review becomes the maintainer for that stack. Maintainers keep their stack honest, respond to issues, and review pull requests against their folder. This is not a credential — it's a commitment to keep the work true to real practice in that stack.

## How to contribute to an existing stack

Open an issue first if you're proposing a meaningful change. Pull requests for typos, broken links, or small clarifications can go straight in. Larger changes need conversation, especially in `standing-orders.md` where every rule has a reason and removing one without understanding why it was added is how methodologies decay.

## What an extension is not

An extension is not a tutorial. It's not a "getting started with X" guide. It's not a curated awesome-list. It's a working developer's notes on how FSE actually behaves when applied to a specific stack — what the methodology asks for, how that stack delivers it, where the friction is, and what the absolute rules are.

If you've never shipped a real project in the stack you want to maintain, this isn't the right place to start. The point of an extension is that it's been validated by use, not assembled from documentation.

## License

Apache 2.0. See [LICENSE](./LICENSE).

## Created by

Scott Michael Wilson / Joticle Inc.
