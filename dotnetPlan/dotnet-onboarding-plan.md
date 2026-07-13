# .NET Onboarding Plan — Senior React/Node Developer
### 4 weeks · 2-3 hours/day · Visual Studio 2022 Community · TaskFlowApi sample project

The approach is **lateral transfer first** — everything is anchored to something already known from React/Node.js.

---

## Phase 0 — Setup & Installation (Day 0, ~1 hour)

Do everything here before Week 1 so no time is lost to environment friction later.

### Step 1: Install .NET 8 SDK
- Download from https://dotnet.microsoft.com/download — choose **.NET 8 (LTS)**
- Verify: `dotnet --version` should print `8.x.x`
- Why .NET 8: current LTS, same version most companies are on or migrating to

### Step 2: Install Visual Studio 2022 Community (free)
- Download from https://visualstudio.microsoft.com/vs/community/
- During the installer, select the **"ASP.NET and web development"** workload — installs everything needed for weeks 2-4 in one click
- Optionally tick **"Data storage and processing"** for EF Core tooling
- Mental model: Visual Studio is to .NET what WebStorm is to JS, but free and more deeply integrated

### Step 3: Install the EF Core CLI tool (global)
```bash
dotnet tool install --global dotnet-ef
dotnet ef --version   # verify
```
Used in Week 4 for database migrations (= Prisma Migrate equivalent).

### Step 4: Install LINQPad 8 (free tier)
- Download from https://www.linqpad.net/
- A C# scratchpad — think of it as browser DevTools console but for C#/LINQ
- **Critical for Week 1**: lets you experiment without a full project

### Step 5: Install DB Browser for SQLite
- Download from https://sqlitebrowser.org/
- Used to inspect the TaskFlowApi database during Weeks 2-4 visually
- Mental model: like TablePlus/DBeaver but for SQLite files

### Step 6: Scaffold the TaskFlowApi sample project
```bash
# In a folder of your choice:
dotnet new webapi -n TaskFlowApi --use-minimal-apis
cd TaskFlowApi
dotnet add package Microsoft.EntityFrameworkCore.Sqlite
dotnet add package Microsoft.EntityFrameworkCore.Design
dotnet add package Swashbuckle.AspNetCore
```
Then create the following files manually (the day-by-day tasks will build these out):
- `Models/TaskItem.cs` — entity definition
- `Data/AppDbContext.cs` — EF Core context
- `Services/ITaskService.cs` — interface
- `Services/TaskService.cs` — implementation
- Wire up `Program.cs` with DI registrations and routes

This project is the **anchor for Weeks 2-4** — every task modifies it directly.

### dotnet CLI cheat sheet

| Command | JS equivalent |
|---|---|
| `dotnet run` | `npm start` |
| `dotnet build` | `tsc` / `npm run build` |
| `dotnet watch run` | `nodemon` / `npm run dev` |
| `dotnet add package X` | `npm install X` |
| `dotnet test` | `npm test` |
| `dotnet ef migrations add X` | `prisma migrate dev --name X` |
| `dotnet ef database update` | `prisma migrate deploy` |

---

## Week 1 — C# the Language (Days 1–5)

**Primary tool: LINQPad 8** — all tasks this week run here, no full project needed.

### Day 1 — Types & syntax
- Strong typing, `var` vs explicit types, string interpolation (`$"Hello {name}"` = template literals), nullable reference types (`string?`)
- JS parallel: TypeScript strict mode — same discipline, different runtime
- **Task:** rewrite 5 small JS functions (a formatter, a validator, a sorter, a transformer, a null-check) in C# inside LINQPad

### Day 2 — Classes, records, properties
- `class` vs `record` — records ≈ immutable TypeScript `type`s with structural equality built in; auto-properties (`{ get; set; }`) vs writing getters/setters manually
- JS parallel: TS `interface`/`type` vs `class`; records are like `Object.freeze` with deep equality for free
- **Task:** model 2-3 real domain objects from a previous project as C# classes and records in LINQPad

### Day 3 — Collections & LINQ
- `List<T>`, `Dictionary<K,V>`, LINQ (`.Where`, `.Select`, `.OrderBy`, `.GroupBy`) — identical to `.filter/.map/.sort/.groupBy`, but lazily evaluated and later translatable directly to SQL by EF Core
- JS parallel: array methods are LINQ; key difference is lazy evaluation
- **Task:** take a real JS array pipeline (filter → map → reduce) and reproduce it in LINQ in LINQPad

### Day 4 — Async & error handling
- `async`/`await` — nearly identical syntax to JS; `Task<T>` = `Promise<T>`; `try/catch`, custom exception types
- JS parallel: almost 1:1; main differences are `ConfigureAwait` and that C# exceptions are richer
- **Task:** write an async method that simulates a fetch with `Task.Delay`, throws on failure, and is called with `await` from a `Main` method

### Day 5 — Pattern matching & nullability
- `switch` expressions, `is`/`as` type checks, null-conditional `?.` and null-coalescing `??`
- JS parallel: these exist in modern JS/TS — vocabulary more than new concepts
- **Task:** refactor Day 4's error handling using pattern matching on exception types; replace if/else null checks with `?.` and `??`

**Weekend buffer:** skim the official "C# for JavaScript developers" doc on learn.microsoft.com

---

## Week 2 — ASP.NET Core Basics (Days 6–10)

**Primary tool: Visual Studio 2022** — open TaskFlowApi and work inside it.

### Day 6 — Minimal APIs & routing
- `app.MapGet/MapPost/MapPut/MapDelete` — line-for-line comparable to Express `app.get/app.post`
- JS parallel: this IS Express, just in C#; route handlers support the same `(id, body) =>` lambda syntax
- **Task:** add `GET /api/tasks/completed` to TaskFlowApi that filters for completed tasks

### Day 7 — Middleware pipeline
- `app.Use(...)` chain = Express middleware; same "call next" mental model — `await next(context)` instead of `next()`
- **Task:** write a custom middleware that logs HTTP method + path + response time in milliseconds

### Day 8 — Model binding & validation
- Route params, query strings, and JSON bodies are automatically deserialized into C# parameters/DTOs — no manual `JSON.parse(req.body)`
- `[Required]`, `[MinLength]` data annotations trigger automatic 400 responses
- **Task:** add validation attributes to a `TaskCreateDto` record; confirm an empty title returns 400 via Swagger

### Day 9 — Configuration & environments
- `appsettings.json` + `appsettings.Development.json` = `.env` + `.env.development`
- `IConfiguration` = `process.env`, but strongly typed through the Options pattern
- **Task:** move the SQLite connection string from `Program.cs` into `appsettings.json`; access it via `IConfiguration`

### Day 10 — Swagger UI + endpoint testing
- Swagger UI is a built-in API tester — like Postman but auto-generated from code annotations
- **Task:** exercise every TaskFlowApi endpoint through Swagger; document anything confusing to ask about

---

## Week 3 — Dependency Injection & Project Structure (Days 11–15)

**Primary tool: Visual Studio 2022** — continue with TaskFlowApi.

### Day 11 — Why interfaces + DI
- Interfaces define contracts; DI wires up which implementation you get at the constructor level
- Mental model: React Context, but framework-managed and constructor-injected rather than tree-injected
- **Task:** create a second `ITaskService` implementation returning hardcoded fake data; swap it in `Program.cs` with one line (`services.AddScoped<ITaskService, FakeTaskService>()`) and confirm the rest of the app doesn't notice

### Day 12 — Service lifetimes
- Scoped = one per HTTP request; Singleton = one for the entire app lifetime (≈ a module-level variable in Node); Transient = new instance every injection
- **Task:** add a Singleton-registered request counter service; observe its count persists across requests, unlike a Scoped service

### Day 13 — Project layering conventions
- Endpoints → Services → Data access maps directly to Express routes → controllers → services → models; in .NET the separation is enforced via interfaces
- **Task:** convert one Minimal API route handler into a traditional `[ApiController]` class-based Controller; run both side-by-side

### Day 14 — Real codebase reading
- **Task:** pull the company's actual repo; read one controller + one service top-to-bottom without touching anything; write down 5 questions for a teammate

### Day 15 — Buffer / catch-up + Q&A session

---

## Week 4 — EF Core Data Layer (Days 16–20)

**Primary tools: Visual Studio 2022 + `dotnet ef` CLI + DB Browser for SQLite**

### Day 16 — EF Core basics
- Code-first modeling — C# classes define the schema (opposite of Prisma where the schema file generates the client)
- `DbContext` ≈ Prisma Client; `DbSet<T>` ≈ a Prisma model accessor
- **Task:** add a `Category` entity (`Models/Category.cs`) with a one-to-many relationship to `TaskItem`; register it in `AppDbContext`

### Day 17 — Migrations
```bash
dotnet ef migrations add CategoryAdded
dotnet ef database update
```
- = `prisma migrate dev`; this is also the day to switch from `EnsureCreated()` to proper migrations
- **Task:** generate and apply the migration for `Category`; read the generated migration file

### Day 18 — Querying with LINQ + EF Core
- Filtering with `.Where()`, loading related data with `.Include()` (= Prisma `include:`), projecting into DTOs with `.Select()`
- **Task:** add `GET /api/tasks?categoryId=` using `.Include(t => t.Category)` and a DTO projection

### Day 19 — Testing with xUnit
- xUnit ≈ Jest; `[Fact]` = `test()`; `[Theory]` = `test.each()`; EF Core's in-memory provider lets you test services without a real database
- **Task:** write 3 xUnit tests for `TaskService` — create task (happy path), get by ID not found, delete task

### Day 20 — Integration & end-to-end milestone
- **Task:** connect TaskFlowApi to a small React component — fetch tasks, render them, create one, delete one via the API
- This is the **"it clicked" milestone** where both halves of the stack work together

---

## Ongoing (Post Week 4)

- **Auth:** JWT bearer tokens are the most common pattern; read the company's existing auth middleware rather than starting from scratch — layer this in once fundamentals are solid
- **Infrastructure specifics:** Azure, SQL Server vs Postgres, CI/CD — best learned from the real repo + a teammate
- **Keep an LLM open throughout Weeks 1-2** specifically for "what's the .NET idiom for X I already know in JS" — fastest lateral-transfer technique available

---

## Key Resources

| Resource | URL |
|---|---|
| .NET 8 SDK | https://dotnet.microsoft.com/download |
| Visual Studio 2022 Community | https://visualstudio.microsoft.com/vs/community |
| LINQPad 8 | https://www.linqpad.net |
| DB Browser for SQLite | https://sqlitebrowser.org |
| Microsoft Learn — C# tour | https://learn.microsoft.com/en-us/dotnet/csharp/tour-of-csharp |
| EF Core docs | https://learn.microsoft.com/en-us/ef/core |

---

## Scope

**Included:** C# language, ASP.NET Core Minimal APIs, Dependency Injection, EF Core, xUnit testing, full tooling setup.

**Excluded (post-week-4):** Blazor, gRPC, SignalR, ASP.NET Core Identity/Auth, Azure deployment — all valuable but not needed for the first 4 weeks.

> TaskFlowApi uses SQLite to avoid needing a database server during onboarding. Production will likely use SQL Server or Postgres — same EF Core API, different provider package (`Microsoft.EntityFrameworkCore.SqlServer` / `.Npgsql`).

---

## Verification Milestones

| End of | Success looks like |
|---|---|
| Week 1 | Can read C# fluently; can explain LINQ in terms of JS array methods |
| Week 2 | All TaskFlowApi endpoints tested via Swagger; custom middleware running |
| Week 3 | Can swap a service implementation with one line; understands why it works |
| Week 4 | A working React + .NET end-to-end screen — the integration milestone |
