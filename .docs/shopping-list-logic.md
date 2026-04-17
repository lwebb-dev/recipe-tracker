# Shopping List Generation Logic

## Goal
Given a meal plan and a pantry, produce a shopping list containing only what the user actually needs to buy — accounting for unit conversions between recipe units and pantry units.

## Rounding Rule
**All unit-conversion outputs round to 1 decimal place.** `UnitConverter.Convert(1, Teaspoon, Milliliter)` returns `4.9`, not `4.92892`. This single rule applies everywhere conversions happen: the `convertUnit` query, shopping-list generation, client-side previews. Sums of rounded values are also displayed to 1 dp.

The rounding is acceptably lossy for a consumer app — shopping lists don't need 4-decimal precision, and 1 dp keeps the UI tidy.

## Algorithm

### Step 1 — Aggregate Required Ingredients
For each `MealPlanEntry` in the plan, iterate the recipe's ingredients and scale by servings:

```
required_qty = recipeIngredient.Quantity * (entry.Servings / recipe.Servings)
```

Group the results by `IngredientId`. For each group, collect all `(quantity, unit, quantityType)` tuples.

### Step 2 — Normalize Units Within a Group
Convert every tuple to a common base unit per `QuantityType`:

| QuantityType | Base unit |
|--------------|-----------|
| Volume | Milliliter |
| Weight | Gram |
| Count | (unitless — sum directly) |
| InStock | (boolean — presence only) |

Each conversion output is rounded to 1 dp. Sum the rounded values in the base unit → `required_base` per ingredient.

### Step 3 — Subtract Pantry Stock
Look up `PantryItem` by `(UserId, IngredientId)`.

- **Volume / Weight:** Convert pantry quantity to the same base unit (rounded to 1 dp), then `needed_base = required_base − pantry_base`.
- **Count:** `needed = required − pantry.Quantity`.
- **InStock:** If pantry reports in stock (Quantity ≥ 1), `needed = 0`; else `needed = 1`.

If `needed ≤ 0`, skip this ingredient — nothing to buy.

### Step 4 — Convert Back to Display Units
Choose the most natural display unit based on magnitude, then round to 1 dp:

- **Volume:** `< 1000 ml → Milliliter`; `≥ 1000 ml → Liter` (metric default). Imperial preference: fl oz / cup / pt / qt / gal by magnitude.
- **Weight:** `< 1000 g → Gram`; `≥ 1000 g → Kilogram`. Imperial: oz / lb.
- **Count / InStock:** no conversion.

Store both the numeric quantity and the display unit on the `ShoppingListItem`.

### Step 5 — Persist
Create a `ShoppingList` row linked to the source `MealPlanId`, then insert one `ShoppingListItem` per ingredient with a remaining quantity. Set `Source = "meal-plan"`.

## Unit Conversion Tables

Factors retain full precision internally; **the returned value from `UnitConverter.Convert` is rounded to 1 decimal place**.

### Volume → Milliliter (base)

| Unit | Factor (internal) | Example: 1 unit → ml (rounded) |
|------|-------------------|-------------------------------|
| Teaspoon | 4.92892 | 4.9 |
| Tablespoon | 14.7868 | 14.8 |
| FluidOunce | 29.5735 | 29.6 |
| Cup | 236.588 | 236.6 |
| Pint | 473.176 | 473.2 |
| Quart | 946.353 | 946.4 |
| Gallon | 3785.41 | 3785.4 |
| Liter | 1000 | 1000.0 |
| Milliliter | 1 | 1.0 |

### Weight → Gram (base)

| Unit | Factor (internal) | Example: 1 unit → g (rounded) |
|------|-------------------|-------------------------------|
| Ounce | 28.3495 | 28.3 |
| Pound | 453.592 | 453.6 |
| Kilogram | 1000 | 1000.0 |
| Gram | 1 | 1.0 |

Cross-category conversion (Volume ↔ Weight) is **not supported** — ingredient-specific density would be required, and we don't track that. If a recipe and pantry use mismatched `QuantityType` for the same ingredient, the generator flags a warning on the list and falls back to the full recipe amount.

## Edge Cases

1. **Ingredient used in recipe, absent from pantry:** include full required quantity.
2. **QuantityType mismatch between recipe and pantry:** warn user, include full recipe quantity.
3. **Pantry has more than required:** exclude from list. **MVP does not auto-decrement pantry.**
4. **Recipe ingredient with no unit/quantity (e.g., "salt to taste"):** skip from shopping list; does not block generation.
5. **Duplicate meal plan entries for the same recipe:** quantities sum normally.
6. **Two ingredients with the same name:** impossible — `Ingredient.Name` is globally unique.

## Worked Example

**Meal plan:**
- Monday dinner: Pasta Pomodoro (4 servings) — 400 g tomatoes, 2 tbsp olive oil
- Wednesday dinner: Pasta Pomodoro (4 servings) — 400 g tomatoes, 2 tbsp olive oil

**Required totals (step 1–2):**
- Tomatoes: 800.0 g (Weight, base = g)
- Olive oil: `2 tbsp → 29.6 ml` × 2 entries = **59.2 ml** (Volume, base = ml, 1 dp throughout)

**Pantry:**
- Tomatoes: 500 g
- Olive oil: 250 ml

**After subtraction (step 3):**
- Tomatoes: 800.0 − 500.0 = **300.0 g needed**
- Olive oil: 59.2 − 250.0 = **−190.8 ml** → skipped

**Shopping list output:**
- Tomatoes: 300.0 g

## Implementation Placement
- `RecipeTracker.Domain/Services/UnitConverter.cs` — pure conversion logic, no DB, fully unit-testable. Single entry point: `Convert(decimal value, Unit from, Unit to) → decimal` (1-dp rounded).
- `RecipeTracker.Domain/Services/ShoppingListGenerator.cs` — orchestration. Takes `(MealPlan, Recipes, PantryItems)` collections in, returns `ShoppingList` entity. No DbContext — the GraphQL mutation resolver loads data and passes it in.
- `RecipeTracker.Api/GraphQL/Mutations/ShoppingListMutations.cs` — thin resolver: load data, invoke generator, persist, return.

## Testing Priority
1. **UnitConverter:** every pair in the same QuantityType, verify 1-dp rounding, edge values (0, very large, negative is rejected).
2. **ShoppingListGenerator:**
   - Pantry > required → item excluded
   - Pantry < required → item included with difference
   - Pantry missing → full required quantity
   - Recipe tbsp vs pantry ml → correct conversion subtraction
   - QuantityType mismatch → warning + full quantity
   - Multiple recipes sharing an ingredient → correctly aggregated
3. **Integration test:** full GraphQL flow — `register` → `createMealPlan` → `addMealPlanEntry` ×N → `generateShoppingList`, assert items.
