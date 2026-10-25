# Responsive Topic Card — Layout Spec

Oct 16, 2026 · @John

Defines how individual topic cards switch between a portrait overlay layout and a landscape side-by-side layout, driven by CSS container queries on the card's own container width.

## Grid & Layout System

- Outer topic card grid: CSS Grid, arranging multiple cards across the page.
- Individual card: internal layout handled by Flexbox, active only in landscape mode.
- Responsiveness driven by CSS Container Queries on the card's own container width, not the viewport.
- The grid uses the configurable `landscapeMaxInlineSize` value as its minimum track width, while each card uses that same value as its portrait-to-landscape container-query threshold. Column count comes from CSS Grid; layout comes from each card's own width, including cards in multi-column grids.
- When the named card-grid container is wide enough for two minimum-width tracks plus the grid gap, cap each card link at 90% of the small viewport height. The link keeps filling the card's inline size when this cap changes its preferred aspect ratio; single-column grids have no height cap.

## Portrait Mode (default)

- Card is `position: relative`.
- Image: `position: absolute`, `inset: 0` — full-bleed, fills the entire card.
- Text overlay: `position: absolute`, layered on top of the image, title above description.
- Text overlay background: semi-transparent scrim so text stays legible against the image.
- Image and text overlay are siblings in the markup, not nested — this is what makes the landscape switch possible.

## Landscape Mode (container query at narrow width)

- Card switches to `display: flex`, `flex-direction: row`.
- Image and text overlay both switch from `position: absolute` to `position: static`, re-entering normal flex flow as siblings.
- Image becomes a flex item at roughly 60% width, fully visible, no longer covered by text.
- Text block becomes a second flex item alongside the image, taking the remaining width.
- Text overlay background becomes fully transparent — page background shows through, so the text appears to float beside the image rather than sit on a panel.
- Chosen approach: keep absolute positioning for the portrait overlay (a true layered effect) and flip to static/flex only for landscape (a true row), rather than forcing both states into one flex metaphor.

## Focal Point

- Single fixed focal point coordinate (x, y) per image; the same value is used in both portrait and landscape — no separate landscape-specific value.
- Portrait: `object-position` derived from the focal point, compensated upward to account for the text overlay obscuring the lower part of the image.
- Portrait compensation is a fixed 10 percentage-point upward adjustment to the focal y coordinate.
- Landscape: `object-position` uses the focal point directly, with no compensation, since the image is no longer covered by the text overlay.

## Text Block

- Structure: title stacked above description, in both portrait and landscape modes.
- Description truncates (line-clamp / ellipsis) when it exceeds its allocated space, rather than growing the card or stretching the image.

## Browser Support & Fallback

- Container query support: Chrome/Edge 105+, Safari 16+, Firefox 110+ (roughly 2023 onward).
- Fallback for unsupported browsers: approximate the switch with a viewport-based media query instead of a true container query — less precise, but an acceptable baseline.
