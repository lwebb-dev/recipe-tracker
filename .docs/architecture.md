# Architecture

## Context (C4 Level 1)
A single authenticated user interacts with the Recipe Tracker web app via browser. The SPA sends GraphQL operations to a single endpoint; the server persists to SQL Server. No external integrations in MVP.

## Containers (C4 Level 2)

```
[Browser: React SPA + Apollo Client]
        |
        | HTTPS — GraphQL (HTTP POST to /graphql), Bearer JWT
        v
[ASP.NET Core 8 + Hot Chocolate]
        |
        | EF Core (TCP 1433)
        v
[SQL Server 2022]
```

## Components

### Frontend SPA (`web/`)
- **Vite + React 18 + TypeScript**
- **Apollo Client v3** — GraphQL client, normalized cache, React hooks (`useQuery`, `useMutation`)
- **GraphQL Code Generator** — typed operations + hooks from `.graphql` files
- **Tailwind CSS + shadcn/ui** — UI primitives
- **Zustand** — client-only state (UI toggles, selected week)
- **React Router v6** — routing
- **dnd-kit** — drag-and-drop meal planner
- **react-hook-form + Zod** — forms + validation

### Backend API (`src/RecipeTracker.Api`)
- **ASP.NET Core 8** — host
- **Hot Chocolate 13+** — GraphQL server, code-first schema, `[Authorize]` directive for field-level auth
- **EF Core 8** — ORM, migrations, global query filters for tenancy
- **ASP.NET Core Identity** — user management (built-in `AspNetUsers`)
- **JWT bearer auth** — HMAC-SHA256 signed tokens
- **FluentValidation** — input validation at resolver boundaries
- **Serilog** — structured logging

### Domain (`src/RecipeTracker.Domain`)
Pure C# library — entities, enums, domain services. No EF or ASP.NET dependencies.
- `UnitConverter` — stateless unit math, 1-dp rounded output
- `ShoppingListGenerator` — takes meal plan + recipes + pantry, returns computed list

### Infrastructure (`src/RecipeTracker.Infrastructure`)
EF Core `DbContext`, entity configurations, migrations, `ICurrentUserService` implementation, `SaveChangesInterceptor` for tenant stamping.

### Database
- **SQL Server 2022** via Docker for local dev
- EF Core migrations only — no hand-written SQL

## Tech Stack Summary

| Layer | Technology | Version |
|-------|-----------|---------|
| Frontend framework | React + TypeScript | 18 / 5 |
| Build tool | Vite | 5+ |
| GraphQL client | Apollo Client | 3 |
| Codegen | GraphQL Code Generator | latest |
| UI | Tailwind CSS + shadcn/ui | 3+ / latest |
| Client state | Zustand | 4 |
| Drag/drop | dnd-kit | latest |
| Forms | react-hook-form + Zod | latest |
| Backend framework | ASP.NET Core | 8 |
| GraphQL server | Hot Chocolate | 13+ |
| ORM | EF Core | 8 |
| Database | SQL Server | 2022 |
| Auth | ASP.NET Core Identity + JWT | built-in |
| Validation (BE) | FluentValidation | latest |
| Testing (BE) | xUnit + FluentAssertions | latest |
| Testing (FE) | Vitest + React Testing Library | latest |

## Directory Structure

```
recipe-tracker/
├── .docs/                              # Project documentation
│   ├── project-brief.md
│   ├── architecture.md
│   ├── data-model.md
│   ├── api-spec.md                     # GraphQL schema + operations
│   └── shopping-list-logic.md
├── CLAUDE.md                           # Agent onboarding entry point
├── src/
│   ├── RecipeTracker.Api/              # GraphQL host
│   │   ├── GraphQL/
│   │   │   ├── Types/                  # ObjectType<T> definitions
│   │   │   ├── Queries/                # Query resolvers
│   │   │   ├── Mutations/              # Mutation resolvers
│   │   │   ├── Inputs/                 # Input record types
│   │   │   └── Errors/
│   │   ├── Validators/
│   │   ├── Middleware/
│   │   └── Program.cs
│   ├── RecipeTracker.Domain/           # Entities, enums, domain services
│   │   ├── Entities/
│   │   ├── Enums/
│   │   └── Services/
│   │       ├── UnitConverter.cs
│   │       └── ShoppingListGenerator.cs
│   ├── RecipeTracker.Infrastructure/   # EF Core, DbContext, migrations
│   │   ├── Data/
│   │   │   ├── RecipeTrackerDbContext.cs
│   │   │   └── Configurations/
│   │   ├── Services/
│   │   │   └── CurrentUserService.cs
│   │   └── Migrations/
│   └── RecipeTracker.Tests/            # xUnit tests
├── web/                                # React frontend
│   ├── src/
│   │   ├── apollo/                     # Apollo client setup + auth link
│   │   ├── graphql/                    # .graphql operation files
│   │   ├── generated/                  # codegen output (git-ignored)
│   │   ├── components/                 # Shared UI primitives
│   │   ├── features/                   # Feature slices
│   │   │   ├── auth/
│   │   │   ├── recipes/
│   │   │   ├── pantry/
│   │   │   ├── planner/
│   │   │   └── shopping/
│   │   ├── routes/
│   │   └── main.tsx
│   ├── codegen.ts
│   ├── index.html
│   └── package.json
├── docker-compose.yml                  # Local SQL Server
└── README.md
```

## Authentication Flow
1. `register` / `login` mutations return `AuthPayload { token, user }`.
2. Frontend stores JWT in localStorage (MVP — httpOnly cookie for production hardening).
3. Apollo's `authLink` attaches `Authorization: Bearer {token}` to every request.
4. API JWT middleware validates, populates `ClaimsPrincipal`.
5. `ICurrentUserService` reads `sub` as `UserId`.
6. Hot Chocolate `[Authorize]` directive guards every field except `register` / `login`.
7. Resolvers call EF Core; global query filters restrict tenant-scoped reads to the current user.

## Tenant Isolation Strategy
- Every tenant-scoped entity implements `ITenantScoped { string UserId { get; set; } }`.
- `Ingredients` is **explicitly excluded** — it's shared reference data (see `data-model.md`).
- Indexed `UserId` column on every tenant-scoped table.
- `RecipeTrackerDbContext` injects `ICurrentUserService`.
- Global query filter per entity: `modelBuilder.Entity<T>().HasQueryFilter(e => e.UserId == _currentUser.UserId)`.
- EF Core `SaveChangesInterceptor` auto-populates `UserId` on inserts — resolvers never set it manually.
- **Future-proofing:** the `UserId` indirection is isolated enough that swapping to `HouseholdId` later requires only a membership table and a filter swap.

## Deployment (out of scope for MVP)
- Local dev: `docker-compose up -d sqlserver` + `dotnet run` + `npm run dev`.
- Production target (future): Azure App Service + Azure SQL Database.

## Non-functional Notes
- **Performance:** not a concern at MVP scale. Hot Chocolate's default query complexity limits guard against abusive queries.
- **Security:** HTTPS in prod, PBKDF2 password hashing (Identity default), JWT expiry ~1 hour with refresh-token flow as a stretch goal, Hot Chocolate's built-in introspection off in production.
- **Observability:** Serilog console + file sinks for local; Hot Chocolate's built-in diagnostic events forwarded for request tracing.
