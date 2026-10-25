# Hero carousel linking update

Status: partially implemented; carousel handoff integrity and rendered-link
verification are implemented, while broader lightbox and remote-image checks
remain.

## Current implementation status

Verified:

- All current `carousel.yaml` files use the new `image`, `hash`, and optional
  `link` schema, including the sport-coat draft.
- Legacy carousel prefix configuration is removed and legacy `slug` input is
  rejected.
- Local images use Hugo processing and responsive preloads.
- Bundle-relative image paths resolve from the bundle containing
  `carousel.yaml`; site-root-relative paths retain their existing behavior.
- The homepage-to-Bandgrind session 13 handoff focuses the matching destination
  slide. A fragment-selected slide remains focused with autoplay stopped.
- The production build validates all generated files and internal links, and
  `git diff --check` passes.
- `scripts/validate-carousel-handoffs.py` validates every image-preserving
  same-site handoff and its rendered anchor/destination hash after a Hugo
  build.

Implemented but not yet verified against the acceptance criteria:

- Remote image rendering and dimension-based aspect-ratio reservation.
- Unlinked local and remote slides opening the full-image lightbox.
- Lightbox desktop/mobile sizing, close behavior, focus containment,
  background suppression, and focus restoration.
- External navigation receiving no coordinated carousel transition.
- Failure cases for missing images, invalid hashes, unresolved destinations,
  and invalid dimensions.
- Regression coverage for carousel looping, focal positioning, responsive
  variants, controls, and keyboard/manual interaction.

The repository test suite includes content and rendered integration coverage
for carousel handoffs. Do not change the broader status to fully implemented
until the remaining acceptance cases are exercised and any failures are fixed.

## Context

Carousel content is currently defined in three files:

- `content/carousel.yaml` for the root homepage.
- `content/topics/qr-codes/bandgrind/carousel.yaml` for the Bandgrind branch.
- `content/topics/atelier-padonma/sport-coat/carousel.yaml` for the sport-coat
  draft.

The former carousel schema used `slug` both to locate an image bundle and to
choose the navigation destination. The implementation now uses the separated
schema described below.

The former runtime prefix handling has been removed from both carousel
partials, Padonma and Cardhaus configuration, and current documentation.
Padonma's main-menu `pageRef = "/topics"` remains intentional navigation
configuration and is unrelated to this cleanup.

The new contract separates image selection from optional navigation.

## Settled future contract

The following contract is implemented. Legacy `slug` input produces an
actionable build error.

Each slide has:

- One required `image`. A relative path is resolved from the bundle containing
  `carousel.yaml`; a site-root-relative path is resolved from the bundle named
  by that path. Both are processed by Hugo. A fully qualified `http://` or
  `https://` URL is loaded remotely.
- One required, stable `hash`, used by Swiper Hash Navigation to identify and
  focus the slide.
- Accessible `alt` text.
- Optional `focal` coordinates for the carousel crop.
- Optional `width` and `height`, primarily for remote images so Cardhaus can
  reserve the correct aspect ratio without waiting for the image to load.
- An optional `link` that determines activation behavior.

An ordinary `link` navigates normally. A same-site link to a carousel page plus
a fragment matching a destination slide's `hash` deep-links to that focused
slide. When `link` is omitted, activating the slide opens the complete,
uncropped image in an accessible full-image lightbox instead of navigating.

A source carousel may contain all four forms:

```yaml
# Deep-link to a focused slide in another same-site carousel.
- image: /topics/qr-codes/bandgrind/bandgrind-session-13/hero-13.png
  hash: hero-13
  link: /topics/qr-codes/bandgrind/#hero-13
  alt: Weaving experiment session 13
  focal:
    x: 0.5
    "y": 0.5

# Navigate to an ordinary content page.
- image: /topics/lotus-fiber/companies/pochi/pochi-weaver.png
  hash: pochi-weaver
  link: /topics/lotus-fiber/companies/pochi/
  alt: Pochi weaver

# Open a Hugo-processed local image in the full-image lightbox.
- image: /topics/lotus-fiber/companies/loro-piana/loro_pianan_jacket_detail.jpg
  hash: loro-piana-jacket-detail
  alt: Detail of a Loro Piana lotus-fiber jacket
  focal:
    x: 0.5
    "y": 0.0

# Open a remote image in the lightbox; dimensions prevent layout shift.
- image: https://images.example.org/lotus-fiber-at-loom.jpg
  hash: remote-lotus-loom
  alt: Lotus fiber being woven at a loom
  width: 2400
  height: 1600
```

The destination carousel in the first example contains the same image and hash,
with its ordinary content-page link:

```yaml
- image: /topics/qr-codes/bandgrind/bandgrind-session-13/hero-13.png
  hash: hero-13
  alt: Weaving experiment session 13
  focal:
    x: 0.5
    "y": 0.5
```

For a same-site carousel-to-carousel handoff, Cardhaus coordinates a view
transition only between the clicked source image and the focused destination
image. Other carousel images do not participate. Ordinary external navigation,
including links to a different origin, receives no coordinated view transition.

## Lightbox behavior

> Historical note: the later carousel-unification work removed the topic
> gallery. `hero-carousel.html` now supplies the only image-sequence renderer
> and its accessible dialog is the supported lightbox implementation.

An image-only slide has no navigation destination. Activating it opens the
complete source image in a modal/lightbox using contain-style sizing. The
topic-gallery experiment discussed by this historical proposal is superseded
and must not be reused.

## Implementation scope

Remove the legacy carousel prefix option entirely from Padonma and Cardhaus configuration, both
carousel partials, and current Cardhaus documentation and specification.
Migrate the established `carousel.yaml` files to the new schema, explicit
site-relative image paths, and stable hashes.

Implementation also covers the renderer, preload helper, Cardhaus default
configuration, Padonma configuration, CSS and JavaScript for modal and hash
navigation behavior, view-transition coordination, accessibility, tests, and
build verification. The `/topics` main-menu page reference remains intact.

Before implementation, re-scan for newly added `carousel.yaml` files. At the
time this note was written, the working tree also contained an untracked draft
at `content/topics/atelier-padonma/sport-coat/carousel.yaml`; it must be
reconciled if it becomes authored content.

## Acceptance criteria

- The legacy carousel prefix option has no remaining configuration, template, README,
  or SPEC references; the intentional main-menu `/topics` reference remains.
- Migrated carousel files use one `image` field, stable `hash` values, and the
  optional `link` contract. Legacy `slug` input is either rejected with an
  actionable warning/error or supported by a documented, time-bounded
  compatibility path.
- Bundle-relative and site-root-relative images pass through Hugo's image
  pipeline. Bundle-relative paths resolve from the bundle containing
  `carousel.yaml`. Fully qualified HTTP/HTTPS image URLs remain remote and
  render without attempted Hugo processing.
- Remote images with `width` and `height` reserve the intended aspect ratio.
- An ordinary linked slide navigates to its declared destination.
- A carousel link whose fragment matches a destination hash focuses that slide.
- A same-site carousel handoff coordinates a view transition only between the
  clicked image and the focused destination image. External navigation receives
  no coordinated carousel transition.
- A slide without `link` opens the selected full image in a modal/lightbox and
  does not navigate.
- The lightbox shows the entire uncropped image with contain-style sizing at
  desktop and mobile viewport sizes.
- The lightbox closes with its visible close button and Escape. Backdrop clicks
  close it where the click is unambiguously outside the image and controls;
  clicks on the image or controls do not.
- Opening the lightbox moves focus to an appropriate modal control, contains
  focus while open, supplies an accessible name and modal semantics, suppresses
  background interaction, and restores focus to the opening slide on close.
  Keyboard activation does not interfere with carousel controls.
- Responsive preloads resolve local slide resources under the new schema
  without producing broken or irrelevant preload URLs. Remote-image loading
  follows the documented remote contract.
- Missing local images, invalid hashes, unresolved same-site destinations, and
  invalid dimensions produce clear, actionable build warnings or errors and do
  not emit broken slide markup.
- Existing carousel looping, focal positioning, alt text, responsive variants,
  controls, and autoplay/manual-interaction behavior remain correct.
- Cardhaus documentation describes the final schema only after implementation;
  no current-usage document claims the feature exists prematurely.
- `scripts/deploy.sh build`, relevant automated tests, and `git diff --check`
  succeed after implementation.
