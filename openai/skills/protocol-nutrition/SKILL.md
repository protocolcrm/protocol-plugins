---
name: protocol-nutrition
description: Use when the operator asks you to build or adjust a client's nutrition plan, hit a calorie or macro target, add a recipe, or fix a meal in Protocol - so the plan matches how this coach actually feeds their clients.
---

# protocol-nutrition - build plans people can actually follow

A nutrition plan that hits a macro target exactly but asks for 2.61 fillets is worthless. Clean,
practical numbers matter more than arithmetic precision.

And before any of that: **coaches do not all model food the same way.** Building a macro-tracked day
plan for a coach whose whole practice is recipes produces something they will not use.

## When to use

- "Cut him to 2800", "build her a meal plan", "swap the breakfast", "recalculate his macros"
- "Add this recipe", "put together next week's menu"

## Step 0 - find out how THIS coach works

Do this once per account, before your first write. `find` with `kind: "nutrition"` and read a few of
their existing plans with `get`. You are looking for which shape they use:

| What you see | Their model | Build it with |
|---|---|---|
| `LABEL` rows each followed by `EMBEDDED_TEMPLATE` meals whose foods sit in `embeddedTemplate.items` | **Meals** (the editor's default) | `build_nutrition`: a label, then a meal row holding its foods |
| `LABEL` rows each followed by loose `MEAL` food rows | **Older plans** (labels and loose foods) | Keep that style when editing; `convertToMeals` only when the coach asks |
| `LABEL` + `EMBEDDED_TEMPLATE` rows embedding library recipes; `nutrients` all zero | **Recipe-book** | `build_nutrition` items, embedding existing recipes |
| Almost no nutrition templates; the food lives in the program's `content` collections | **Menu-as-content** | `build_program` content, not `build_nutrition` |

Match what you find. If a coach's plans have no macros anywhere, that is their model - not missing
data to helpfully fill in. A coach with no plans yet gets **meals**.

## A. Day plans made of meals (the default for anything new)

This is what the coach's editor builds: a **label** is a time of day, and under it sits a **meal**
with a name, an optional method and its foods. The client app shows each meal as one card.

1. Read the client first (see `../protocol-client-review`) - the nutrition profile carries
   preferences, allergies, and dislikes that override any target.
2. Resolve real foods with `manage_library` (`resolve_foods`). You need a `foodId` for a food that
   should count toward macros. A food with no library match can be `{ add: "MEAL", name: "..." }`.
3. Build with `build_nutrition`. The label and its meal are **siblings** at the top level; the foods
   nest **inside the meal**:

   ```
   items: [
     { add: "LABEL", name: "Breakfast 8:00" },
     { add: "EMBEDDED_TEMPLATE", name: "Oats with whey", description: "Soak the oats overnight.",
       items: [ { add: "MEAL", foodId: "<oats>", quantity: 60, measure: "gram" },
                { add: "MEAL", foodId: "<whey>", quantity: 1, measure: "scoop" } ] },
     { add: "LABEL", name: "Lunch 13:00" },
     { add: "EMBEDDED_TEMPLATE", name: "Chicken rice bowl", items: [ ... ] }
   ]
   ```

   Name each meal for what it is ("Chicken rice bowl"), not for the time - the label already says
   when. Macros are computed from the foods; never pass `nutrients`. Only when a meal's foods carry
   no numbers, or the coach gave you its figures, pass `total: { calories, protein, carbs, fat }` on
   that meal; it is kept as set by hand.
4. **A choice** ("pick one of these for lunch") is a `{ add: "GROUP", name: "Lunch", items: [ meal,
   meal ] }` whose items are meal rows. Each meal becomes one option and the day counts their average.
5. Sanity-check as a coach would: are these portions a person would actually weigh out?
6. Read it back with `get` and confirm the meals landed before reporting. A meal's foods come back
   in `embeddedTemplate.items`; to change one, send the meal row as `{ modify: "<mealId>", items: [
   { ref: ... }, { modify: "<foodId>", quantity: 200 } ] }` inside the full list.

**Older plans (labels followed by loose food rows).** Edit them in their own style - loose
`{ add: "MEAL", foodId, quantity, measure }` rows straight after the label - unless the coach asks to
modernise the plan. Then send `build_nutrition` with `templateId` and `convertToMeals: true` and no
`items`: each label's foods become one meal named after the label, and each option of a food choice
("Pick 1") becomes one meal named after the option, exactly as the editor's "Convert to meals" does.

## B. Recipe-book plans (a day assembled from library recipes)

The coach keeps a library of reusable recipes and composes each day out of them.

- A **recipe** is a template with `metadata.templateMode: "SIMPLE"` ("My Meals/Recipes"). Its rows
  may be plain ingredient lines - `{ add: "MEAL", name: "badem" }` with no `foodId`, quantity or
  measure. That is legitimate; do not force library foods on a coach who does not use them.
- A **day plan** is `ADVANCED` and is built as `{ add: "LABEL", name: "Doručak" }` followed by
  `{ add: "EMBEDDED_TEMPLATE", templateId: "<recipe>" }`. Repeat per meal.
- The recipe is **snapshotted** into the day plan, so editing the recipe later does not rewrite
  plans already built from it. The embedded row's own `name` is empty - the name lives inside the
  snapshot, so read `embeddedTemplate.name`, not `name`.
- Recipes written as plain ingredient lines carry no numbers, so such a plan can **total zero
  calories, and that is correct.** Reuse the coach's existing recipes (`find kind=nutrition`,
  `isTemplate: true`) rather than inventing new ones.

## C. Menu-as-content

Some coaches deliver food as a menu attached to the program, not as day plans at all: a collection
named for the menu, whose items are `SUBCOLLECTION`s per meal-time, each holding recipes.

Build it with **`build_program`** `content`, not `build_nutrition`:

```
content: { collections: [ { name: "JELOVNIK", type: "recipes", items: [
  { type: "SUBCOLLECTION", name: "Doručak", items: [
      { type: "NUTRITION_TEMPLATE", nutritionTemplateId: "<recipe>" } ] } ] } ] }
```

For a NEW menu, the recipes can be nutrition templates - create them with `build_nutrition` first,
then reference them by id.

**Editing an existing menu** is different: most real menu options are text only, a
`NUTRITION_TEMPLATE` item whose `nutritionTemplate` snapshot has `id: null`, no rows, and the recipe
in its `description`. There is nothing to reference and nothing for `build_nutrition` to edit. Read
the program with `get kind=program`, change the option's text inside `contentCollections`, and send
ALL of it back as `build_program` `content.collections`, keeping every `id`: only what you changed
is written. Leaving anything out is refused with `contentToDelete` until the coach confirms
(`confirmDelete: true`). Do not swap a text option for a `nutritionTemplateId`: that replaces its
recipe with the library version. Full grammar: `../protocol-reference/references/surface-programming.md`.

## Rules

- **Prefer clean numbers over an exact target.** Whole eggs, whole or half scoops, grams to the
  nearest 5-10. Landing at ~2950 kcal with tidy portions beats hitting 3000 exactly with strange
  fractions - artificial precision reads as machine-generated and coaches will not use it.
- Never override a stated allergy or restriction to make a target work. Report the conflict instead.
- **Supplements get structure/function wording, never a treatment claim.** A supplement row, and any
  note you write beside one, says what it *supports* or *helps maintain*: "supports sleep quality",
  "helps maintain normal magnesium levels". It never treats, prevents, cures or reduces the risk of
  a named condition. "Magnesium supports sleep quality" is fine; "magnesium treats insomnia" or
  "vitamin D prevents osteoporosis" is not, and neither is a dose framed as therapy for something.
  The same holds for the food plan itself: a plan is aimed at how the client performs, recovers and
  feels, never at a disease. Protocol does not diagnose, treat, cure or prevent any disease, and
  nothing you write into a plan may imply it does. Full rule:
  `../protocol-reference/references/guardrails.md`, under House style.
- `items` replaces the item tree. Read, merge, write the whole thing - every row needs one control
  key (`add` / `ref` / `modify`), and a row you omit is deleted. Keep untouched rows with
  `{ ref: "<id>" }`. A write that would delete rows is refused with the rows listed; repeat it with
  `confirmDelete: true` only after the coach agrees.
- **Use a unit the food actually offers, spelled as `resolve_foods` lists it** ("cup, dry", not
  "cup"). A unit the food lacks is saved on its default unit with your quantity - "1 cup" becomes
  "1 g" - and comes back in `unknownMeasures` with the units it does offer. Fix every one of those
  rows before you report the plan.
- **Check `failCount` before you tell the operator it is done.** Rows the server cannot parse are
  skipped and the call still succeeds; `failCount: 3` means three foods are missing from the plan you
  are about to describe. If the whole write is refused, retry with the `templateId` you were handed -
  do not create a second plan.
