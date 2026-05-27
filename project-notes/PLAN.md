# Gaming Shop — Final Push Plan

Temporary working file. Delete this whole folder once the project is shipped.

Three repos involved:
- **Backend** — `c:\Users\Gebruiker\Desktop\general\Gaming-Shop` (Express + EJS, current ref for UI)
- **Frontend** — `c:\Users\Gebruiker\Desktop\general\Gaming-shop-frontend` (SvelteKit, target)
- **Sass library** — `c:\Users\Gebruiker\Desktop\general\axlothecook-sass-library` (own utility-class library)

---

## Step 1 — Fix the Sass library so it can be pushed [IN PROGRESS]

Bugs/inconsistencies surfaced from the directory walk. Order: fixes first → push.

- [ ] **`_input.scss` is broken** — uses `searchbar-core` mixin with no `@use` and zero args. Either fix the imports + pass `$bg-color`/`$txt-color`, or delete the file (and remove from `index.scss`). Recommend deletion since it duplicates `_searchBar.scss`.
- [ ] **`_keyframes.scss` duplicate `disappear`** — defined at lines 1–16 and again at 75–90. Delete one.
- [ ] **`_grid.scss` xxl breakpoint bug** — line 198–204 generates `.col-{i}-xs` inside the xxl block. Should be `.col-{i}-xxl`.
- [ ] **Inconsistent relative imports** — most components use `./variables` but `_navbar.scss` uses `../variables`. Standardize on `../variables` since they live in `components/`.
- [ ] **`_colors.scss` half-commented pseudo-selectors** — either keep or delete the `:last-child / :first-of-type / :last-of-type` blocks consistently. Recommend deleting all of them — they introduce surprise specificity.
- [ ] **`_card.scss` is empty** — already commented out in `index.scss`. Either flesh out a basic card or delete the file + the commented `@use` line. Recommend deleting for now (no cards used in `index.html`).
- [ ] **Deprecated Sass color functions** in `_functions.scss` (`lighten`, `darken`, `complement`). Migrate to `color.adjust()` / `color.complement()`. Note: this whole file's helpers are unused by anything else in the library (`_button.scss` references them in *commented-out* code). Recommend deleting `_functions.scss` entirely — drop dead code.
- [ ] **`_searchBar.scss` has hardcoded modal positioning** (`top: 2.8%`, `left: 32.8%`) and many commented debug borders — these belong in the consuming app, not a library. Decide: leave for now (frontend will use it directly) or split out into the frontend's own `custom-styles/`. Recommend leaving for now, revisit in step 4.

After fixes:
- Verify the library still compiles (`gulp` watch task or `npx sass sass-library/index.scss css/index.css`).
- Commit + push to `axlothecook-sass-library` repo.

## Running list of "before git" library items (running)

These are mid-step-1 additions surfaced after item 16. Order: 1 → 5 → 4 → 6 → 2 → 3, then git push.

1. **Big info card** (`.default-big-info-card` in `_card.scss`) — image (background OR `<img>` child) + multi-line text overlay. Ref: Food-For-Dollar `Slider.scss` `.slide` + `.slide-text-wrapper`. Standalone, no JS, also the slide content for item 4.
2. **Component for individual item like individual game page** — ref: `Gaming-shop-frontend/src/routes/games/[slug]/+page.svelte` + `game_card.css`. Two-column layout (image left, info right), title row, basic-info, description paragraph, rating + dev list, option-buttons row.
3. **Component for individual item like individual genre/dev page** — ref: `Individual-Category.svelte` + `category_id.css`. Big image up top with frosted text overlay (`.category-details`) and options, followed by a products grid.
4. **Carousel** (`_carousel.scss` — new file) — combines Food-For-Dollar `Slider.scss` (slides track + ticking dots with 5s progress) and archery `style.css` carousel arrow buttons (`.arrow`, `.left/right-side-arrow`). CSS-only: track layout, slides, dots, progress bar, arrows. JS-needed (note for implementation): auto-advance 5s timer (setInterval), drag-to-swipe (Food-For-Dollar uses Framer Motion `motion` package — for SvelteKit either same package, `svelte-motion`, or hand-rolled pointer events ~30 LOC), click-to-jump dots, arrow click advance. The library only provides the visual structure. — DONE. VERIFIED: can recreate Food-For-Dollar's carousel (with or without arrow buttons) using the library — every visual piece maps; defaults already match FFD's exact values for `.carousel-dot.current` / `.carousel-dot-progress` / dot gap / `.default-big-info-card` h-tag sizes / tint opacity. Two trivial overrides for pixel-parity: `$h1-weight: 400` (FFD doesn't bold h1) and `$auto-advance-duration: 5.1s` (FFD uses 5.1s not 5s). FFD's `.slider-wrapper` scroll-driven `appear` animation is app decoration, not a carousel feature — consumer adds it on `.default-carousel` if wanted. No library changes needed.
5. **Finish `_searchBar.scss`** — current file has two mixins: `game-shop-searchbar` (simple form+input+button on navbar) and `modal-searchbar` (Food-For-Dollar `<dialog>` modal with autocomplete `<ul>`). Issues: `modal-searchbar` mixin defines `.search-bar-wrapper` selector internally but isn't exposed as a `.default-*` class; heavy `@extend` use in the simple variant; hardcoded modal position (`top: 2.8%; left: 32.8%` — leftover); hardcoded colors; debug comments; old naming.
   To do:
   - `.default-search-bar` — a REAL, functional search bar (input + submit), the kind you'd put in a navbar. Parameterized colors. Drop `@extend` → plain CSS.
   - `.fake-search-bar` — a styled div that *mimics* `.default-search-bar` visually but isn't functional. Always visible on the page; on click it opens the search dialog. When the dialog opens, `.fake-search-bar` goes `visibility: hidden` so it doesn't "peek" out from behind the popup (the old bug — couldn't pixel-match the two, so just hide the fake one).
   - `.search-bar-dialog` — a native `<dialog>` (compose item 7's `default-dialog` styling, or at least its backdrop + animation pattern) holding the real input + a suggestions dropdown, dimming the page behind it. The dropdown REUSES `_radioInput.scss` classes (`.default-radio-input-container` + `.default-radio-input-btn`) — picking a suggestion is single-select → navigate to results, same nature as the Gaming Shop price filter. Items animate in like the `<select>` picker.
   - Keep a `simple-search-bar` mixin too if Gaming Shop's basic navbar bar differs from `.default-search-bar` (probably the same — reconcile during the work).
6. **Edit `_notification.scss`** — DONE. `warning`/`.warning-popup` → `default-notification-popup` mixin + `.default-notification-popup` class in `_notification.scss`; all `@extend`s replaced with plain CSS (now only `@use "sass:map"` + `../variables`). Parameterized (`$bg-color`, `$text-color`, `$position`, `$bottom`, `$right`, `$padding`, `$radius`, `$gap`, `$icon-size`, `$message-size`, `$message-weight`, `$duration`, `$z-index`) — defaults reproduce `Mini-Messaging-Board/public/warning.css` exactly (`#fff` via `"white"`, `#232323` via the new `"shark"` colour, `.3rem` gap, `.8rem 7rem .8rem .8rem` pad, `7px` radius, `.8rem`/`600` h2, `3.1s`, `z-index: 5`, `position: absolute`, `bottom`/`right: 5rem`, kept `visibility: visible` + `opacity: 1`, `animation: disappear $duration ease-out`). No icon variants — consumer drops their own inline `<svg>`/`<img>` as first child (library sizes a direct `> svg, > img` child to `$icon-size`); doc comment recommends a colour utility class on the wrapper for "kinds". Dropped the original CSS's two narrow-screen `@media` tweaks (app-side). Also: added `"shark": #232323` to the `$colors` map (alphabetically, between `"seashell"` and `"snow-drift"`) and a "Conventions" note in the library README (insert `$colors` entries alphabetically, check for dupes first).
7. **`_dialog.scss` (new file)** — styling for native HTML `<dialog>` elements (the modal + backdrop). Currently the library only styles a `<dialog>` *inside* `_searchBar.scss`'s `modal-searchbar` mixin (hardcoded position, transparent bg, `::backdrop { rgba(0,0,0,.5) }`). To do: extract a reusable `default-dialog` mixin / `.default-dialog` class for any modal use case — backdrop styling, centered positioning, optional fade-in via `@starting-style` + `allow-discrete`, injectable colors/sizing. Once this exists, `_searchBar.scss`'s modal variant should compose it instead of inlining dialog rules. Surface this dependency when working on item 5.
8. **Audit `_textarea.scss`** — current file has only a bare `textarea { ... }` global element selector with hardcoded styling (white bg, .9rem font, no-resize, etc.). Verify it compiles, behaves correctly, and decide if it should be converted to opt-in via `.default-textarea` class (like `_select.scss`) or stay as a global element rule. Quick pass. — DONE: nothing was broken; wrapped body in a `default-textarea($bg-color)` mixin (default `white-smoke`), kept the global `textarea {}` rule calling it.
9. **Audit `_base.scss`** — check for broken / conflicting / redundant rules in the base reset + element defaults. Quick pass like item 8.
10. **Audit `_navbar.scss`** — DONE. Was clean (no bugs/conflicts/redundancy); aligned it to the library's mixin pattern: `nav-width-100` and `nav-height-100` are now mixins (NO params — values unchanged: `width:100%; min-height:fit-content; display:flex; flex-direction:row` and `height:100%; width:fit-content; display:flex; flex-direction:column`) with the matching `.nav-width-100` / `.nav-height-100` classes. Kept the two globals: `nav ul { list-style-type: none }` and `nav { box-shadow: $base-box-shadow }`. Compiles clean.
11. **Create `_footer.scss` (new file)** — DONE. Built 3 self-contained footer mixins (each = mixin + ready-to-use class, no cross-deps), `@use`d in `index.scss` after `errorPopup`:
   - **`default-footer` / `.default-footer`** — Gaming Shop simple bar. Hardcoded `display:flex; flex-direction:row; align-items:center; justify-content:center`. Injectable: `$padding`(`1rem`), `$gap`(`0`), `$align-content`(`normal`), `$bg-color`(`primary-text`), `$text-color`(`dark-bg-primary`), `$link-color`(`deep-sea-blue` = `#005180`, matches GS `footer h1 a`), `$text-size`(`1.3rem`). Styles `h1..h4,p` + nested `a`.
   - **`store-footer` / `.store-footer`** (+ `store-footer-links`, `store-footer-row`) — FFD-derived. Band (`$bg-color` `supermarket-blue`) wrapping `.store-footer-content` flex column (`$content-margin` `4rem 14rem`, `$content-gap` `4rem`, `$text-color` `white`). `.store-footer-links` = N-col grid (`$columns` 5), each child a flex-col list; `.store-footer-row` = centred flex row (`$gap` `3rem`). Covers FFD `.main-links`/`.document-links`/`.github-link`.
   - **`club-footer` / `.club-footer`** (+ `club-footer-columns`, `club-footer-col`, `club-footer-sponsors`) — archery-derived. Full-width band `flex-flow:column wrap` (`$bg-color` `supermarket-blue`) + styled `hr` divider (`$divider-color` `white`, `$divider-width` `70%`, `border:none`). `.club-footer-columns` = wrapping flex row (`$margin` `5rem 6rem 3rem`); `.club-footer-col` = flex column + `margin-right` (`$gap-to-next` `90px`); `.club-footer-sponsors` = centred wrapping row (`$gap` `100px`).
   Inner link/heading/icon content stays consumer-side (utility classes / scoped rules). Compiles clean.
12. **Animated underline nav-link effects (`_hover.scss`)** — DONE. Removed the old `.navbar-hover-effect`; rewrote `_hover.scss` with TWO effects (each = mixin + ready-to-use class). Common: wrap a text element (`h1..h4 / a / span`) in a `position:relative` box, draw an `::after` underline bar that animates IN on `:hover`, keep it shown when the wrapper has `.current` (the open page), and recolour the text differently on `:hover` vs `.current`. The bar itself is ONE colour (`$bar-color`, default `supermarket-blue`, injectable) — doesn't change between states; only its presence/shape animates.
   - **`drop-underline-effect` / `.drop-underline-effect`** — FFD/Gaming Shop: full-width bar, `visibility:hidden` at `bottom:$bar-rest-offset`(`-3px`), on `:hover`/`.current` → visible + slides to `bottom:$bar-active-offset`(`-7px`). `$hover-text-color`(`supermarket-blue`), `$current-text-color`(`$secondary`) — Gaming Shop uses these as-is. Also `$bar-thickness`(`2px`), `$text-size`(`1.2rem`), `$text-weight`(`400`), `$duration`(`.15s`).
   - **`grow-underline-effect` / `.grow-underline-effect`** — archery: bar grows `width:0→100%` on `:hover`/`.current`; `$origin`(`center`|`left`|`right`, default `center` → `left:50%; translateX(-50%)`; `left` → `left:0`; `right` → `right:0`; invalid → `@error`). `$hover-text-color`(`fun-blue` `#2352A7`), `$current-text-color`(`macaroni-and-cheese` `#efb52f`) — archery's blue-hover/gold-active. Also `$bar-offset`(`-3px`), `$bar-thickness`, `$text-size`, `$text-weight`, `$duration`(`.2s`).
   - New `$colors` added (alphabetically): `deep-sapphire #102E66` (archery `a:active` click colour — currently unused by any component, kept in palette), `fun-blue #2352A7`, `macaroni-and-cheese #efb52f`. Compiles clean.
13. **Create `_menu.scss` (new file)** — DONE. `@use`d in `index.scss` (after `input`, before `navbar`). Shared `menu-panel-base($side, $gap, $padding, $bg-color)` mixin — `position:fixed; top:0; bottom:0`, anchored to `$side` (`left`→`left:0` / `right`→`right:0`; invalid → `@error`), `display:flex; flex-direction:column` HARDCODED, `$gap`(`.5rem`), `$padding`(`1.5rem`), `overflow-y:auto`, `background-color: $bg-color` (default `$secondary`). All slide motion is CSS `transition` on `transform`, no `@keyframes`. Three flavours + helpers:
   - **`expand-on-hover-menu` / `.expand-on-hover-menu`** — admin-dashboard sidebar. Always visible at `$collapsed-width`(`5rem`), `transition:width`; `:hover` → `width:$expanded-width`(`33.333vw`). `.menu-option` = flex row (icon + `<span>` label); label `opacity:0; white-space:nowrap` collapsed, fades to `opacity:1` on parent hover. Params: `$collapsed-width`, `$expanded-width`, `$side`, `$gap`, `$padding`, `$bg-color`, `$duration`(`.3s`). Pure CSS.
   - **`fullscreen-menu` / `.fullscreen-menu`** — mobile hamburger. `width:100vw`; closed `transform:translateX(-100%)` (`+100%` if `$side:right`); `.open` → `translateX(0)`. Needs JS to toggle `.open`. Params: `$side`, `$gap`, `$padding`, `$bg-color`, `$duration`.
   - **`half-screen-menu` / `.half-screen-menu`** (+ `menu-backdrop` / `.menu-backdrop`) — big-screen hamburger. Same as fullscreen but `width:$panel-width`(`50vw`, injectable); ships a sibling overlay `.menu-backdrop` (`position:fixed; inset:0; background:$backdrop-color` default `rgba(0,0,0,.5)`, `opacity:0; visibility:hidden` → `.open` shows it). Toggle `.open` on BOTH panel + backdrop; backdrop-click closes.
   - **`menu-toggle` / `.menu-toggle`** — bare clickable square (`$size` default `2.5rem`) for the hamburger icon; drop `<svg>` inside.
   HTML shapes are in the file's doc comments. Used `@if`/`@else` for the `$closed-x` translate value (the `if()` function triggers a Sass deprecation warning). Compiles clean.

14. **Post-step-1 tweaks (made while building the demo page)** — DONE.
   - **`_select.scss`** — added `option` rules the library was missing (it only styled the `<select>` box before, so options fell back to ugly `appearance:base-select` rendering): `option {background-color:$bg-color; color:$txt-color; padding:.3rem; transition:.2s; > svg/img {flex-shrink:0}}`, `option:hover, option:checked {background-color:$option-hover-bg}` (new param `$option-hover-bg`, default `$secondary` — matches Gaming Shop's `option:hover`), `option:not(:last-of-type) {border-bottom:none}`, and **`option::checkmark {display:none}`** (kills the browser's selected-option checkmark — Gaming Shop abandoned it). Box colours were already `$primary`/`$text` which already matched Gaming Shop's `--main-bg-clr`/`--main-txt-clr` (those are `#7f011f`/`#f5ebd0`, NOT the other way round). Works with `multiple`. Consumer supplies their own selected/unselected icon swap — library doesn't style a "checkbox square".
   - **`_radioInput.scss`** — `.default-radio-input-btn` now lays out a row of: optional leading icon (`> svg, > img, > div {flex-shrink:0}` — the `<div>` so a consumer's svg-wrapper / fake square doesn't shrink), primary `<h3>`, and `h3 + h3 {margin-left:auto}` so a second `<h3>` (a count like `[ 12 ]`, mirroring Gaming Shop's genre/dev `<select multiple>` options) sits flush right. No checkbox styling added.
   - **`_dialog.scss`** — fixed the recommended-usage comment: was `bg-jet-grey-dark-3 text-white` (a color-only util → renders a default-sized button, no padding/font); now `btn-jet-grey text-white` (a real `btn-*` component class), with a note that `btn bg-jet-grey-dark-3 text-white` is the way to get a *darker* shade. Re-verified: `btn-{color}` exists for every `$colors` entry; `.bg-/.text-/.border-{color}-dark-N/-light-N` shade variants exist for every colour too (not just theme colours) — but there is NO `btn-{color}-dark-N`.
   - **Rename:** library demo page `index.html` → `demo.html` (`git mv`; nothing referenced it by name; gulpfile globs `*.html`).
   - Full library compile re-verified clean (no deprecation warnings); regenerated `css/index.css` (it had been a purged gulp-build output).

15. **Library changes made DURING the frontend port (Step 5 rule — fix library → push → `npm update` in frontend)** — IN PROGRESS, not yet pushed.
   - **`_select.scss`** — `$option-padding` default `.5rem` → **`.3rem .5rem`** (matches Gaming Shop's original `option { padding: .3rem .5rem }`). Recompiled clean (`.default-select option { padding: 0.3rem 0.5rem }`).
   - **Deleted `_option.scss`** (`git rm`) + removed its `@use 'components/option';` from `index.scss`. It was a redundant older file styling bare `<option>` globally — overlapped/contradicted `_select.scss` (its `option::checkmark { content: "☑️" }` fought `_select.scss`'s `option::checkmark { display: none }`, and that emoji-checkmark would leak into every consuming app's bare `<option>`s). Now `<option>` styling is opt-in via `class="default-select"` only, which is what Gaming Shop will use. Consequence: bare `<option>`s lose the `8px` top-left/bottom-left corner radii `_option.scss` had (`.default-select option` doesn't have them) — minor; add back in `_select.scss` if wanted. Recompiled clean — no bare-`option` rules emitted.
   - **TODO when ready:** commit + push `axlothecook-sass-library` (these two changes + the earlier item-14 tweaks + the rename + `package.json` `gulp-purgecss`→devDeps), then in the frontend `npm update axlothecook-sass-library` (or `npm install github:axlothecook/axlothecook-sass-library --force` if `npm update` no-ops; clear `node_modules/.vite`; restart `npm run dev`).

**STEP 1 COMPLETE** — all "before git" items + the hover-link component + `_menu.scss` + the post-step-1 tweaks (item 14) are done; final library compile is clean. Item 15 = ongoing library changes the frontend port reveals (Step 5). User commits + pushes the `axlothecook-sass-library` repo to GitHub themselves (one push done earlier; another due for items 14 + 15 + the rename + `package.json` `gulp-purgecss`→devDeps move).

Colours added to `$colors` across step 1 (all alphabetical, dedup-checked): `alabaster #f3f6f8`, `shark #232323`, `deep-sapphire #102E66`, `fun-blue #2352A7`, `macaroni-and-cheese #efb52f`. README has a "Conventions" note: insert `$colors` entries alphabetically, check for dupes first. Also `package.json`: `gulp-purgecss` moved from `dependencies` → `devDependencies`.

Library demo page (`axlothecook-sass-library/demo.html`) showcases: colour palette, text colours, bg variations, colour shades, font sizes, buttons, cards (game/genre/background-image — using a temp `sass-library/images/aura.png` that gets deleted after screenshots), then a component showcase: search bar, select & radio-input "fake select" (fruit options), dialog, error popup, notification, spinner, footer — with `position:static`/`animation:none` inline overrides so the normally-fixed/animated ones render in flow for screenshots.

## NEXT: import the library into the Gaming Shop frontend (start of Step 2/3)
Order: (1) `cd Gaming-shop-frontend; npm install -D sass` (2) `npm install github:axlothecook/axlothecook-sass-library` (installs straight from GitHub — does NOT need npm-publish) (3) create `src/lib/styles/index.scss` with any `@use '...' with (...)` variable overrides + `@use 'axlothecook-sass-library/sass-library' as *;` (4) import that once in `src/routes/+layout.svelte`'s `<style lang="scss">` (5) start removing `src/styles/*.css`, port pages to library classes using `Gaming-Shop/src/views/*.ejs` as the visual ref + the "Frontend port — notes & class renames" section below. Fallback if the `@use` path doesn't resolve: explicit `@use '../node_modules/axlothecook-sass-library/sass-library' as *;` or a `kit.alias`.

## Step 1b — Item 16: more library components (added mid-step-1) [PENDING]

Four new components to add to the Sass library, modelled on existing frontend code:

1. **Page-wide loading spinner** — full-screen overlay. Role model: `.loading-overlay` + `.spinner` in `Gaming-shop-frontend/src/routes/+layout.svelte` (and `src/styles/loading.css`). Covers the whole viewport, dimmed backdrop, centered spinner, `@keyframes spin`.
2. **Item-level loading spinner** — small overlay for individual images / fetched lists. Role model: `.img-spinner-overlay` in `loading.css` + usage in `/games +page.svelte` and the genres/developers fetch. Absolute-positioned inside a relative parent, smaller spinner.
3. **Error-page classes** — for when the server redirects to a full error page. Role model: `src/routes/+error.svelte` (+ `src/styles/error-page.css`). Classes: `.error-page` (centered column layout), the `.content { display: block }` override, the link styling.
4. **Error-popup classes** — toast-style error notifications. Role model: `src/lib/reusable/ErrorPopup.svelte` (+ `src/styles/error-popup.css`). Classes: `.error-wrapper` (fixed-position list, `disappear` animation) + `.error-popup` (the individual `<li>` — orange text, dark bg, rounded). NOTE: library already has `_errorPopup.scss` with an `.error-wrapper` — reconcile/extend it rather than duplicate. Also the `disappear` keyframe already exists in `_keyframes.scss`; the `spin` keyframe will need adding.

## Step 2 — Install Sass tooling in the SvelteKit frontend [PENDING]

SvelteKit + Vite uses `vite-plugin-svelte` which auto-detects preprocessors via `svelte-preprocess`. To enable SCSS:

```bash
cd c:\Users\Gebruiker\Desktop\general\Gaming-shop-frontend
npm install -D sass
```

That's usually it — `<style lang="scss">` in `.svelte` files Just Works once `sass` is on the dev deps. Confirm `svelte.config.js` has `vitePreprocess()` (it does by default with `npm create svelte@latest`).

## Step 3 — Import the Sass library into the frontend [PENDING]

Two options to wire it up. Pick one:

- **A. Local path install** (fastest while iterating across both repos):
  ```bash
  npm install --save ../axlothecook-sass-library
  ```
  Then `@use '../node_modules/axlothecook-sass-library/sass-library' as *;` from a `.svelte` `<style lang="scss">` or from `src/app.scss`.
  Downside: Vite may not always re-watch `node_modules` reliably.

- **B. Direct relative import** (simplest, no install):
  Add path alias in `svelte.config.js` → `kit.alias = { 'sass-lib': '../axlothecook-sass-library/sass-library' }`. Then `@use 'sass-lib' as *;` everywhere.
  Downside: assumes folder layout next to the frontend; doesn't survive deployment unless we copy or publish.

- **C. Copy into frontend** (best for deployment): symlink or `cp -r` the `sass-library/` into `Gaming-shop-frontend/src/lib/sass-library/`. Trade-off: changes have to be synced manually until published to npm.

**Recommend A** for now (cheap to install, easy to override variables via a wrapper SCSS file). When publishing to Railway, switch to a real npm publish OR option C.

Plan:
1. Create `src/lib/styles/index.scss` in the frontend that overrides `$primary`, etc., then `@use 'axlothecook-sass-library/sass-library' as *;`.
2. Import that one file in `src/routes/+layout.svelte` via `<style lang="scss">@use '$lib/styles';</style>`.
3. Remove existing global stylesheets one by one, replacing classes/styles with library utilities.
4. Use the EJS templates in `Gaming-Shop/src/views/*.ejs` as visual reference for what each Svelte page is meant to look like.

## Frontend port — notes & class renames to apply (from Sass library work)

Markup class renames required (library uses new names):
- `.select-imitation` → `.default-radio-input-container`
- `.select-radio-btn` → `.default-radio-input-btn`
- `#error-wrapper` (ID) → `.default-error-popup-wrapper` (class) — library exposes the class form; <li> children styled automatically, no class needed on them
- global `select { ... }` element rule → put `class="default-select"` on `<select>` elements (picker rules stay global, no change needed there)
- Cards (frontend → library):
  - game card `.product-card-wrapper` → `.default-informative-card`; child `.product-card-img` → `.card-img`; `.product-card-info` → `.card-body`
  - the game-card body internals (`.developer-card-info`, `.product-card-price` + its `h4:first/last-of-type`) — NOT in the library; rebuild inside `.card-body` with utilities / scoped rules
  - genre card `.category-card` (when it has text) → `.default-minimal-info-card`; child `.genre-card` → `.card-overlay`
  - dev card `.category-card` (image only, `path === 'developers'`) → `.default-only-image-card` — and in Gaming Shop pass `$placeholder-bg: #fff` (or the white color) instead of the dark-charcoal default
  - `.loaded` (on the `<img>`) — unchanged, it's a state class
  - optional: switch dev cards to `.default-flip-card` (image flips on hover to reveal a back face) — new capability, not a 1:1 port
- Individual game page (`game_card.css` `.game-card-content` / `.product-content` / `.product-image` / `.product-info`) → use `.default-split-card`. **Inject `$border: variables.$base-border-thickness solid variables.$text` (the `1px solid $text` border) and `$grid-cols: "1fr 2fr"`, `$padding: 2rem`, `$img-aspect-ratio: "6 / 7"`** to match the current look. The `<div class="product-info">` internals (`.title-div`, `.basic-info`, the h1/h2/h3/h4/p sizing, the rating line, the dev list) — NOT in the library; rebuild inside `.split-card-content` with utilities / scoped rules.
- Individual genre/dev page (`category_id.css` `.category-img` / `.category-details` + the `.products-list` grid box) → use `.default-profile-card` for the banner: `.profile-card-img` holds the image + a `.card-overlay-frosted` containing the name (`<h1>` / styled heading) + "Contains N games" + the Update/Delete `.option-buttons`. Pass `$img-object-fit: contain` (already the default), `$img-aspect-ratio: "3.7 / 1"` (already the default). The `.products-list` below (bordered rounded box wrapping a grid of game cards) — NOT a library component; compose with `.border-primary-text .br-xl .p-2 .grid-cols-3` (+ responsive `.grid-cols-2` / `.grid-cols-1` via breakpoint mixins).
- Genres card on the `/genres` list page (`.category-card` + `.genre-card`) → use `.default-minimal-info-card` with a `.card-overlay-frosted` inside (the genre name as the overlay text).
- Loading: `.loading-overlay` / `.spinner` / `.img-spinner-overlay` — all now in the library (`_spinner.scss`), classes unchanged. `.spinner-sm` available for a smaller variant.
- Error page: `.error-page` → `.default-error-page` (drops the debug `border: 1px solid blue`). The `.content { display: block }` override on the error route — not a library class; apply `display: block` there directly or use `.display-b`.
- (more will accumulate as the library evolves — keep adding here)

Things that deliberately stay app-side (no library equivalent — rebuild with utilities in the component's `<style lang="scss">`):
- The games-page filter sidebar: `.filter-category`, `.filter-form`, `.filters-wrapper`, `.long-radio-input-div` / `.radio-input-div` outer layout (the radio-input *rows* themselves ARE in the library via `_radioInput.scss`).
- Page section layouts: `.main-content`, `.product-content`, `.title-and-add-btn-div`, `.option-buttons`, `.category-wrapper`, `.category-img`, `.content`, `.main`, etc. — all just flex/grid + gap + padding + border combos → express as stacked utility classes or scoped rules.
- Responsive resizing (h2 shrinks at 935px, etc.) — goes inside each component's `<style lang="scss">` (next to the base rule it overrides), NOT in `src/lib/styles/index.scss` (that's only for the library import + truly-global rules). No responsive utility classes in the library (decided: option A). **OPEN — breakpoint strategy not yet chosen (see "Open questions" → "Frontend port — breakpoint strategy"). Until decided, don't port responsive rules.** Library's `_breakpoints.scss` provides `min-width` (mobile-first) mixins only: `xxs`(0) `xs`(480px) `sm`(720px) `md`(960px) `lg`(1024px) `xl`(1200px) `xxl`(1450px), plus `breakpoint($bp)`. Gaming Shop's existing CSS is all `max-width` (desktop-first) with oddball values (1780/1453/1420/1280/1024/935/907/750/675/480/365/337/320).
- The sort-filter underline toggle (`.radio-btn` animated underline) — NOT ported into the library; if reused later, generalize the existing `.navbar-hover-effect` in `_hover.scss`.

## Step 4 — Smooth out inconsistencies [PENDING]

**Known issue:** `/developers +page.svelte` — array often doesn't render on client navigation but does on refresh.

Hypothesis (verify first): SSR `load` returns data that the client-side navigation isn't getting because either (a) the load function only runs server-side and the client is reusing stale state, or (b) the `+page.server.ts` is missing an `export const ssr = true` / `csr` setting, or (c) the data is being fetched in the component instead of in `load`. Need to check the actual files before guessing.

Plan:
1. Read `gaming-shop-frontend/src/routes/developers/+page.svelte` and any `+page.ts` / `+page.server.ts` next to it.
2. Compare against another working route (e.g. `games`).
3. Fix the data flow so the page hydrates the same way on nav vs refresh.
4. Audit all routes for the same pattern.

## Step 5 — Push library changes (when needed) before continuing [PENDING]

Standing rule for the rest of the project: any time the frontend reveals a missing/wrong rule in the library, fix the library first, push it, then pull/reinstall in the frontend before continuing.

## Step 6 — Final review / leftover notes [PENDING]

Pulled forward from memory + this conversation:

- **Loading screen Phase 2** — image-loading gap. Three paths were on the table; user to pick. (See `project_loading_phase2.md` in user memory.)
- **Games filter persistence** — multi-select Name filter with Set-based client state + URL param restoration. Verify both the client and server pieces are wired.
- **Basic project features doc** — flesh out the third bullet ("Basis for CSS") in the `basic-project-features` skill once Sass refactor is fully done.

## Step 7 — Deploy to Raspberry Pi [PENDING]

**Decision (2026-05-17):** Railway is dropped. The project deploys straight to a
self-hosted Raspberry Pi — the Pi is both the app host and the database host.
(This merges what were previously Step 7 "Railway" + Step 8 "CI/CD to Pi".)

**➡️ FULL DEPLOYMENT GUIDE: see `.project-notes/DEPLOY.md`** — a learning-oriented
companion doc explaining Docker, Docker Compose, Cloudflare Tunnel and GitHub
Actions (the *why* behind each choice), with a suggested learning order and the
concrete pre-deploy checklist. The user wants to LEARN these tools (not paste
configs blind), so the step-by-step lives there. The notes below are the
high-level summary; `DEPLOY.md` is the working document for executing Step 7.

**⚠️ BEFORE deploying — strip a lot of comments from the Sass library.** Per user: the library's `.scss` files have heavy doc comments (HTML-shape examples, usage notes, design rationale in `_card.scss`, `_dialog.scss`, `_footer.scss`, `_menu.scss`, `_hover.scss`, `_searchBar.scss`, etc.). Do a comment-trimming pass on `axlothecook-sass-library` — keep brief headers / param-purpose notes, cut the verbose blocks — then push the library, before/as part of the Pi deploy. (Could also just rely on the consumer's build minifying away comments in the compiled CSS, but the user wants the source trimmed too. Confirm scope with user when this comes up — they said "strip away a lot", not all; the `Mini-Messaging-Board/.../warning.css` precedent and the "removed most of comments" past commit on this repo are the style target.)

Three things run on the Pi: backend (Express) + frontend (SvelteKit) + MongoDB.

**Storage architecture (updated 2026-05-27 — migrated Supabase -> Cloudflare R2):**
- **MongoDB** stores game/genre/developer data + URLs pointing to the image files. Runs ON the Pi.
- **Cloudflare R2** serves the actual image files (S3-compatible, $0 egress), via the custom domain `images.axlothecook.com` (shared bucket `axlothecook-images`, `gameshop/<bucket>/<file>` key prefix). The Pi passes through the `R2_*` env vars. (Previously Supabase Storage — migrated off because Supabase free-tier projects pause after 7 days of no DB activity, taking images offline. The backend `storage.js` wraps the AWS S3 SDK against R2.)

**Mongo on the Pi:**
- Needs **64-bit Pi OS** — official MongoDB packages dropped 32-bit ARM long ago (so Pi 4/5 only). For older Pis: the unofficial community ARM build, or run Mongo in Docker.
- Local dev's data: if local Mongo has the games/genres/devs collections, `mongodump` locally → `mongorestore` into the Pi's Mongo. The image URLs in the docs point at `images.axlothecook.com` (Cloudflare R2) and stay valid.
- Connection string is local (`mongodb://localhost:27017/...`) — set as `MONGO_URI` in the backend's `.env` on the Pi.

**Running the apps on the Pi:**
- Backend: Node/Express, `npm start` (or under a process manager — `pm2` recommended so it survives reboots / restarts on crash).
- Frontend: SvelteKit needs `@sveltejs/adapter-node` instead of the default `adapter-auto`. Build `npm run build`, run `node build` (also under `pm2`).
- CORS: backend must allow the frontend's origin. Frontend needs its API base URL pointing at the backend (env var, not hardcoded).

**Localhost → production code changes — VERIFY these are done during / at the end of deployment.**
A scan (2026-05-17) confirmed neither repo hardcodes localhost/IPs/ports — the backend URL, Mongo URI, and port are all env-driven (frontend `LINK` via `$env/static/private`; backend `process.env.NODE_ENV_DB_LOCALHOST` / `NODE_ENV_PORT_LOCALHOST`). So the localhost→prod switch is a config change, NOT a code change. BUT three deploy-time items must still be checked off:

1. **`adapter-auto` → `adapter-node`** — `Gaming-shop-frontend/svelte.config.js` currently uses `@sveltejs/adapter-auto`, which will NOT run on the Pi. Switch to `@sveltejs/adapter-node` (build → `npm run build`, run → `node build`). VERIFY before/at deploy.
2. **Tighten CORS** — backend `app.js` currently has `app.use(cors())` = allow ALL origins (insecure for production). Change to `cors({ origin: <frontend URL from env> })`, env-driven (so it survives a future domain switch). VERIFY done + frontend can still reach the API.
3. **Frontend `LINK` is baked in at BUILD time** — it's read via `$env/static/private`, so the production `LINK` value must be present when `npm run build` runs in the CI pipeline (provide it as a GitHub Actions secret), NOT just at container start. VERIFY the CI build has the prod value. (If runtime-changeable URLs are ever wanted, switch to `$env/dynamic/private` — not needed for a fixed deploy.)

Optional (cosmetic, not a verify-item): the backend env var names `NODE_ENV_DB_LOCALHOST` / `NODE_ENV_PORT_LOCALHOST` say "LOCALHOST" but will hold prod values — they function fine; could rename to `MONGO_URI` / `PORT` for clarity.

**CI/CD to the Pi:**
- Simplest: GitHub Actions → SSH into Pi → `git pull && npm ci && pm2 restart`.
- Better: build artifacts in Actions, `scp` to the Pi, run.
- Best: Docker Compose on the Pi; Actions builds + pushes images to GHCR; Pi pulls + restarts (webhook or cron).

**Networking & TLS:**
- The Pi must be reachable: port-forward + DDNS (DuckDNS), or **Cloudflare Tunnel** (no port forwarding, free tier).
- TLS: **Caddy** is easiest — auto-provisions Let's Encrypt certs.

To research before this step:
- Whether the frontend's API base URL is hardcoded vs read from env.
- Adapter currently configured in `svelte.config.js` (needs `adapter-node`).
- Pi model / OS (must be 64-bit for native Mongo).

## DEFERRED — publish the Sass library to npm? [revisit AFTER the archery project]

User asked whether to publish `axlothecook-sass-library` as a real npm package. Decision: **not yet — revisit after the VSK-archeryClub rebuild** (the library keeps evolving as both projects consume it; publish once the component set + class names have stabilised). Arguments:
- **Not needed now:** `npm install github:axlothecook/axlothecook-sass-library` already works — no publish, no registry account, no name worries; Railway/any host fetches from GitHub fine. For a handful of own-projects that's enough. Tag releases (`git tag v1.0.0 && git push --tags`) → consumers can pin `github:axlothecook/axlothecook-sass-library#v1.0.0` for a frozen version (most of the "package" benefit without publishing).
- **API still moving:** components added/renamed every session (`_select.scss` changed again recently). Publishing now = version churn — each tweak is a `1.0.x` bump + re-publish + `npm update` in each consumer. Step 5 of this plan explicitly expects the Gaming Shop port to surface more library changes.
- **Not packaged for publish yet:** `package.json` has `"main": "index.js"` pointing at a nonexistent file, no `"files"` allowlist, no `"exports"` map, no `"sass"` field; a naive `npm publish` would ship `gulpfile.js`, lockfile, `css/` build output, etc. Publish-prep TODO when the time comes: add `"files": ["sass-library"]` (or `["sass-library","css"]`), `"exports": { ".": "./sass-library/index.scss" }`, drop the bogus `"main"`, maybe a `.npmignore`; check name availability (`npm view axlothecook-sass-library`) or go scoped (`@axlothecook/sass-library` — always publishable under your username); then `npm publish` (`--access public` if scoped) and switch consumers from `github:...` to the registry version.
- **When publishing IS worth it:** many projects installing it, want clean `npm install` + semver, want it discoverable on npmjs.com, consumers shouldn't need a GitHub-aware install.

---

# Open questions / decisions needed

## [ASK TOMORROW / next session — accumulated during the global.css migration]
- **Library: add `option:first-of-type` / `option:last-of-type` rules to `_select.scss`?** Gaming Shop's old `<option>` styling had `option:first-of-type { padding-top: .5rem; border-radius: 8px 0 0 0 }` and `option:last-of-type { border-radius: 0 0 0 8px; padding-bottom: .5rem }`. The library's `.default-select option` has neither (the old `_option.scss` that had them was deleted). For now those two rules stay live in `global.css` (flagged as a DIFFERENCE). Decision: add equivalents to `_select.scss` (`.default-select option:first/last-of-type`, maybe a `$option-edge-radius` param) so it IS covered, or leave them app-side forever? — defer until the `<select>` port.
- **Navbar `<nav>`: should it become a flex-row container (so `.nav-width-100` fits)?** Gaming Shop's `nav { width: 100% }` is JUST width — the inner `<ul>` does the flex layout. The library's `.nav-width-100` is `width:100% + min-height:fit-content + display:flex + flex-direction:row`. So `.nav-width-100` isn't a clean swap for the current `<nav>` (flagged DIFFERENCE in global.css; not swapped). Decide during the navbar port whether to restructure the navbar so `<nav>` is the flex container (then use `.nav-width-100`) or keep the `<ul>`-does-the-layout structure (then leave `nav { width: 100% }` app-side or use a utility). Note `_navbar.scss` also globally adds `nav { box-shadow: $base-box-shadow }` and `nav ul { list-style-type: none }` to ALL `<nav>`s in any consuming app — the box-shadow is new behaviour Gaming Shop didn't have; decide if that's wanted.
- **`textarea { background-color: #fff; }` in `src/lib/styles/index.scss` — NOT YET ADDED.** The commented-out `textarea {}` block in `global.css` references this override (library's `_textarea.scss` defaults the bg to `white-smoke` #F3F4F6, not #fff). Need to add that one line to `index.scss`. (Was agreed, just not done yet at end of session.)
- **`<select>` port deferred:** in `global.css` the `select {}`, `select:focus`, `option {}`, `option:hover`, `option:not(:last-of-type)`, `option::checkmark` rules are still LIVE — they need `class="default-select"` added to the `<select>` elements in `SelectEdit.svelte` & `SelectMultiple.svelte` first, THEN those rules can be commented out (`.default-select option` covers them). User said they'll do the `<select>` styling when they reach that element (not while focused on the homepage).

- **Frontend port — breakpoint strategy [ASK USER — pending].** User said "Option 2: snap everything to the library's breakpoint scale, use the mixins" — but the library's `_breakpoints.scss` mixins are **`min-width` only (mobile-first)**, while ALL of Gaming Shop's existing CSS is **`max-width` (desktop-first)**. So "Option 2" actually splits into: (2a) **mobile-first rewrite** — make the mobile layout the base in each component, add `@include breakpoints.sm/md/lg/xl { ... }` for larger screens (most "correct", most work, highest risk of subtle responsive regressions during the port); (2b) **keep desktop-first / `max-width`, just snap the pixel values to the library's scale** (480/720/960/1024/1200/1450) — written as raw `@media (max-width: ...)` (the library has no `max-*` mixins) or by adding a tiny project-level `max-*` mixin set in `src/lib/styles/index.scss` — consistent numbers, low risk, doesn't use the library mixins directly; (2c) **add a `max-width` mixin set to the library itself** (e.g. `breakpoints.md-down { ... }`) so the frontend can stay desktop-first AND use mixins (a library change — fits the "Step 5: port reveals a library need → fix the library" rule). Decision needed before porting any responsive rules. Library breakpoints recap: `xxs`(0) `xs`(480px) `sm`(720px) `md`(960px) `lg`(1024px) `xl`(1200px) `xxl`(1450px).
- Step 3 strategy — A, B, or C? Default is A.
- Step 7 — RESOLVED (2026-05-27): Mongo runs in a container ON the Pi; images on Cloudflare R2 (not Supabase). Deploy is live.
- Step 8 — Docker on Pi or bare Node? Defer.
- Publish the Sass library to npm? Deferred until after the archery project — see "DEFERRED — publish the Sass library to npm?" section above.
