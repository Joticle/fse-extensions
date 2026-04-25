# .NET Patterns

Patterns are reusable approaches that have been validated across multiple FSE .NET projects. Each pattern names the situation, sketches the implementation, and notes what it replaces. These are not recommendations — they're the things we reach for first because they've earned that position through use.

## Modular monolith via Razor Class Libraries

**Situation:** You need module isolation without microservice overhead.

**Approach:** Each domain module is a Razor Class Library (`.csproj` with `<RazorCompileOnBuild>true</RazorCompileOnBuild>`) that contains its own `Areas/{ModuleName}/Pages/` tree, its own `{ModuleName}DbContext`, its own services, and its own `ServiceCollectionExtensions.AddModuleServices` method. The web host references all module RCLs and calls each module's `AddModuleServices` extension in `Program.cs`.

**What it replaces:** A single monolithic web project with feature folders. The monolith approach feels lighter at first and becomes unmanageable at the third or fourth feature. The RCL approach has slightly more setup ceremony and dramatically better long-term maintainability.

## Service-only modules as classlibs

**Situation:** A module has no UI — it's purely services, entities, and data access.

**Approach:** Use a plain `Microsoft.NET.Sdk` classlib instead of an RCL. Same `Data/`, `Entities/`, `Services/`, `Extensions/` structure, no `Areas/`. Register identically through `AddModuleServices`.

**What it replaces:** Forcing every module into the RCL shape even when it has no Razor surface. RCL setup for a module that exposes only services is dead weight.

## Cross-module data via raw SQL on consuming module's DbContext

**Situation:** Module A needs to read data that lives in module B's tables.

**Approach:** Module A's service uses its own `ADbContext` to execute raw SQL against module B's tables. The query is written and owned by module A. Module A never references `BDbContext` or any of B's services.

**Implementation sketch:**

```csharp
public async Task<List<SomeReadModel>> GetCrossModuleDataAsync(int tenantId)
{
    const string sql = @"
        SELECT Id, Name, CreatedUtc
        FROM dbo.OtherModuleTable
        WHERE TenantId = @TenantId
        ORDER BY CreatedUtc DESC";

    return await _aDbContext.Database
        .SqlQueryRaw<SomeReadModel>(sql, new SqlParameter("@TenantId", tenantId))
        .ToListAsync();
}
```

**What it replaces:** Injecting `BDbContext` or `IBService` into module A. That coupling is the failure mode standing orders 6 and 7 prevent.

## Wizard state via TempData and a typed wizard data class

**Situation:** Multi-step forms where state must persist across steps without database writes until final submission.

**Approach:** Define a `{Feature}WizardData` class with all fields the wizard collects. Serialize to JSON into TempData on each step's POST. Deserialize on the next step's GET. Final submission writes once to the database.

**What it replaces:** Session state, hidden form fields scattered across pages, or premature database writes that have to be cleaned up if the user abandons the wizard.

## Per-module CSS file loaded in registration order

**Situation:** Each module has its own UI styling needs without stepping on other modules' styles.

**Approach:** Each module ships exactly one `{module}.css` file at `wwwroot/css/{module}.css` (or the project's equivalent path). The web host's `_Layout.cshtml` loads them in module registration order, after the design system files. Module CSS uses only project CSS custom property tokens (e.g., `--cc-*`, `--br-*`, `--fse-*`).

**What it replaces:** A single sprawling `site.css` that everyone edits, or per-page inline styles, or unscoped Bootstrap-style global utilities.

## Auditable entity base hierarchy

**Situation:** Most domain entities need created/modified timestamps, soft delete, and audit trail support.

**Approach:** Three-level base hierarchy:

- `BaseEntity` — `Id`, `PublicId` (Guid), `CreatedUtc`
- `TenantEntity : BaseEntity` — adds `TenantId`
- `AuditableEntity : TenantEntity` — adds `ModifiedUtc`, `IsDeleted`, `DeletedUtc`

Domain entities derive from the level they need. A truly tenant-free table (e.g., `SystemSetting`) derives from `BaseEntity`. Tenant-scoped without audit derives from `TenantEntity`. Most things derive from `AuditableEntity`.

**What it replaces:** Repeating timestamp and tenant fields on every entity, or building a one-size-fits-all base that forces audit fields onto entities that don't need them.

## Service registration via `ServiceCollectionExtensions`

**Situation:** Each module needs to register its DbContext, services, and configurations with DI.

**Approach:** Each module exposes a static `ServiceCollectionExtensions` class with an `Add{ModuleName}Module(this IServiceCollection services, IConfiguration config)` method. The web host's `Program.cs` calls each module's extension in registration order. The order is documented in the host's `Program.cs` and matters because it controls CSS load order and middleware sequence.

**What it replaces:** Manual service registration in `Program.cs` with module-specific knowledge leaking into the host.

## Vanilla JS in `{module}-{purpose}.js` files

**Situation:** A module needs client-side behavior beyond what Razor delivers.

**Approach:** One `.js` file per purpose, named `{module}-{purpose}.js`, loaded from `_Layout.cshtml` or the module's specific page. No inline `<script>` blocks larger than ~10 lines. No frameworks introduced unless the project explicitly approves one in its `CLAUDE_PACKAGES.md`.

**What it replaces:** Bundling React or Vue into a Razor Pages app for two interactive components, or scattering inline scripts across pages until the markup becomes unreadable.

## DbContext per module, single connection string

**Situation:** Each module owns its tables but they all live in the same SQL Server database.

**Approach:** Each module defines its own `{Module}DbContext` configured with the same connection string from configuration. Modules don't share DbContext instances. Migrations don't exist (standing order 1). Each module's DbContext describes only its own tables in `OnModelCreating`.

**What it replaces:** A single god-DbContext that knows about every entity in the system. That god-context becomes a merge conflict magnet and a coupling vector.

## Health check endpoint per host

**Situation:** Production deploys need a stable endpoint the deploy script can hit to confirm the app came up.

**Approach:** Add `app.MapHealthChecks("/health")` in `Program.cs`. The deploy script (`publish.ps1`) hits `https://{host}/health` after MSDeploy completes. A non-200 response aborts the deploy and reports.

**What it replaces:** Hoping the deploy worked, or pinging the home page (which may return 200 even when the app is in a degraded state).
