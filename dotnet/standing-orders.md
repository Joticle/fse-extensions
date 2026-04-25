# .NET Standing Orders

These are the absolute rules for .NET projects built under FlowState Engineering. Every rule has a consequence — the failure mode that justified adding it. Rules without consequences belong in patterns, not here.

If you violate one of these and your build still passes, that's the rule working as designed: the cost shows up later, in production, in data, or in the next developer's session.

## Database and schema

**1. No EF migrations. Ever.**
Schema is created and modified manually in SSMS using raw SQL scripts. EF Core is used for querying only. The migrations folder does not exist in any FSE .NET project.
Consequence of violation: Migrations and SSMS-managed schema drift apart silently. Production schema becomes a fiction the codebase no longer describes accurately. The recovery path is rebuilding the schema by hand under pressure.

**2. Never call `EnsureCreated`, `Migrate`, or any auto-schema method on a DbContext.**
Not in Program.cs, not in tests, not in seed routines, not anywhere.
Consequence of violation: A developer running tests against a misconfigured connection string can silently create a parallel schema in the wrong database. We have lost a full evening to this.

**3. `TenantId` on every tenant-scoped table and every tenant-scoped query.**
Tables that hold tenant data must carry `TenantId` as a non-null column. Every query that reads tenant data must filter by `TenantId`. No exceptions.
Consequence of violation: Cross-tenant data leak. This is the single most expensive mistake possible in a multi-tenant system.

**4. `DateTime` is always UTC. Always.**
Stored as UTC, retrieved as UTC, compared as UTC. Local time conversion happens at the UI edge, never in services or queries.
Consequence of violation: Off-by-N-hour bugs that surface only in specific timezones, often in production, often in audit logs that legal needs.

**5. Raw SQL for schema goes in `db/` scripts, not in code.**
Schema scripts are checked in as `.sql` files for SSMS execution. They are not run from the application at startup.
Consequence of violation: The application becomes responsible for schema state, which contradicts standing order #1 and reintroduces migration-style problems through the back door.

## Module boundaries

**6. No cross-module DbContext injection.**
A module never receives or references another module's DbContext. If module A needs data from module B's tables, module A reads it via raw SQL through its own DbContext, or through a contract interface defined in Platform.Shared.
Consequence of violation: Module A becomes coupled to module B's schema. Renaming a column in B silently breaks A. The modular monolith stops being modular and degrades into a tangled monolith with extra ceremony.

**7. No cross-module service injection.**
The same rule applies to services. Module A does not inject `IBService` from module B. Cross-module communication happens through interfaces defined in `Platform.Shared` and implemented as contracts.
Consequence of violation: Same as #6. Hidden coupling, silent breakage, eventual collapse of module independence.

**8. `Platform.Shared` has zero references to any module.**
`Platform.Shared` defines interfaces, enums, base entities, and constants. It does not depend on any module project. The dependency arrow points one way.
Consequence of violation: Circular references that the compiler will eventually catch, but only after the design has rotted enough that fixing it is a multi-day refactor.

## Code structure

**9. No stubs, TODOs, placeholders, or `NotImplementedException` in committed code.**
Every committed file is production-ready. If it isn't done, it doesn't get committed.
Consequence of violation: TODO accumulation. The codebase fills with unfinished work that no one remembers the context for. Six months later, a TODO in a critical path causes a production incident.

**10. No snippets, partials, or "fill in the rest" comments.**
Files are complete. When a session ends, every modified file is in a state where it could ship.
Consequence of violation: Same as #9, plus the additional damage of a developer (or AI assistant) trying to "complete" a snippet from memory and producing something that diverges from the original intent.

**11. `@@media` in Razor Pages, never `@media`.**
Razor requires the doubled `@@` to escape the at-sign in CSS media queries inside `.cshtml` files.
Consequence of violation: Razor parser errors at runtime. The page fails to render and the error message is unhelpful unless you already know this rule.

**12. Zero inline styles in `.cshtml` files unless the project explicitly documents an exception.**
All styling lives in module CSS files using project CSS custom property tokens.
Consequence of violation: Style drift. The design system is bypassed. Six months later, hardcoded colors and one-off margins are scattered across dozens of pages and consolidating them requires a CSS archaeology project.

**13. No hardcoded colors in CSS. Use project CSS custom property tokens.**
If a color appears as a literal hex code or named color in a stylesheet, the design system has been violated.
Consequence of violation: Same as #12. Theming becomes impossible. Brand changes become a search-and-replace nightmare.

## Build and validation

**14. Self-healing build loop runs after every change, not just at session end.**
After every meaningful file change, the full solution is built. Errors are addressed immediately. Warnings are checked against the project baseline.
Consequence of violation: Errors accumulate silently. By the time you discover one, you've stacked three more on top of it and untangling them is harder than fixing them one at a time would have been.

**15. Maximum three self-healing attempts per error.**
If the same error survives three fix attempts, stop. Report the error, the attempted fixes, and the relevant standing orders consulted. Do not attempt a fourth fix using the same reasoning.
Consequence of violation: Reasoning loops. The fourth attempt usually makes things worse because it's the same logic that produced the first three failures.

**16. Never declare a session complete with a broken build.**
Zero errors. Warnings at or below the documented baseline. If you can't get there, the session ends with the build broken and a clear handoff describing what's left to fix — not with a claim of completion.
Consequence of violation: The next session starts in a hole. Trust in session reports degrades. The methodology stops working.

## Secrets and deploy

**17. `publish.ps1` is gitignored and contains deploy credentials.**
It is never committed. Its contents are never output in conversation, logs, or AI assistant transcripts.
Consequence of violation: Credentials in version control. Credentials in chat history. The recovery path is credential rotation across every dependent system, which is a bad day.

**18. `appsettings.json` contains no secrets. All secrets are in Azure Key Vault.**
Connection strings with passwords, API keys, third-party tokens — all in Key Vault, all retrieved at runtime, none in source.
Consequence of violation: Same as #17.

**19. `appsettings.Development.json` is gitignored.**
Local development configuration that may contain local secrets or developer-specific connection strings stays out of version control.
Consequence of violation: Developer machine state leaks into the repo. Sometimes that includes credentials, sometimes it includes paths that only work on one machine.

## Testing

**20. Tests use xUnit, Moq, and FluentAssertions.**
This is the standard stack. Other libraries are not introduced without explicit project-level discussion documented in `CLAUDE_PACKAGES.md` or the equivalent.
Consequence of violation: Test stack fragmentation. Different modules using different test libraries makes shared test infrastructure impossible.

**21. Tests do not connect to production-shaped databases by default.**
Unit tests mock. Integration tests use a clearly-isolated test database. The test runner cannot accidentally hit a real shared database under any configuration.
Consequence of violation: Test runs that mutate real data. We have seen this exactly once. Once was enough.
