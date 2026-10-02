# Savorly Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use `superpowers:subagent-driven-development` (recommended) or `superpowers:executing-plans` to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build Savorly: a mobile-first personal cookbook that turns a saved food video into a trustworthy, editable recipe card users can find, cook, favourite, share, and schedule.

**Architecture:** Keep the current React/Vite mobile runtime intact. Split Savorly screens into domain-focused feature modules around a typed recipe model and client repository; add Supabase only after the local flow is stable. Store source URL, source metadata, dish image, user edits, and extraction evidence—not third-party video bytes.

**Tech Stack:** React 19, TypeScript, Vite, existing mobile-app runtime, Radix Icons, Vitest + React Testing Library, Playwright, Supabase Auth/Postgres/Storage and a server-side extraction function.

---

## Product contract

- Save TikTok, Instagram Reels, YouTube Shorts, and web recipe links.
- Every recipe must have a dish image. Prefer platform cover image; otherwise make the user select a frame or upload a dish photo.
- Extract/edit title, cuisine, category, ingredients, method, portions, and original source.
- Classify food as starter, main, salad, soup, side, breakfast/brunch, snack, dessert, cake/baking, sweet, or drink; cuisine is a separate filter.
- Mark facts as **From video**, **Estimated**, or **Not stated**; link each extracted fact to evidence/timestamp where possible.
- Display a traditional recipe card, scale portions, switch Metric/Imperial, favourite and reorder recipes, share the recipe, open original video, and add it to a calendar meal plan.
- Do not download/rehost videos, add social feeds, comments, grocery checkout, nutrition tracking, or AI chat in v1.

## Design system

| Token | Value | Purpose |
|---|---:|---|
| `color.coral.500` | `#EF4938` | Primary action, active navigation, favourite |
| `color.cream.50` | `#FFFAF1` | App background |
| `color.paper` | `#FFFDF8` | Recipe sheets and cards |
| `color.leaf.600` | `#58753D` | Secondary action/filter accent |
| `color.espresso.900` | `#3A1C16` | Headline/body text |
| `color.muted.500` | `#957E73` | Supporting metadata |
| `font.display` | Fraunces | Page and dish titles |
| `font.ui` | DM Sans | Controls and instructions |
| `radius.card` | `18px` | Recipe/calendar surfaces |
| `spacing` | 4/8/12/16/24/32px | Layout rhythm |

Use familiar image-led discovery, simple search/filtering, a full recipe detail view, and cooking/planning flows. Do not reproduce Tasty brand assets, wording, colors, or exact layouts.

## Generated asset registry

| Project asset | Role |
|---|---|
| `public/brand/savorly-logo.png` | Primary logo lockup |
| `public/brand/savorly-app-icon.png` | App icon/store source |
| `public/brand/savorly-icon-wordmark.png` | Alternate icon/wordmark |
| `public/brand/concept-home-feed.png` | Visual-feed reference |
| `public/brand/concept-home-kitchen.png` | Selected category-led home reference |

## Target module structure

```text
src/
  app/AppShell.tsx
  design/tokens.css
  design/components.tsx
  features/home/HomePage.tsx
  features/recipes/{types,recipe-store,RecipeCard,RecipeDetail,conversions}.ts(x)
  features/capture/CaptureReview.tsx
  features/cookbook/{CookbookPage,reorder}.ts(x)
  features/planning/{MealPlanPage,calendar,shopping-list}.ts(x)
  test/
supabase/
  migrations/001_savorly.sql
  functions/extract-recipe/index.ts
public/brand/
```

### Task 1: Recipe domain and fixtures

**Files:**
- Create: `src/features/recipes/types.ts`
- Create: `src/features/recipes/recipe-fixtures.ts`
- Test: `src/features/recipes/types.test.ts`

- [ ] Write failing tests:

```ts
expect(scaleQuantity(400, 4, 2)).toBe(200);
expect(celsiusToFahrenheit(180)).toBe(356);
expect(scaleQuantity(undefined, 4, 2)).toBeUndefined();
```

- [ ] Implement the canonical types: `Recipe`, `Ingredient`, `MethodStep`, `SourceVideo`, `Evidence`, `MeasurementSystem`, and `RecipeCategory`.
- [ ] Implement quantity scaling and only safe metric/imperial conversions. Do not convert unknown units or invent a quantity.
- [ ] Run: `npm run test -- src/features/recipes/types.test.ts`. Expected: PASS.
- [ ] Commit: `git add src/features/recipes && git commit -m "feat: add Savorly recipe domain"`.

### Task 2: Persistent mobile shell and design tokens

**Files:**
- Create: `src/app/AppShell.tsx`
- Create: `src/design/tokens.css`
- Test: `src/app/AppShell.test.tsx`
- Modify: `src/Prototype.tsx`, `src/prototype.css`

- [ ] Write failing test:

```tsx
render(<AppShell initialRoute="recipe" />);
expect(screen.getByRole("navigation", { name: "Primary navigation" })).toBeVisible();
expect(screen.getByRole("link", { name: "Meal plan" })).toBeVisible();
```

- [ ] Render the app-owned bottom nav as a sibling overlay to `MobileScroll`, not as scrollable child content. Give every screen bottom padding equal to nav height plus safe-area height.
- [ ] Use the semantic token table above. Preserve the protected mobile runtime files.
- [ ] Run: `npm run check:runtime && npm run test -- src/app/AppShell.test.tsx`. Expected: PASS on iPhone and Pixel previews.
- [ ] Commit: `git add src/app src/design src/Prototype.tsx src/prototype.css && git commit -m "feat: add persistent Savorly navigation"`.

### Task 3: Home, cookbook, favourites, and ordering

**Files:**
- Create: `src/features/home/HomePage.tsx`
- Create: `src/features/cookbook/CookbookPage.tsx`
- Create: `src/features/cookbook/reorder.ts`
- Test: `src/features/cookbook/reorder.test.ts`

- [ ] Write failing test:

```ts
expect(moveRecipe(["a", "b", "c"], "c", "a")).toEqual(["c", "a", "b"]);
```

- [ ] Implement Home with search, Mains/Drinks/Cakes & Baking/Salads tiles, and a saved-from-video card.
- [ ] Implement Cookbook filters for category, cuisine, favourites, and recently added.
- [ ] Implement drag reorder only in Favourites and named collections. Persist `manualOrder`; do not alter global library sort.
- [ ] Ensure cards show dish image, title, cuisine, meal category, favourite state, and source badge.
- [ ] Run: `npm run test -- src/features/cookbook`. Expected: PASS.
- [ ] Commit: `git add src/features/home src/features/cookbook && git commit -m "feat: add cookbook and favourites"`.

### Task 4: Capture review and source integrity

**Files:**
- Create: `src/features/capture/CaptureReview.tsx`
- Create: `src/features/capture/validation.ts`
- Test: `src/features/capture/validation.test.ts`
- Create: `src/features/recipes/SourceVideo.tsx`

- [ ] Write failing tests:

```ts
expect(validateRecipeDraft({ title: "Jollof Rice", heroImage: "", ingredients: [] }))
  .toContain("A dish image is required.");
expect(validateRecipeDraft({ title: "Jollof Rice", heroImage: "/dish.jpg", ingredients: [] }))
  .toContain("At least one ingredient is required.");
```

- [ ] Build an editable review screen for title, dish image, cuisine, category, ingredients, numbered method, servings, and source evidence.
- [ ] Show `From video`, `Estimated`, and `Not stated` labels. Missing/uncertain values must not look authoritative.
- [ ] Implement `Play original video` as a canonical link/deep link, never an embedded copied video.
- [ ] Run: `npm run test -- src/features/capture`. Expected: PASS.
- [ ] Commit: `git add src/features/capture src/features/recipes && git commit -m "feat: add reviewed recipe capture"`.

### Task 5: Traditional recipe detail, units, portions, and sharing

**Files:**
- Create: `src/features/recipes/RecipeDetail.tsx`
- Create: `src/features/recipes/conversions.ts`
- Test: `src/features/recipes/conversions.test.ts`
- Create: `src/features/recipes/share.ts`

- [ ] Build this order: hero image; title; cuisine/category; source; facts; original-video link; servings stepper; Metric/Imperial toggle; ingredient checklist; numbered method; share; add-to-plan.
- [ ] Keep original creator measurement in a collapsed `Original recipe` section; use rounded practical conversions and let the user edit them.
- [ ] Use Web Share API when available; fallback copies a public recipe URL. Share title, recipe URL, and source attribution—not platform media bytes.
- [ ] Run: `npm run test -- src/features/recipes`. Expected: PASS.
- [ ] Commit: `git add src/features/recipes && git commit -m "feat: add cookable recipe detail"`.

### Task 6: Calendar meal planning and shopping list

**Files:**
- Create: `src/features/planning/calendar.ts`
- Create: `src/features/planning/MealPlanPage.tsx`
- Create: `src/features/planning/shopping-list.ts`
- Test: `src/features/planning/calendar.test.ts`, `src/features/planning/shopping-list.test.ts`

- [ ] Write failing tests:

```ts
expect(placeRecipe("2026-10-06", "jollof", []))
  .toEqual([{ date: "2026-10-06", recipeId: "jollof" }]);
expect(mergeIngredients([
  { name: "rice", quantity: 200, unit: "g" },
  { name: "rice", quantity: 400, unit: "g" },
])).toEqual([{ name: "rice", quantity: 600, unit: "g" }]);
```

- [ ] Build Monday–Sunday calendar, selected-day state, recipe image/chip, and explicit Add recipe action. Selecting a day changes the below-calendar day list.
- [ ] Make `Add to meal plan` open a day picker, add the selected recipe, and confirm placement.
- [ ] Build shopping list merger. Combine only ingredients with matching normalized name and compatible unit; leave incompatible amounts as separate rows.
- [ ] Run: `npm run test -- src/features/planning`. Expected: PASS.
- [ ] Commit: `git add src/features/planning && git commit -m "feat: add calendar meal planning"`.

### Task 7: Production persistence and extraction boundary

**Files:**
- Create: `supabase/migrations/001_savorly.sql`
- Create: `supabase/functions/extract-recipe/index.ts`
- Create: `src/features/recipes/recipe-store.ts`
- Test: `src/features/recipes/recipe-store.test.ts`

- [ ] Define tables: `recipes`, `ingredients`, `method_steps`, `collections`, `collection_recipes`, `meal_plan_entries`. Include `user_id` and RLS policy `auth.uid() = user_id` on every user-owned table.
- [ ] Add client repository interface first; use in-memory implementation for prototype and Supabase implementation for production.
- [ ] Implement extraction endpoint input as canonical URL + permitted source metadata. Return a typed review draft with evidence, timestamp, and confidence. Reject unsupported hosts; never fabricate a complete recipe.
- [ ] Run: `npm run test && supabase db reset`. Expected: PASS and RLS policies applied.
- [ ] Commit: `git add supabase src/features/recipes && git commit -m "feat: persist recipes securely"`.

### Task 8: QA and release gate

**Files:**
- Create: `e2e/savorly.spec.ts`
- Create: `design-qa.md`
- Modify: `README.md`

- [ ] Write critical journey test:

```ts
test("review, scale, plan, and see a recipe on calendar", async ({ page }) => {
  await page.getByRole("button", { name: "Review recipe" }).click();
  await page.getByRole("button", { name: "Add to meal plan" }).click();
  await expect(page.getByLabel("Meal plan calendar")).toContainText("Jollof");
});
```

- [ ] Verify images have meaningful alt text; icon buttons have labels; tap targets are 44px minimum; text and controls meet WCAG AA contrast.
- [ ] Compare iPhone/Pixel screenshots against `public/brand/concept-home-kitchen.png`. Fix P0/P1/P2 differences in navigation persistence, safe areas, typography, tokens, image crops, cards, or calendar.
- [ ] Run: `npm run check:runtime && npm run build && npm run test:sites && npm run test`. Expected: all PASS.
- [ ] Write `design-qa.md` with source path, screenshots, interaction results, findings, and exact `final result: passed`.
- [ ] Commit: `git add e2e design-qa.md README.md && git commit -m "test: verify Savorly cooking flow"`.

## Acceptance checklist

- [ ] Every recipe has an image and opens its original source video.
- [ ] Cuisine and food/drink categories work independently.
- [ ] Portion scaling and Metric/Imperial selection are accurate and editable.
- [ ] Favourites/collections reorder by drag.
- [ ] Nav is fixed and visible on every route.
- [ ] Meal planner has a week calendar and supports day-specific recipe additions.
- [ ] Shares preserve attribution and do not rehost video.
- [ ] All tests, runtime check, production build, Sites test, E2E, and design QA pass.

## Execution order

Tasks 1–2 establish the stable product contract and shell; Tasks 3–6 deliver the usable local app; Task 7 adds secure persistence and extraction; Task 8 blocks release until visual and functional quality pass.

