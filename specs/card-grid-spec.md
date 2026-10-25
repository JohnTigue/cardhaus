# Cardhaus Card Grid

Status: Implemented; shared fluid typography added, visual tuning pending.

Oct 13, 2026 · @John

Spec for the Cardhaus card grid system: a Hugo theme built around image-heavy, magazine-style topic cards using modern CSS (grid, container queries, fluid units).

## Philosophy

Cardhaus leans on modern CSS primitives rather than ad-hoc rules or heavy frameworks. The goal: less code, because the platform baseline now does more.

- CSS Grid for macro layout (how many cards fit per row), container queries for micro layout (how one card reshapes based on its own size) — complementary layers, not competing techniques.
- Container query units (`cqi`) and `ch`, wrapped in `clamp()`, drive fluid sizing instead of fixed breakpoint jumps. Card geometry follows each card's inline size. Registered, inherited length properties compute typography from the containing card grid so cards in the same grid share a size; separate grids scale independently. This also improves behavior under browser zoom, since relative units stay proportionally sensible.
- Logical properties (`inline-size`, `block-size`, `inset-inline`, etc.) used throughout instead of physical ones (`width`, `height`, `left`/`right`), keeping the system RTL-ready by default.
- A reusable "Stack" spacing primitive (from Andy Bell and Heydon Pickering's Every Layout, popularized by Kevin Powell) for consistent internal rhythm inside the overlay and other components — the parent owns the spacing between its children via `gap`, not margins on each child. This follows the same separation-of-concerns principle as the Card Grid / Card split: a stack never knows what content it holds, just as the Card Grid never knows what's inside a card — each layer stays ignorant of whatever's nested inside it. If you're customizing Cardhaus's CSS later, this is why spacing lives on containers, not on the elements themselves.
- Everything targets current Baseline "widely available" features only — no experimental or flagged APIs.
- Light/dark mode is treated as an orthogonal, separate color-token layer — out of scope for this spec.

  Mobile-native, not desktop-shrunk-to-mobile: Cardhaus is designed thumb-first, not as a responsive site retrofitted for small screens. The hero carousel's bottom liquid-glass control bar — progress indicators plus left/right controls, positioned for thumb reach rather than a conventional top header — sets the site's interaction language. The card metaphor itself exists in service of this: swipeable, full-bleed, touch-friendly. Future components (including the card overlay) should stay aligned with this bottom-anchored, swipe-native aesthetic rather than conventional desktop-first patterns.

## Card Grid component

Shortcode: topic card group. Renders the grid wrapper only — stays deliberately "dumb," with no knowledge of card internals.

- `display: grid`
- `grid-template-columns: repeat(auto-fit, minmax(min(100%, $cardLayoutBreakpoint), 1fr))` — fluid column count, no manual viewport breakpoints. The shared breakpoint is also the card's own container-query threshold.
- `gap: 1.5rem`.
- With the 584px default and 1.5rem gap, two columns fit once the grid is at least 1192px wide (`2 × 584px + 24px`). The grid uses auto-fit to add columns as the available width allows. Cards independently query their own widths, so cards in multi-column grids can use landscape mode.
- Once the grid reaches that two-column width, each card link has a `90svh` maximum block size. Its preferred aspect ratio remains in effect below the cap; single-column grids are not capped.
- `max-inline-size` around 1800–1900px on the wrapper, centered with auto margins, so cards don't stretch absurdly wide on ultrawide monitors. Parent wrappers that currently impose a 1280px cap must allow the card grid to reach this size. Preferred over a hard column cap — keeps the system fully fluid.
- The card grid does not use the existing viewport-based `mobile` and `tablet` breakpoints for its column count. Those breakpoints remain in use for other components, including the site header and hero carousel.
- Also carries `container-type: inline-size` and a `card-grid` container name. Registered `<length>` properties are computed on each direct card from the grid's inline size, then inherited by its text, keeping typography consistent among cards in that grid. Each grid computes its own scale.
- Separate shortcode from the card itself, by design — the card component has no knowledge of the grid, and is portable to any context (sidebar, grid, elsewhere).

## Card component

Shortcode: topic card. Fully self-contained — knows nothing about the grid it sits in, reacts only to its own box size.

- `container-type: inline-size` with the name `card` in every context. Its own size controls whether the card is portrait or landscape, independently of the grid's column count. When it is a direct child of `.card-grid`, typography inherits the registered fluid sizes computed against that shared grid; a standalone card retains card-relative fluid typography.
- A full-bleed hero image fills the card edge to edge — this is the card's visual anchor, echoing the site's full-screen magazine-style hero carousel.
- `object-fit: cover` uses the authored focal-point data in both modes. Portrait mode applies a fixed upward adjustment to the focal y coordinate; landscape mode uses the authored x and y coordinates directly.
- Two aspect-ratio states, switched via `@container` query, not a media query:
  - **Portrait** (default / wider container): `aspect-ratio: 3 / 4`; the image fills the card and the frosted text overlay sits over it.
  - **Landscape layout** (narrow container): `aspect-ratio: 16 / 9`; the image takes 60% of the row and the transparent text block takes the remaining 40%.
- The shared threshold is `params.cardhaus.card.landscapeMaxInlineSize`, expressed in CSS pixels and defaulting to 584px. Hugo uses the value as the grid's minimum track width and emits the same configured value into each card's `@container` query. It is independent of viewport breakpoints.
- Fluid units (`clamp()` with `cqi` and `ch`) size card geometry, line lengths, and padding rather than the threshold itself. In a grid, titles, summaries, and labels use registered inherited lengths from the grid so their computed sizes match across its cards.

## Text overlay

A single sub-component (title + description) that sits over the full-bleed image in portrait mode and beside it in landscape mode. The title remains above the description in both states.

**Position by state**

- Portrait: bottom-anchored frosted overlay with a fixed upward adjustment to the image focal y coordinate.
- Landscape layout: transparent text column uses the remaining 40% of the card. The 60% image column uses the authored focal point directly.

**Transition behavior**

- Treated as one continuous elastic object being reshaped, not independently-animated properties: position and wrapping respond to the card's container, while the shared fluid type size responds to its grid. These changes morph smoothly as the layout resizes (e.g. on viewport resize or device rotation) — a fluid "liquid glass" stretch rather than a hard snap between two fixed states.
- `prefers-reduced-motion` respected: motion-sensitive users get a clean, immediate state change instead of the morphing animation. The interaction section also disables image zoom and card lift for these users.

**Visual style — "frosted liquid glass"**

- Glassmorphic baseline (blur + translucency, in the spirit of "liquid glass"), but tuned down from typical liquid-glass transparency for legibility — a frosted variant, not a strict liquid-glass implementation.
- Also informed by Material Design 3 Expressive principles.
- In portrait mode, a frosted scrim behind the text preserves legibility over busy or bright image regions. The landscape text column is transparent.

  Overflow handling keeps card height predictable while making room for available copy. Titles wrap naturally instead of being cut off with an ellipsis. The summary flexes into the remaining overlay height after the title and fades only at its clipped lower edge; longer titles leave less room for summary text.
- Exact backdrop treatment (gradient vs. solid panel vs. blur) refined during implementation.

## Interaction states

- Hover/focus image zoom — already implemented for hover; add the same treatment for keyboard `:focus-visible`. The zoom on the background hero image is the focus indicator — no added outline, ring, or border; stays deliberately minimal, consistent with the rest of the aesthetic. When reduced motion is requested, use a static focus-visible scrim treatment instead of zoom so focus remains visible without movement.
- Cards remain in normal document tab order: Tab advances through card links. ArrowLeft/ArrowRight navigate the hero carousel when focus is within it; ArrowUp/ArrowDown retain native page scrolling. Do not add arrow-key traversal or roving tabindex to card grids.
- Give card grids accessible list semantics and an accessible name associated with their heading. Use native list markup where practical, with ARIA list roles and `aria-labelledby` when the rendered shortcode structure requires them. Do not use `role="grid"`, which would imply a different keyboard interaction model.
- **Scrim strengthening on hover/focus** — new: backdrop behind the text gets a touch stronger on interaction, reinforcing that cards are clickable, magazine-style teasers.
- **Reduced motion** — resolved: `prefers-reduced-motion` disables card lift and image zoom; a static focus-visible scrim preserves a visible keyboard focus cue in portrait mode.

**Performance**

- Image-heavy site; responsive images and lazy loading already handled via the Hugo asset pipeline — no open concerns here. No separate loading/skeleton state is needed for the hero image either — the card's aspect-ratio box already reserves its space ahead of time, so there's no layout jump to guard against regardless of decode timing.

  Print styles: explicitly out of scope. No work has been done on print styling for Cardhaus — this is a deliberate decision, not an oversight, and should be noted as such in the README.

## Open questions

- **Portrait-to-landscape threshold** — resolved as the shared `params.cardhaus.card.landscapeMaxInlineSize` setting. It controls the grid's minimum track width and the card's own container query. Each card responds to its own width, including cards in multi-column grids.
- **Shared fluid type scale** — resolved with registered inherited lengths calculated from each `.card-grid` width. The initial curves are conservative starting values; tune them against narrow, multi-column, and maximum-width grids in the browser.
- **Landscape ratio and image/text proportions** — resolved to a 16:9 card, with a 60% image column and a 40% transparent text column.
- **Max columns at very wide viewports** — resolved: no hard column cap; rely on `max-inline-size` on the grid wrapper (~1800–1900px) plus `auto-fit`/`minmax` underneath it. Widen enclosing wrappers as needed so they do not retain the old 1280px limit.
- **Light/dark mode** — confirmed orthogonal to this spec; a separate color-token layer, not addressed here.
- **Backdrop/transparency treatment** — resolved: use Cardhaus's existing transparency technique rather than a fresh approach. Built on SVG filter functions for the frosted/blur effect, already hardened against the wavy rendering artifact seen on older (circa 2016–2019) browsers/GPUs during earlier development. Carries forward into the overlay unchanged.
- **Stack spacing primitive** — resolved: baseline on the "Stack" pattern from Andy Bell and Heydon Pickering's Every Layout (every-layout.dev), popularized further by Kevin Powell. The parent container (not the children) owns the spacing: children stay unstyled and unaware they're in a stack, while the parent injects margin/gap between them via an adjacent-sibling selector (or native `gap` in a flex column). Scoped locally to the overlay's title + description grouping — a flex-based micro-layout nested inside the grid-based card, itself nested inside the grid-based page. Does not blur the Card Grid / Card separation-of-concerns boundary: the grid still only lays out cards as whole units and has no knowledge of the stack inside a card's overlay.

> **Implementation note:** tune the `params.cardhaus.card.landscapeMaxInlineSize` setting and the portrait backdrop together in the browser. Keep this card-width threshold independent from the viewport `mobile` and `tablet` breakpoints.
