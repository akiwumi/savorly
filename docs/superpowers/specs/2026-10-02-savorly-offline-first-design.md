# Savorly Offline-First v1 Design

## Decision

Savorly v1 is a free, standalone, offline-first personal cookbook. It has no accounts, cloud service, video downloads, embedded social feeds, or automatic link extraction. Recipes, user edits, favourites, manual ordering, meal-plan entries, and shopping-list state persist locally.

The data boundary is intentionally shaped for optional future sync: screens use a small `RecipeRepository` interface instead of accessing browser persistence directly. A future authenticated cloud implementation can replace the local implementation without changing recipe screens or their domain model.

## Product scope

Users save an original recipe-source URL and build a trustworthy, editable recipe card. Every recipe requires a dish image. The recipe distinguishes fields sourced from a video, user estimates, and unstated facts, rather than presenting uncertain data as authoritative.

V1 includes:

- Home discovery with search, category entry points, and a saved-from-video recipe.
- Guided local capture and review of a source URL, dish image, title, cuisine, category, ingredients, method, servings, and evidence labels.
- Cookbook search, independent category/cuisine/favourites filters, and manual ordering inside favourites.
- A cookable traditional recipe detail with the original-source link, servings scaling, metric/imperial display, ingredient checklist, numbered method, and sharing fallback.
- Monday–Sunday meal planning, explicit recipe placement, and a compatible-unit shopping-list merger.

V1 excludes social scraping/extraction, account authentication, cross-device backup/sync, hosted recipe-image storage, analytics, comments, grocery checkout, nutrition tracking, and AI chat.

## Visual direction

The supplied project branding is authoritative: `public/brand/savorly-logo.png` is the principal lockup and `public/brand/savorly-app-icon.png` is the app icon source. The visual system follows the approved warm, image-led cookbook direction:

- Cream app background (`#FFFAF1`) and paper recipe surfaces (`#FFFDF8`).
- Coral (`#EF4938`) for primary actions and active states; leaf (`#58753D`) for secondary filters and metadata accents.
- Espresso (`#3A1C16`) text, muted brown (`#957E73`) supporting information.
- Fraunces for dish/page display type; DM Sans for interfaces.
- Rounded, generously spaced image cards. Recipe origin and certainty remain visible without overwhelming cooking content.

`public/brand/concept-home-kitchen.png` is the category-led home-screen reference. The design must not imitate another food brand’s assets, wording, or layout.

## Mobile architecture

The existing mobile runtime remains intact. App-specific composition stays in `src/Prototype.tsx` and `src/prototype.css`.

- Persistent app navigation is a fixed sibling overlay to `MobileScroll`, never a child inside it.
- Scrollable page content reserves bottom space for the navigation and device safe area.
- Text controls use `KeyboardInput`, `KeyboardTextarea`, or `MobileTextField`.
- Navigation and sheets dismiss the simulated keyboard before changing state.
- No modifications are made to protected runtime files.

## Data model and persistence

The recipe domain contains `Recipe`, `Ingredient`, `MethodStep`, `SourceVideo`, `Evidence`, `MeasurementSystem`, and `RecipeCategory`. A recipe records the canonical original URL, source creator/platform metadata where the user supplies it, a locally stored image reference, editable recipe fields, evidence/uncertainty labels, favourite state, and a manual order value when it belongs to a manually ordered view.

`RecipeRepository` is the sole interface consumed by feature screens. Its initial local implementation persists a versioned Savorly data document in browser/device local storage. The repository exposes recipe CRUD, favourite ordering, meal-plan entries, and local preference updates. It never stores video bytes.

A future sync implementation may add authenticated remote storage, conflict handling, image upload, and recovery. Those concerns do not exist in v1 and are not simulated by the local implementation.

## Interaction flows

### Capture and review

The user enters an original source URL, chooses/adds a dish image, then completes or corrects the recipe. Validation blocks saving until a dish image, title, and at least one ingredient exist. The original source opens as a link; video is never copied or rehosted.

### Cooking

Recipe detail presents: hero image; title and cuisine/category; source and evidence; original-video link; servings control; measurement-system control; ingredient checklist; numbered method; sharing; and add-to-plan. Scaling only adjusts known numeric quantities. Unit conversion uses safe, labelled conversions and preserves unknown units as written.

### Planning

The plan shows one Monday–Sunday week and a selected day. Adding a recipe selects a day explicitly and confirms placement. The shopping list combines normalized ingredient names only when units are compatible; incompatible amounts remain separate.

## Quality checks

Tests cover scaling/conversions, recipe validation, manual reorder, meal-plan placement, and shopping-list merging. The complete build verifies protected runtime integrity, TypeScript build, Sites worker output, and the critical journey: review recipe, change cooking values, add it to a selected day, and see it in the meal plan.

Accessibility requirements: meaningful image alt text, labelled icon controls, 44px minimum tap targets, and AA contrast for text and controls.
