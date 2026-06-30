# AGENTS.md

## Project Overview

Slickflow is a .NET 8 open-source BPMN2 workflow engine with AI/LLM integration (OpenAI, QianWen, DeepSeek), multi-agent orchestration, and a code-first workflow definition model. Version 3.5.0.

---

## Solution Structure

```
source/
├── Slickflow.sln              # Main solution (14 projects, net8.0)
├── sfdapi/                    # Production ASP.NET Core Web API (minimal hosting)
├── sfd/ClientApp/             # Frontend BPMN designer (Webpack + sirv + bpmn-js)
├── core/
│   ├── Slickflow.Data/        # Data access layer (Dapper + DapperExtensions.New)
│   ├── Slickflow.Engine/      # Core workflow engine (partial classes)
│   ├── Slickflow.Graph/       # Code-first workflow definition model
│   ├── Slickflow.AI/          # AI/LLM integration module
│   ├── Slickflow.Module*/     # Plugins: Form, Resource, Localize, BusinessRule, External
│   └── Slickflow.WebUtility/  # ResponseResult wrapper, HTTP helpers
├── test/Slickflow.WebApi/     # API-based test project (Swagger UI, Startup.cs pattern)
└── demo/                      # MvcDemo, BizAppService
nuget-test/                    # NuGet package validation tests
database/                      # PostgreSQL schema DDL + seed data
bpmn-files/                    # Sample BPMN files at repo root
nupkgs/                        # Prebuilt NuGet packages (v3.5.0)
```

`sfd` is a Website project (not SDK-style `.csproj`), targets .NET Framework 4.8 in the solution file.

---

## Build & Run Commands

### Backend (.NET 8)
```bash
# Build entire solution
dotnet build source/Slickflow.sln

# Run production API (minimal hosting, top-level statements)
cd source/sfdapi && dotnet run

# Run test API (older Startup.cs pattern, Swagger at /swagger)
cd source/test/Slickflow.WebApi && dotnet run

# Run NuGet package test
cd nuget-test/Slickflow.Engine.Test && dotnet run
```

### Frontend Designer
```bash
cd source/sfd/ClientApp
npm install
npm run dev          # build:watch + sirv on port 5000
npm run build:prod   # Production build (KCONFIG_ENV=prod)
npm run dev:clean    # Kill port 5000 first, then start
```

Frontend uses Webpack (not Vite), sirv as dev server, jQuery + Bootstrap 4 + bpmn-js. Configure backend API URL in `source/sfd/ClientApp/app/kconfig.js`.

### Database Setup (PostgreSQL 15+)
```bash
createdb -h 127.0.0.1 -U postgres wfdbtest2099
psql -h 127.0.0.1 -U postgres -d wfdbtest2099 -f database/wfdbtest2099_pgsql_schema.sql
psql -h 127.0.0.1 -U postgres -d wfdbtest2099 -f database/wfdbtest2099_pgsql_data.sql
```

---

## Key Architecture Patterns

### Data Access: Dapper (NOT Entity Framework)

The engine uses standard NuGet packages: `Dapper 2.1.66` + `DapperExtensions.New 1.7.0`. **Do NOT use EF Core for workflow tables.**

Three-layer pattern:
- `Repository` (IRepository) — generic CRUD via Dapper + DapperExtensions
- `ManagerBase` — abstract class owning `IRepository` (lazy-initialized)
- `Manager` classes (e.g., `ProcessManager`) — extend `ManagerBase`, contain business logic

Transaction pattern (not DI-based):
```csharp
using (var session = SessionFactory.CreateSession())
{
    var transaction = session.BeginTrans();
    try
    {
        // ... Repository operations with session.Connection, session.Transaction
        transaction.Commit();
    }
    catch
    {
        transaction.Rollback();
        throw;
    }
}
```

### DB Init at Startup (Static, Not DI)

All backend projects call this in `Program.cs`/`Startup.cs`:
```csharp
var dbType = Configuration.GetConnectionString("WfDBConnectionType");
var connStr = Configuration.GetConnectionString("WfDBConnectionString");
Slickflow.Data.DBTypeExtenstions.InitConnectionString(dbType, connStr);
```

Supported types: `PGSQL`, `MYSQL`, `SQLSERVER`, `ORACLE`

### Entity Mapping

Custom attributes (not DapperExtensions mapping):
```csharp
[Table("wf_process")]
public class ProcessEntity
{
    [Column("process_id")]
    public string ProcessId { get; set; }
}
```

Dapper is configured with `Dapper.DefaultTypeMap.MatchNamesWithUnderscores = true` for automatic snake_case mapping.

### API Controllers

DI-injected controllers use `IWorkflowService`:
```csharp
public class RuleSetController : Controller
{
    private readonly IWorkflowService _workflowService;
    public RuleSetController(IWorkflowService workflowService) { ... }

    [HttpGet]
    public ResponseResult<IList<RuleSetEntity>> GetList()
    {
        try
        {
            var list = _workflowService.GetRuleSetList();
            return ResponseResult<IList<RuleSetEntity>>.Success(list);
        }
        catch (Exception ex)
        {
            return ResponseResult<IList<RuleSetEntity>>.Error(ex.Message);
        }
    }
}
```

Route pattern: `api/{controller}/{action}/{id?}`

### Response Format

All API responses use `ResponseResult` or `ResponseResult<T>`:
- Status: `0` = default, `1` = success, `-1` = error
- Message, NewId, ExtraData

### WorkflowService (Partial Class)

Split across 7 files by responsibility:
- `IWorkflowService.cs` — interface definition
- `WorkflowService.cs` — constructor, ResourceService + FormService init
- `WorkflowService.Chain.cs` — fluent API (`CreateRunner`, `UseApp`, `UseProcess`, `Start`, `Run`)
- `WorkflowService.ProcessDefinition.cs` — CRUD for process definitions
- `WorkflowService.ProcessExecution.cs` — start/run/withdraw/sendback
- `WorkflowService.Query.cs` — query operations
- `WorkflowService.Resource.cs` — role/user resource operations

### Fluent Chain API (Human Tasks)

```csharp
wfService.CreateRunner("10", "Jack")
    .UseApp("DS-100", "Book-Order", "DS-100-LX")
    .UseProcess("PriceProcessCode")
    .NextStepInt("20", "Alice")
    .Run();
```

### Code-First Workflow Definition (Slickflow.Graph)

```csharp
var wf = new Workflow("Order Process", "OrderProcess_Code");
wf.Start("Start")
  .ServiceTask("Validate Order", "Validate001", "ValidateOrder")
  .LlmService("LLM Enrich", "LLM001")
  .End("End");

var result = await new WorkflowExecutor()
    .UseProcess(wf)
    .AddVariable("OrderId", "ORD-001")
    .Run();
```

### Plugin System

`Slickflow.Module.External` is copied to `Plugins/` folder after build via a custom MSBuild target in `sfdapi.csproj`. The host loads it dynamically.

---

## Naming Conventions

| Layer | Convention | Example |
|-------|-----------|---------|
| C# classes | PascalCase + role suffix | `ProcessManager`, `ProcessEntity`, `WorkflowService` |
| C# properties | PascalCase | `ProcessId`, `CreatedDateTime` |
| DB columns | snake_case | `process_id`, `created_datetime` |
| DB tables | prefix + snake_case | `wf_process`, `ai_model_provider`, `sys_user` |
| Namespaces | Dot-separated PascalCase | `Slickflow.Engine.Business.Entity` |
| Comments | Bilingual (EN + ZH) | `/// 流程实体类` / `/// Process Entity` |

Table prefixes: `wf_` (workflow), `ai_` (AI), `sys_` (system), `vw_wf_` (views)

---

## Testing

**No xUnit/NUnit/MSTest.** Testing is done via API controllers and console apps:

1. `source/test/Slickflow.WebApi/` — ASP.NET Core WebAPI with Swagger UI
   - Controllers: `WfUnitTestController`, `WfProcessTestController`, `WfActivityTestController`
   - Run: `cd source/test/Slickflow.WebApi && dotnet run`, open `/swagger`

2. `nuget-test/Slickflow.Engine.Test/` — console app validating the NuGet package
   - 5 progressive test levels: package verification → assembly info → type reflection → service instantiation → business methods (DB-dependent)
   - Run: `cd nuget-test/Slickflow.Engine.Test && dotnet run`

3. `source/core/Slickflow.Module.External.Tests/` — console app for Supabase integration tests
   - Requires Supabase config in `appsettings.json`
   - Run: `cd source/core/Slickflow.Module.External.Tests && dotnet run`

---

## Common Pitfalls

1. **Don't use EF Core** for workflow tables — the engine uses Dapper exclusively
2. **Don't forget session.Dispose()** — always use `using` blocks with `SessionFactory.CreateSession()`
3. **Check execution status** before committing: `if (result.Status == WfExecutedStatus.Success)`
4. **Custom Dapper fork** — the `source/core/Dapper/` project is a local fork, not the NuGet package
5. **Static DB config** — `DBTypeExtenstions.InitConnectionString()` must be called before any DB operations
6. **JSON casing** — sfdapi sets `PropertyNamingPolicy = null` (preserves PascalCase from C#)
7. **Plugin loading** — `Slickflow.Module.External` must be in `Plugins/` subdirectory (MSBuild copies it automatically)
8. **No CI/CD** — no GitHub Actions or pipeline configs exist; deployment is manual
9. **Database credentials** — `appsettings.Development.json` and `*.env` are gitignored; use placeholder values in `appsettings.json`
10. **Frontend port** — dev server runs on port 5000 by default; API also defaults to 5000 — check for conflicts
11. **Missing DLL directory** — `sfdapi.csproj` references `../DLL/System.Configuration.ConfigurationManager.dll` and `System.Data.SqlClient.dll` which are NOT in the repo; build may fail without them
12. **sfmcp not in repo** — README references an MCP server (`sfmcp`) but it is not present in this source tree
13. **No linter/formatter** — no `.editorconfig`, no StyleCop, no analyzers configured; follow existing code style manually

---

## Docker Deployment

Images on Docker Hub (`besley2096/`):
- `slickflow-all:latest` — all-in-one (ports 5000, 5001, 8090)
- `slickflow-api:latest` — backend API only (port 5000)
- `slickflow-designer:latest` — frontend designer only (port 8090)
- `slickflow-webtest:latest` — test web app (port 5001)

All containers accept env vars: `WfDBConnectionType`, `WfDBConnectionString`

---

## Key References

- Docs: http://doc.slickflow.net (EN), http://doc.slickflow.com (ZH)
- Config guide: `slickflow-configuration.md` (detailed setup steps)
- Data model: `database/slickflow_datamodel-EN.pdm`
- BPMN samples: `source/sfdapi/samples/bpmn/` (business + logic patterns)
- Sample BPMN files: `bpmn-files/` (askforleave, bookbuying, eorder)
