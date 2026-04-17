# Recipe Tracker — Project Brief

## Vision
A personal recipe management and meal planning app that helps users plan weekly meals, track pantry/fridge inventory, and auto-generate shopping lists that only include what they actually need to buy.

## Core Value Proposition
Most meal planning apps ignore what you already have at home. This one checks your pantry first, so shopping lists don't include the 5 cans of tomatoes you already own.

## MVP Features

### Recipes
- CRUD: create, edit, delete recipes
- Ingredients (name, quantity, unit, optional notes like "finely chopped")
- Preparation instructions (markdown-supported)
- Prep time, cook time, servings
- Tags (e.g., "vegetarian", "quick", "italian")
- Optional source URL

### Pantry / Fridge Inventory
- Track owned ingredients with quantity + unit
- Four tracking modes (`QuantityType`):
  - **Volume** (ml, L, fl oz, cup, tbsp, tsp, pt, qt, gal)
  - **Weight** (g, kg, oz, lb)
  - **Count** (discrete units — e.g., "3 apples")
  - **InStock** (boolean — have it or don't)
- Low-stock threshold per item (optional, triggers a flag on dashboard)
- Quick adjust buttons (+ / -) and full edit form
- Filter / search by name, category, low-stock flag

### Meal Planner
- Weekly calendar grid (7 days × 4 meal slots)
- Drag recipes from library onto slots
- Specify servings per slot (defaults to recipe's servings)
- Meal types: Breakfast, Lunch, Dinner, Snack

### Shopping List
- Generate from a meal plan with one click
- **Pantry-aware:** subtracts what's already on hand
- **Unit conversion:** handles recipe-unit vs pantry-unit mismatches (2 tbsp oil vs 250 ml in pantry)
- Check off items as you shop
- Manual add / remove items post-generation

### Auth / Multi-tenancy
- Email + password registration
- JWT bearer auth
- Per-user data isolation (tenant = user)
- Each user sees only their own recipes, ingredients, pantry, plans, lists

## Out of Scope (MVP)
- Household / shared-family accounts — data model leaves room to add later
- Recipe import from URL / HTML scraping
- Nutrition tracking
- Social features (sharing, comments, public recipes)
- Mobile native apps
- Grocery delivery API integrations
- Barcode scanning
- Auto-decrement pantry when shopping list item is checked (tempting but adds scope)

## Success Criteria
1. A new user can register, create a recipe with ingredients, and save it.
2. User can add items to their pantry with quantity + unit + tracking mode.
3. User can drag 3+ recipes onto a weekly planner across multiple days.
4. User clicks "Generate shopping list" and the result correctly subtracts pantry quantities with unit conversion (verified by example in `shopping-list-logic.md`).
5. Two users' data is fully isolated — neither can see the other's recipes/pantry.

## Timeline
1–2 days of focused work with agentic AI assistance. Scope has been trimmed aggressively to fit.
