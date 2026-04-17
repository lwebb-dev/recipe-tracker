# Recipe Tracker

Personal recipe management, weekly meal planning, and pantry-aware shopping list generation.

## Agent Onboarding
If you are an AI coding assistant picking up this repo, **start here**, then read `.docs/` in order:

1. `.docs/project-brief.md` — what we're building, features in scope, success criteria
2. `.docs/architecture.md` — tech stack, structure, tenant isolation strategy
3. `.docs/data-model.md` — entities, enums, ERD, tenancy rules
4. `.docs/api-spec.md` — GraphQL schema, queries, mutations
5. `.docs/shopping-list-logic.md` — non-obvious business logic + rounding rule

## Stack At A Glance
- **Backend:** ASP.NET Core 8, **Hot Chocolate 13+ (GraphQL)**, EF Core 8, SQL Server 2022
- **Frontend:** React 18 + TypeScript, Vite, **Apollo Client v3**, GraphQL Code Generator, Tailwind + shadcn/ui, dnd-kit
- **Auth:** ASP.NET Core Identity + JWT bearer, `[Authorize]` directive on fields
- **Multi-tenancy:** Per-user (TenantId = UserId), EF Core global query filters. **Ingredients are global** — shared reference catalog.

## Commands

### Backend
```bash
dotnet run --project src/RecipeTracker.Api
# GraphQL playground: https://localhost:5001/graphql (Banana Cake Pop UI)

dotnet ef migrations add <Name> -p src/RecipeTracker.Infrastructure -s src/RecipeTracker.Api
dotnet ef database update  -p src/RecipeTracker.Infrastructure -s src/RecipeTracker.Api
dotnet test
```

### Frontend
```bash
cd web
npm install
npm run codegen              # regenerate typed operations from schema
npm run dev
npm run build
npm run test
```

### Local DB
```bash
docker-compose up -d sqlserver
```

## Conventions

### Backend
- Resolvers live in `src/RecipeTracker.Api/GraphQL/{Queries,Mutations}`. Types in `GraphQL/Types`. Inputs in `GraphQL/Inputs`.
- Keep resolvers thin — delegate business logic to domain services. Resolvers load data from `DbContext`, pass it to a service, persist results.
- Never expose entities directly — map to GraphQL types. Hot Chocolate's projection works with properly configured `ObjectType<T>` classes.
- Use `ICurrentUserService` to resolve UserId; do not read claims inside resolvers.
- Tenant-scoped entities implement `ITenantScoped { string UserId }`. EF Core global query filter auto-scopes reads; a `SaveChangesInterceptor` auto-stamps `UserId` on insert. **Do not set `UserId` manually.**
- `Ingredient` is **not** tenant-scoped — global catalog. Any authenticated user can create/read/update; delete blocked by FK restrict when referenced.
- Domain services (`UnitConverter`, `ShoppingListGenerator`) are pure C# — no EF or ASP.NET dependencies.
- Conversion outputs round to 1 dp everywhere.
- Enum values serialize as `SCREAMING_SNAKE_CASE` in GraphQL (Hot Chocolate default).

### Frontend
- `.graphql` files live in `src/graphql/`; `npm run codegen` emits typed React hooks into `src/generated/`.
- Use generated hooks (`useRecipesQuery`, `useCreatePantryItemMutation`) — do not write raw `useQuery` calls.
- Apollo client + auth link configured in `src/apollo/`.
- Client state via Zustand — one store per feature slice.
- Feature folders own routes, components, and operation files.
- Forms: react-hook-form + Zod resolver.
- UI primitives: shadcn/ui components composed with Tailwind utilities.

### Naming
- Backend: PascalCase C# names, `SCREAMING_SNAKE_CASE` GraphQL enums, camelCase GraphQL fields.
- Frontend: camelCase functions/vars, PascalCase components.
- Database: PascalCase tables and columns (EF default).

## Where to Find Things

| Want to... | Go to... |
|------------|----------|
| Add a new entity | `src/RecipeTracker.Domain/Entities/` + register in `RecipeTrackerDbContext` + new migration |
| Add a new query/mutation | `src/RecipeTracker.Api/GraphQL/{Queries,Mutations}/` + input in `GraphQL/Inputs/` |
| Expose a new field on a type | `src/RecipeTracker.Api/GraphQL/Types/<Type>.cs` |
| Change shopping list behavior | `src/RecipeTracker.Domain/Services/ShoppingListGenerator.cs` |
| Add a unit conversion | `src/RecipeTracker.Domain/Services/UnitConverter.cs` |
| Add a page | `web/src/features/<feature>/` |
| Add a client operation | write a `.graphql` file under `web/src/graphql/`, then `npm run codegen` |
| Change the data model | update `.docs/data-model.md` first, then entity + migration |

## Current Status
Docs phase complete (`.docs/` + root `CLAUDE.md` populated). Scaffolding — solution, projects, Hot Chocolate setup, Vite app, Apollo config, Docker Compose — is the next step.
