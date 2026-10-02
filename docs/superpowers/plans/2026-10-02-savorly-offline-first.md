# Savorly Offline-First Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use `superpowers:subagent-driven-development` (recommended) or `superpowers:executing-plans` to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Deliver a polished standalone Savorly mobile cookbook whose recipes, favourites, ordering, meal plan, and shopping list persist locally, with a clean future-sync boundary.

**Architecture:** Keep all application UI in `src/Prototype.tsx` and `src/prototype.css`, preserving the existing phone runtime. Typed recipes and plan entries are persisted by one versioned local repository; every screen mutation travels through that repository. Store source links and recipe metadata, never video bytes.

**Tech Stack:** React 19, TypeScript, Vite, existing `MobileScroll`, `BottomSheet`, keyboard-aware inputs, Radix Icons, browser `localStorage`, existing Playwright runtime checks.

---

## Locked file structure

| File | Responsibility |
|---|---|
| `src/Prototype.tsx` | Recipe types, fixture seed, local repository, routes, and all feature UI/interaction. |
| `src/prototype.css` | Savorly tokens, mobile layout, fixed navigation, cards, forms, sheets, and visual states. |
| `docs/superpowers/specs/2026-10-02-savorly-offline-first-design.md` | Approved product contract. |

Do not modify `src/App.tsx`, `src/main.tsx`, `src/styles.css`, `src/mobile/`, `vite.config.ts`, worker files, or device assets.

### Task 1: Define local domain and repository

**Files:**
- Modify: `src/Prototype.tsx`

- [ ] **Step 1: Add recipe and plan types before all seed data.**

```tsx
type EvidenceState = "from-video" | "estimated" | "not-stated";
type RecipeCategory = "Main" | "Drink" | "Cake & Baking" | "Salad" | "Soup" | "Side" | "Breakfast & Brunch" | "Snack" | "Dessert";
type Ingredient = { id: string; name: string; quantity?: number; unit?: "g" | "ml" | "tbsp" | "tsp" | "piece"; evidence: EvidenceState };
type Recipe = { id: string; title: string; category: RecipeCategory; cuisine: string; minutes?: number; image: string; sourceUrl: string; sourceName: string; servings: number; ingredients: Ingredient[]; method: string[]; favourite: boolean; createdAt: string; manualOrder?: number };
type MealPlanEntry = { id: string; recipeId: string; date: string };
type SavorlyData = { version: 1; recipes: Recipe[]; mealPlan: MealPlanEntry[] };
type RecipeRepository = { load(): SavorlyData; save(next: SavorlyData): void };
```

- [ ] **Step 2: Implement the versioned local repository with a safe fallback.**

```tsx
const STORAGE_KEY = "savorly-data-v1";
const localRepository: RecipeRepository = {
  load() {
    try { const raw = localStorage.getItem(STORAGE_KEY); const value = raw ? JSON.parse(raw) as SavorlyData : null;
      return value?.version === 1 && Array.isArray(value.recipes) && Array.isArray(value.mealPlan) ? value : { version: 1, recipes: seedRecipes, mealPlan: [] };
    } catch { return { version: 1, recipes: seedRecipes, mealPlan: [] }; }
  },
  save(next) { localStorage.setItem(STORAGE_KEY, JSON.stringify(next)); },
};
```

- [ ] **Step 3: Hydrate once and route updates through the repository.**

```tsx
const [data, setData] = useState<SavorlyData>(() => localRepository.load());
const updateData = (change: (current: SavorlyData) => SavorlyData) => setData(current => { const next = change(current); localRepository.save(next); return next; });
```

- [ ] **Step 4: Verify the boundary.**

Run: `npm run check:runtime && npm run build`

Expected: both commands exit `0`.

- [ ] **Step 5: Commit.**

```bash
git add src/Prototype.tsx
git commit -m "feat: add offline Savorly recipe repository"
```

### Task 2: Build safe persistent navigation

**Files:**
- Modify: `src/Prototype.tsx`
- Modify: `src/prototype.css`

- [ ] **Step 1: Compose app-owned fixed layers outside `MobileScroll`.**

```tsx
return <div className="savorly-app">
  <MobileScroll className="app-scroll"><main className="screen-content">{screen}</main></MobileScroll>
  <nav className="bottom-nav" aria-label="Primary navigation">{navButtons}</nav>
  <BottomSheet open={captureOpen} onOpenChange={setCaptureOpen} title="Save a recipe">{captureForm}</BottomSheet>
</div>;
```

- [ ] **Step 2: Replace every raw form control with `KeyboardInput`, `KeyboardTextarea`, or `MobileTextField`; dismiss keyboard before opening any sheet.**

```tsx
<MobileTextField label="Original recipe link" value={draft.sourceUrl} onChange={value => setDraft(current => ({ ...current, sourceUrl: value }))} />
<KeyboardInput aria-label="Recipe title" value={draft.title} onChange={event => setDraft(current => ({ ...current, title: event.target.value }))} />
```

- [ ] **Step 3: Reserve content space for the nav and safe area.**

```css
.savorly-app { min-height: 100%; background: var(--cream); color: var(--espresso); }
.app-scroll { height: 100%; background: var(--cream); }
.screen-content { min-height: 100%; padding: 56px 20px calc(104px + var(--mobile-safe-area-height)); }
.bottom-nav { position: fixed; inset: auto 0 0; min-height: 76px; padding: 10px 12px calc(10px + var(--mobile-safe-area-height)); }
```

- [ ] **Step 4: Verify both device presets manually.**

Run: `npm run check:runtime && npm run dev`

Expected: fixed nav remains visible and content clears iPhone/Pixel safe areas.

- [ ] **Step 5: Commit.**

```bash
git add src/Prototype.tsx src/prototype.css
git commit -m "feat: build safe-area-aware Savorly shell"
```

### Task 3: Implement branded home and cookbook

**Files:**
- Modify: `src/Prototype.tsx`
- Modify: `src/prototype.css`

- [ ] **Step 1: Replace text brand with supplied lockup.**

```tsx
<img className="brand-lockup" src="/brand/savorly-logo.png" alt="Savorly" />
```

```css
.brand-lockup { display: block; width: 122px; height: auto; }
```

- [ ] **Step 2: Derive the visible recipe list from text/category/favourite filters.**

```tsx
const visibleRecipes = data.recipes.filter(recipe => selectedCategory === "All" || recipe.category === selectedCategory).filter(recipe => !favouritesOnly || recipe.favourite).filter(recipe => (recipe.title + " " + recipe.cuisine).toLowerCase().includes(query.trim().toLowerCase()));
```

- [ ] **Step 3: Add category-led home, saved-from-video card, card detail navigation, and accessible search. Use the logo folder assets only for Savorly branding.**

- [ ] **Step 4: Implement manual order only for favourites.**

```tsx
const moveFavourite = (recipeId: string, direction: -1 | 1) => updateData(current => {
  const favourites = current.recipes.filter(recipe => recipe.favourite).sort((a, b) => (a.manualOrder ?? 0) - (b.manualOrder ?? 0));
  const from = favourites.findIndex(recipe => recipe.id === recipeId); const to = from + direction;
  if (from < 0 || to < 0 || to >= favourites.length) return current;
  [favourites[from], favourites[to]] = [favourites[to], favourites[from]];
  const order = new Map(favourites.map((recipe, index) => [recipe.id, index]));
  return { ...current, recipes: current.recipes.map(recipe => order.has(recipe.id) ? { ...recipe, manualOrder: order.get(recipe.id) } : recipe) };
});
```

- [ ] **Step 5: Verify discovery.**

Run: `npm run check:runtime && npm run build`

Expected: search, category, and favourite filter results update; ordering changes only favourites.

- [ ] **Step 6: Commit.**

```bash
git add src/Prototype.tsx src/prototype.css
git commit -m "feat: add branded cookbook discovery"
```

### Task 4: Implement source-aware local capture

**Files:**
- Modify: `src/Prototype.tsx`
- Modify: `src/prototype.css`

- [ ] **Step 1: Define an editable draft and validation.**

```tsx
const emptyDraft = () => ({ title: "", image: "", sourceUrl: "", cuisine: "", category: "Main" as RecipeCategory, servings: 4, ingredients: [] as Ingredient[], methodText: "" });
const draftErrors = (draft: ReturnType<typeof emptyDraft>) => [!draft.image && "A dish image is required.", !draft.title.trim() && "A recipe title is required.", draft.ingredients.length === 0 && "At least one ingredient is required.", !draft.sourceUrl.trim() && "An original recipe link is required."].filter(Boolean) as string[];
```

- [ ] **Step 2: Add ingredient/method editing and evidence select values `From video`, `Estimated`, and `Not stated`. Keep uncertainty labels visible on the saved recipe.**

- [ ] **Step 3: Save only validated recipes locally and route to detail.**

```tsx
const saveDraft = () => {
  const errors = draftErrors(draft); if (errors.length) { setCaptureErrors(errors); return; }
  const recipe: Recipe = { id: crypto.randomUUID(), title: draft.title.trim(), image: draft.image, sourceUrl: draft.sourceUrl.trim(), sourceName: "Original source", cuisine: draft.cuisine.trim() || "Not stated", category: draft.category, servings: draft.servings, ingredients: draft.ingredients, method: draft.methodText.split("\n").map(step => step.trim()).filter(Boolean), favourite: false, createdAt: new Date().toISOString() };
  updateData(current => ({ ...current, recipes: [recipe, ...current.recipes] })); setSelectedRecipeId(recipe.id); setCaptureOpen(false); setRoute("recipe");
};
```

- [ ] **Step 4: Verify capture.**

Run: `npm run check:runtime && npm run build`

Expected: invalid drafts list errors; a valid saved recipe remains after refresh.

- [ ] **Step 5: Commit.**

```bash
git add src/Prototype.tsx src/prototype.css
git commit -m "feat: add source-aware offline recipe capture"
```

### Task 5: Finish cooking and source sharing

**Files:**
- Modify: `src/Prototype.tsx`
- Modify: `src/prototype.css`

- [ ] **Step 1: Add safe scaling and limited conversions.**

```tsx
const scaleQuantity = (quantity: number | undefined, base: number, servings: number) => quantity === undefined ? undefined : quantity * servings / base;
const displayIngredient = (item: Ingredient, base: number, servings: number, system: "metric" | "imperial") => {
  const quantity = scaleQuantity(item.quantity, base, servings); if (quantity === undefined) return item.name;
  if (system === "imperial" && item.unit === "g") return `${Math.round(quantity / 28.3495 * 10) / 10} oz ${item.name}`;
  if (system === "imperial" && item.unit === "ml") return `${Math.round(quantity / 236.588 * 10) / 10} cups ${item.name}`;
  return `${Math.round(quantity * 10) / 10}${item.unit ?? ""} ${item.name}`;
};
```

- [ ] **Step 2: Render hero, title, cuisine/category, provenance, external original-source link, servings stepper, unit control, checklist, numbered method, share, and plan action in that order.**

- [ ] **Step 3: Share source attribution and canonical URL only.**

```tsx
const shareRecipe = async (recipe: Recipe) => { const payload = { title: recipe.title, text: recipe.title + " — source: " + recipe.sourceName, url: recipe.sourceUrl }; if (navigator.share) await navigator.share(payload); else await navigator.clipboard.writeText(recipe.title + "\n" + recipe.sourceUrl); };
```

- [ ] **Step 4: Verify cook view.**

Run: `npm run check:runtime && npm run build`

Expected: unknown quantities are unchanged, only g/ml values convert, and source opens externally.

- [ ] **Step 5: Commit.**

```bash
git add src/Prototype.tsx src/prototype.css
git commit -m "feat: add cookable local recipe detail"
```

### Task 6: Implement weekly plan and shopping list

**Files:**
- Modify: `src/Prototype.tsx`
- Modify: `src/prototype.css`

- [ ] **Step 1: Build Monday–Sunday helpers and selected-day state.**

```tsx
const startOfWeek = (date: Date) => { const copy = new Date(date); copy.setDate(copy.getDate() - ((copy.getDay() + 6) % 7)); return copy; };
const dateKey = (date: Date) => date.toISOString().slice(0, 10);
const weekDays = (anchor: Date) => Array.from({ length: 7 }, (_, index) => { const day = new Date(anchor); day.setDate(anchor.getDate() + index); return day; });
```

- [ ] **Step 2: Use a day-picker sheet before persisting placement.**

```tsx
const placeRecipe = (recipeId: string, date: string) => updateData(current => current.mealPlan.some(entry => entry.recipeId === recipeId && entry.date === date) ? current : { ...current, mealPlan: [...current.mealPlan, { id: crypto.randomUUID(), recipeId, date }] });
```

- [ ] **Step 3: Merge only normalized-name quantities with identical units; leave all incompatible amounts separate.**

```tsx
const mergeIngredients = (items: Ingredient[]) => items.reduce<Ingredient[]>((merged, item) => { const found = merged.find(existing => existing.name.trim().toLowerCase() === item.name.trim().toLowerCase() && existing.unit === item.unit && existing.quantity !== undefined && item.quantity !== undefined); return found ? merged.map(existing => existing === found ? { ...existing, quantity: existing.quantity! + item.quantity! } : existing) : [...merged, item]; }, []);
```

- [ ] **Step 4: Verify planner persistence.**

Run: `npm run check:runtime && npm run build`

Expected: placement appears on selected date after refresh and compatible shopping quantities merge.

- [ ] **Step 5: Commit.**

```bash
git add src/Prototype.tsx src/prototype.css
git commit -m "feat: add local meal planning and shopping list"
```

### Task 7: Release QA and remote handoff

**Files:**
- Modify: `src/Prototype.tsx`
- Modify: `src/prototype.css`
- Create: `design-qa.md`

- [ ] **Step 1: Complete the journey: save a valid recipe, refresh, filter/favourite it, open source, adjust servings/units, add it to a selected day, and confirm the shopping item.**

- [ ] **Step 2: Check useful image alt text, labels for icon controls, 44px targets, AA contrast, keyboard behavior, and both device safe areas.**

- [ ] **Step 3: Run release checks.**

Run: `npm run check:runtime && npm run build && npm run test:sites && npm run test:runtime`

Expected: every command exits `0`.

- [ ] **Step 4: Record results.**

```markdown
# Savorly Design QA
- Source reference: `public/brand/concept-home-kitchen.png`
- Devices checked: iPhone and Pixel 10
- Critical journey: passed
- Runtime check: passed
- Production build: passed
- Sites test: passed
- Playwright runtime test: passed
- Final result: passed
```

- [ ] **Step 5: Initialize, commit, attach the user-provided remote, and push. Stop to reconcile if remote work appears; never force-push.**

```bash
git init
git add .
git commit -m "feat: build Savorly offline-first cookbook"
git branch -M main
git remote add origin https://github.com/akiwumi/savorly.git
git push -u origin main
```

## Plan self-review

Coverage: Tasks 1–2 establish the future-sync-ready local boundary and safe mobile shell. Tasks 3–6 deliver branding, discovery, trustworthy capture, cooking, sharing, planning, and shopping. Task 7 verifies the full offline critical path and carries out the approved remote handoff.

Consistency: every recipe/plan mutation uses `updateData`; source URLs are canonical links; no task adds cloud services, social scraping, hosted media, or copied video.
