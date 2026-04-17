# Recipe Tracker

Personal recipe management, weekly meal planning, and pantry-aware shopping list generation.

Most meal planners ignore what you already have at home. This one checks your pantry first, so shopping lists don't include the 5 cans of tomatoes you already own.

## Stack

- **Backend:** ASP.NET Core 8, Hot Chocolate 13+ (GraphQL), EF Core 8, SQL Server 2022
- **Frontend:** React 18 + TypeScript, Vite, Apollo Client, Tailwind + shadcn/ui, dnd-kit
- **Auth:** ASP.NET Core Identity + JWT bearer
- **Multi-tenancy:** Per-user isolation via EF Core global query filters

## Features (MVP)

- Recipe CRUD with ingredients, instructions, prep/cook time, tags
- Pantry inventory with four tracking modes: Volume, Weight, Count, In-Stock
- Weekly meal planner (7 days × 4 slots, drag-and-drop)
- Pantry-aware shopping list with unit conversion

## Getting Started

### Backend

```bash
docker-compose up -d sqlserver
dotnet ef database update -p src/RecipeTracker.Infrastructure -s src/RecipeTracker.Api
dotnet run --project src/RecipeTracker.Api
```

GraphQL playground: https://localhost:5001/graphql

### Frontend

```bash
cd web
npm install
npm run codegen
npm run dev
```

## Documentation

- [`.docs/project-brief.md`](.docs/project-brief.md) — scope, features, success criteria
- [`.docs/architecture.md`](.docs/architecture.md) — tech stack, structure, tenancy
- [`.docs/data-model.md`](.docs/data-model.md) — entities, enums, ERD
- [`.docs/api-spec.md`](.docs/api-spec.md) — GraphQL schema
- [`.docs/shopping-list-logic.md`](.docs/shopping-list-logic.md) — business logic
- [`CLAUDE.md`](CLAUDE.md) — agent onboarding and conventions

## Status

Docs phase complete. Scaffolding (solution, projects, Hot Chocolate setup, Vite app, Apollo config, Docker Compose) is the next step.
