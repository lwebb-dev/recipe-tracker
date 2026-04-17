# API Specification (GraphQL)

Single endpoint: `POST /graphql`
All operations except `register` / `login` require `Authorization: Bearer {jwt}`.
Server: **Hot Chocolate 13+** (schema-first code), code-first resolvers in `src/RecipeTracker.Api/GraphQL/`.

## Scalars & Enums

```graphql
scalar DateTime        # ISO 8601 UTC
scalar Date            # YYYY-MM-DD
scalar UUID            # GUID string
scalar Decimal         # JSON number, 3 dp max on the wire

enum QuantityType { VOLUME WEIGHT COUNT IN_STOCK }

enum Unit {
  MILLILITER LITER
  TEASPOON TABLESPOON FLUID_OUNCE CUP PINT QUART GALLON
  GRAM KILOGRAM
  OUNCE POUND
}

enum MealType { BREAKFAST LUNCH DINNER SNACK }

enum ShoppingListItemSource { MEAL_PLAN MANUAL }
```

## Types

```graphql
type User {
  id: UUID!
  email: String!
}

type AuthPayload {
  token: String!
  user: User!
}

type Ingredient {
  id: UUID!
  name: String!
  defaultQuantityType: QuantityType!
  defaultUnit: Unit
  category: String
  createdAt: DateTime!
}

type Recipe {
  id: UUID!
  title: String!
  description: String
  servings: Int!
  prepMinutes: Int
  cookMinutes: Int
  instructions: String!
  sourceUrl: String
  ingredients: [RecipeIngredient!]!
  tags: [Tag!]!
  createdAt: DateTime!
  updatedAt: DateTime!
}

type RecipeIngredient {
  id: UUID!
  ingredient: Ingredient!
  quantity: Decimal!
  quantityType: QuantityType!
  unit: Unit
  notes: String
}

type PantryItem {
  id: UUID!
  ingredient: Ingredient!
  quantity: Decimal!
  quantityType: QuantityType!
  unit: Unit
  lowStockThreshold: Decimal
  isLowStock: Boolean!
  updatedAt: DateTime!
}

type Tag {
  id: UUID!
  name: String!
}

type MealPlan {
  id: UUID!
  weekStartDate: Date!
  name: String
  entries: [MealPlanEntry!]!
  createdAt: DateTime!
}

type MealPlanEntry {
  id: UUID!
  recipe: Recipe!
  date: Date!
  mealType: MealType!
  servings: Int!
}

type ShoppingList {
  id: UUID!
  name: String!
  mealPlan: MealPlan
  items: [ShoppingListItem!]!
  generatedAt: DateTime!
}

type ShoppingListItem {
  id: UUID!
  ingredient: Ingredient!
  quantity: Decimal!
  quantityType: QuantityType!
  unit: Unit
  isChecked: Boolean!
  source: ShoppingListItemSource!
}

type ConvertedValue {
  value: Decimal!
  unit: Unit!
}
```

## Connection Types (pagination)

```graphql
type RecipeConnection {
  items: [Recipe!]!
  totalCount: Int!
  hasNextPage: Boolean!
}
```

Hot Chocolate's built-in cursor pagination is available; for MVP we use simple offset pagination via `page` / `pageSize` arguments.

## Queries

```graphql
type Query {
  me: User!

  recipes(search: String, tag: String, page: Int = 1, pageSize: Int = 20): RecipeConnection!
  recipe(id: UUID!): Recipe

  ingredients(search: String, category: String): [Ingredient!]!
  ingredient(id: UUID!): Ingredient

  pantryItems(lowStock: Boolean): [PantryItem!]!
  pantryItem(id: UUID!): PantryItem

  tags: [Tag!]!

  mealPlans(weekStart: Date): [MealPlan!]!
  mealPlan(id: UUID!): MealPlan

  shoppingLists: [ShoppingList!]!
  shoppingList(id: UUID!): ShoppingList

  convertUnit(value: Decimal!, from: Unit!, to: Unit!): ConvertedValue!
}
```

## Mutations

```graphql
type Mutation {
  # Auth
  register(input: RegisterInput!): AuthPayload!
  login(input: LoginInput!): AuthPayload!

  # Ingredients (global catalog)
  createIngredient(input: CreateIngredientInput!): Ingredient!
  updateIngredient(id: UUID!, input: UpdateIngredientInput!): Ingredient!
  deleteIngredient(id: UUID!): Boolean!      # rejected if referenced

  # Recipes
  createRecipe(input: CreateRecipeInput!): Recipe!
  updateRecipe(id: UUID!, input: UpdateRecipeInput!): Recipe!
  deleteRecipe(id: UUID!): Boolean!

  # Pantry
  createPantryItem(input: CreatePantryItemInput!): PantryItem!
  updatePantryItem(id: UUID!, input: UpdatePantryItemInput!): PantryItem!
  adjustPantryItem(id: UUID!, delta: Decimal!): PantryItem!   # +/- button
  deletePantryItem(id: UUID!): Boolean!

  # Tags
  createTag(name: String!): Tag!
  deleteTag(id: UUID!): Boolean!

  # Meal plans
  createMealPlan(input: CreateMealPlanInput!): MealPlan!
  addMealPlanEntry(mealPlanId: UUID!, input: AddMealPlanEntryInput!): MealPlanEntry!
  removeMealPlanEntry(id: UUID!): Boolean!

  # Shopping lists
  generateShoppingList(mealPlanId: UUID!): ShoppingList!
  createShoppingList(name: String!): ShoppingList!
  addShoppingListItem(shoppingListId: UUID!, input: AddShoppingListItemInput!): ShoppingListItem!
  updateShoppingListItem(id: UUID!, isChecked: Boolean!): ShoppingListItem!
  deleteShoppingListItem(id: UUID!): Boolean!
  deleteShoppingList(id: UUID!): Boolean!
}
```

## Inputs

```graphql
input RegisterInput { email: String!, password: String! }
input LoginInput    { email: String!, password: String! }

input CreateIngredientInput {
  name: String!
  defaultQuantityType: QuantityType!
  defaultUnit: Unit
  category: String
}
input UpdateIngredientInput {
  name: String
  defaultQuantityType: QuantityType
  defaultUnit: Unit
  category: String
}

input RecipeIngredientInput {
  ingredientId: UUID!
  quantity: Decimal!
  quantityType: QuantityType!
  unit: Unit
  notes: String
}

input CreateRecipeInput {
  title: String!
  description: String
  servings: Int!
  prepMinutes: Int
  cookMinutes: Int
  instructions: String!
  sourceUrl: String
  ingredients: [RecipeIngredientInput!]!
  tagIds: [UUID!]
}
input UpdateRecipeInput {
  title: String
  description: String
  servings: Int
  prepMinutes: Int
  cookMinutes: Int
  instructions: String
  sourceUrl: String
  ingredients: [RecipeIngredientInput!]   # full replace when provided
  tagIds: [UUID!]
}

input CreatePantryItemInput {
  ingredientId: UUID!
  quantity: Decimal!
  quantityType: QuantityType!
  unit: Unit
  lowStockThreshold: Decimal
}
input UpdatePantryItemInput {
  quantity: Decimal
  quantityType: QuantityType
  unit: Unit
  lowStockThreshold: Decimal
}

input CreateMealPlanInput { weekStartDate: Date!, name: String }
input AddMealPlanEntryInput {
  recipeId: UUID!
  date: Date!
  mealType: MealType!
  servings: Int!
}

input AddShoppingListItemInput {
  ingredientId: UUID!
  quantity: Decimal!
  quantityType: QuantityType!
  unit: Unit
}
```

## Auth
- `register` and `login` are the only unauthenticated operations.
- All other fields are guarded with `[Authorize]` (Hot Chocolate authorization directive).
- Client sends JWT via `Authorization: Bearer {token}` HTTP header.
- `me` resolves from the current JWT's `sub` claim.

## Error Handling
Hot Chocolate returns errors in the standard GraphQL `errors` array:

```json
{
  "data": null,
  "errors": [
    {
      "message": "Quantity must be positive.",
      "path": ["createPantryItem"],
      "extensions": {
        "code": "VALIDATION",
        "field": "quantity"
      }
    }
  ]
}
```

**Error codes used:**
- `VALIDATION` — input failed validation
- `NOT_FOUND` — resource does not exist (or is cross-tenant; don't leak distinction)
- `UNAUTHENTICATED` — missing/invalid JWT
- `FORBIDDEN` — authenticated but not permitted
- `CONFLICT` — unique-constraint or FK-restrict (e.g., delete referenced ingredient)
- `INTERNAL_ERROR` — unhandled; server logs with correlation id

## Example Operations

### Generate a shopping list
```graphql
mutation GenerateList($mealPlanId: UUID!) {
  generateShoppingList(mealPlanId: $mealPlanId) {
    id
    name
    items {
      quantity
      unit
      quantityType
      isChecked
      ingredient { name }
    }
  }
}
```

### Low-stock pantry scan
```graphql
query LowStock {
  pantryItems(lowStock: true) {
    quantity
    unit
    quantityType
    ingredient { name category }
  }
}
```

## Conventions
- Enums serialize as SCREAMING_SNAKE_CASE on the wire (GraphQL convention) — mapped to PascalCase in C#.
- Dates: `Date` scalar is `YYYY-MM-DD`; `DateTime` is ISO 8601 UTC.
- Decimals serialized as JSON numbers, 3 dp max on the wire; conversion results rounded to 1 dp (see `shopping-list-logic.md`).
- Query complexity / depth limits enforced by Hot Chocolate defaults to prevent abusive queries.
