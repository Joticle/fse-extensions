# .NET Anti-Patterns

These are things that look reasonable in .NET — sometimes even idiomatic — but that consistently cause damage in FSE projects. Each anti-pattern names the trap, explains why it's tempting, describes what actually goes wrong, and points at the correct alternative.

If you find yourself reaching for one of these, stop and re-read the standing orders.

## Using EF migrations because "they're built in"

**The trap:** EF Core ships with a migrations system. It's documented, it's official, it's what the tutorials use. Adopting it feels like the path of least resistance.

**Why it's tempting:** It automates schema changes. It generates SQL for you. It plays well with CI. It's "the .NET way."

**What actually goes wrong:** Migrations encode the schema in C# files that drift from production reality the first time anyone makes a manual SSMS change under pressure. Once the drift exists, the migrations history lies, and every future migration either ignores reality or overwrites it. Recovery requires either rebuilding migration history from scratch (high risk) or abandoning the system entirely (which is what FSE does up front).

**Correct alternative:** Standing order 1. Schema lives in `.sql` files in `db/`. SSMS executes them. EF Core queries the resulting schema. The schema is never described in C#.

## Calling `dbContext.Database.EnsureCreated()` for tests

**The trap:** Integration tests need a database. `EnsureCreated()` will spin one up automatically based on the model.

**Why it's tempting:** It's one line. It avoids the test-database-setup chore. It "just works."

**What actually goes wrong:** It violates standing order 2. The schema it creates is whatever EF infers from the model, which diverges from the SSMS-managed production schema in subtle ways. Tests pass against a schema that doesn't exist in production. Eventually a test passes that should fail, or fails for reasons that don't reproduce in production, and the trust in the test suite collapses.

**Correct alternative:** Maintain a `db/test-setup.sql` script that mirrors the production schema. Run it against a clearly-isolated test database in CI. Tests assume the schema exists; they don't create it.

## Injecting another module's `DbContext` "just for one query"

**The trap:** Module A needs a single piece of data from module B's tables. Injecting `BDbContext` is one constructor parameter and one LINQ query. It feels surgical.

**Why it's tempting:** It's familiar EF Core. The compiler accepts it. The query is type-safe. It works.

**What actually goes wrong:** Standing order 6. Module A is now coupled to module B's entity model. A schema change in B silently changes A's behavior. The dependency graph between modules becomes implicit and impossible to audit. The first injection is harmless; the tenth is a tangle.

**Correct alternative:** Raw SQL through module A's own DbContext (see the cross-module pattern in `patterns.md`), or a contract interface defined in `Platform.Shared` and implemented in module B.

## "I'll just use AutoMapper to keep things clean"

**The trap:** Mapping between entities and DTOs by hand feels repetitive. AutoMapper promises to eliminate that boilerplate.

**Why it's tempting:** It's a well-known library. The setup is small. The first mapping feels like a win.

**What actually goes wrong:** Mapping configuration drifts from the entity definitions. A property added to the entity but not the DTO compiles cleanly and silently fails to map. Debugging mapping issues requires understanding AutoMapper's resolution rules, which are non-obvious. The cure is worse than the disease for a modular monolith where mapping rules are local to a service.

**Correct alternative:** Hand-written mapping methods on the service or extension methods on the entity. They're verbose, they're obvious, they fail at compile time when properties drift, and they live next to the code that uses them.

## Using `@media` instead of `@@media` in Razor Pages

**The trap:** You're writing CSS inside a `.cshtml` file (in a `<style>` block, which we already disallow, but also as a string in some scenarios). You write `@media` because that's the CSS syntax you've used for twenty years.

**Why it's tempting:** Muscle memory. CSS doesn't have an at-sign escaping concept anywhere else.

**What actually goes wrong:** Razor interprets the single `@` as the start of a Razor expression. The page fails to render. The error message points at a parsing failure that doesn't mention CSS. Debugging time is lost.

**Correct alternative:** Standing order 11. `@@media` in any `.cshtml` file. Better still, don't write CSS in `.cshtml` files at all — keep it in `.css` files where the rule doesn't apply.

## Putting connection strings or API keys in `appsettings.json`

**The trap:** Configuration goes in `appsettings.json`. Secrets are configuration. Therefore secrets go in `appsettings.json`.

**Why it's tempting:** It's the obvious place. The `IConfiguration` injection pattern is built around it. It works in development without additional infrastructure.

**What actually goes wrong:** Standing orders 17, 18, 19. The file gets committed. The secret is now in git history and on every developer machine and in every CI artifact. The recovery path is credential rotation across every dependent system.

**Correct alternative:** Azure Key Vault for production secrets, retrieved at startup via the Key Vault configuration provider. `appsettings.Development.json` (gitignored) for local development convenience.

## Forgetting `TenantId` on a query because the calling code is "internal"

**The trap:** A service method runs in a context where the developer is sure only one tenant could ever be involved. Adding the `TenantId` filter feels redundant.

**Why it's tempting:** The current call path makes it look unnecessary. Removing the filter simplifies the query.

**What actually goes wrong:** Standing order 3. The "current call path" is one of many. The next refactor, the next feature, the next AI-assisted session adds a code path that calls the same method without tenant scoping. The query now returns cross-tenant data. The leak is silent until a user notices, which may be never, or may be in a way that causes a regulatory incident.

**Correct alternative:** Every tenant-scoped query filters by `TenantId`. Always. Even when it feels redundant. Especially when it feels redundant.

## Wrapping every service method in `try/catch (Exception)` and swallowing

**The trap:** Defensive programming says catch exceptions to prevent the app from crashing. The catch block logs the error and returns a default.

**Why it's tempting:** It feels safe. It prevents user-facing 500 errors. It looks responsible.

**What actually goes wrong:** Real failures are hidden. The service "succeeds" with a default value, downstream code processes that default as if it were valid data, and the failure surfaces three layers later in a form that doesn't point at the root cause. Debugging time multiplies.

**Correct alternative:** Catch specific exceptions you can meaningfully handle. Let unexpected exceptions propagate to the page model, which renders an error state without exposing stack traces to users. Log structured error information at the boundary where the exception escapes the service layer.

## Using `DateTime.Now` instead of `DateTime.UtcNow`

**The trap:** `DateTime.Now` is shorter, reads naturally, and produces the timestamp the developer sees on their own clock.

**Why it's tempting:** It's the obvious choice when you're not thinking about timezones. Examples in older documentation use it.

**What actually goes wrong:** Standing order 4. The application stores local time. Different servers in different timezones produce inconsistent timestamps. Audit logs lie. Timestamp comparisons break across DST transitions. Reports drift by hours.

**Correct alternative:** `DateTime.UtcNow` everywhere in services and storage. Convert to local time only at the UI rendering edge, using the user's stated timezone.

## Inline `<style>` and `<script>` blocks in `.cshtml` because "it's just for this one page"

**The trap:** A page needs a small style tweak or a tiny bit of JavaScript. Putting it inline keeps the change in one file.

**Why it's tempting:** It's local. It's fast. It avoids touching the CSS or JS file.

**What actually goes wrong:** Standing order 12. The "one page" exception becomes a habit. Six months later, dozens of pages have inline styles and scripts. The design system is bypassed in dozens of places. Consolidating them is a multi-day project that nobody scopes for.

**Correct alternative:** Add the style to the appropriate module's `.css` file. Add the script to the appropriate `{module}-{purpose}.js` file. Even for "just one page." The discipline is the point.

## Trusting `dotnet ef database update` because it ran without errors

**The trap:** A migration command completed successfully. The schema must be correct.

**Why it's tempting:** Green output is reassuring. The tool reports success.

**What actually goes wrong:** This is FSE — there are no migrations. If `dotnet ef database update` ran successfully, something is wrong. Either someone has reintroduced migrations against standing order 1, or the command ran against an unintended database, or the project state is corrupted in a way that needs immediate investigation.

**Correct alternative:** The command should not be available. If you find yourself running it, stop, audit how migrations got back into the project, remove them, and document the incident in `SESSION_STATE.md` lessons learned.
