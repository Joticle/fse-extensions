# .NET Extension for FlowState Engineering

This is the reference implementation of an FSE extension. It exists for two reasons: it's the stack the methodology was originally pressure-tested in, and it's the working example other stack maintainers can match their submissions against.

## What this folder covers

FSE applied to .NET projects. Specifically:

- .NET 8 and newer (latest stable assumed)
- C# as the language
- ASP.NET Core for web hosts
- Razor Pages and Razor Class Libraries as the modular monolith UI strategy
- SQL Server with raw SQL as the database approach
- EF Core used for querying only, never for schema management
- xUnit, Moq, and FluentAssertions for testing

If your .NET project uses Blazor, MAUI, WinForms, or WPF, most of the standing orders still apply but the patterns and anti-patterns lean Razor Pages. PRs adding companion patterns for those scenarios are welcome.

## Maintainer

Scott Michael Wilson / Joticle Inc. This stack is maintained as part of the broader FSE work because it's where the methodology came from. If a future contributor wants to take over day-to-day maintenance of this folder specifically, that conversation is open.

## Production grounding

The patterns in this folder are drawn from eight production solutions running on .NET in 2026: a multi-tenant healthcare compliance SaaS, a senior lifestyle platform, an investment platform, a legal compliance platform, a methodology marketing site, a corporate command center, an education platform, and the Bedrock generator that produces the others. Some are in active production traffic, some are in late-stage build. None are demos.

That grounding is the reason this is the reference. If you're submitting a different stack, your bar is to be able to point at comparable real work in your stack — not the same scale necessarily, but the same kind of pressure-tested.

## The four files

This folder contains the four files every stack extension is required to have:

- `README.md` — this file
- `standing-orders.md` — the absolute rules for .NET projects under FSE
- `patterns.md` — proven patterns we use repeatedly
- `anti-patterns.md` — things that look reasonable in .NET but consistently break

Read them in order. Standing orders are non-negotiable. Patterns are reusable. Anti-patterns are the traps.

## Relationship to fse-core

Everything in this folder is downstream of [fse-core](https://github.com/Joticle/fse-core). Where the core says "no cross-module DbContext injection," this folder describes specifically how that plays out in EF Core and what the .NET-specific traps look like. Where the core says "raw SQL only for schema," this folder explains the SSMS workflow and the DbContext patterns that support it.

If anything in this folder ever conflicts with the core, the core wins. Open an issue.

## What you won't find here

- Tutorials. This isn't a "learn ASP.NET Core" resource.
- Tool-specific advice. Visual Studio, Rider, VS Code, and editor preferences are out of scope.
- Library evangelism. We name the libraries we use, but we don't recommend them to you. Use what fits your project.
- AI assistant prompts. Those belong as optional addons, not in core methodology or stack extensions.

## License

Apache 2.0. See the root [LICENSE](../LICENSE).
