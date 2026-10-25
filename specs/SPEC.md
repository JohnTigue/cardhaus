# Cardhaus specification

## Purpose and boundary

Cardhaus is a reusable, image-first Hugo theme for topic-centered knowledge
sites. A topic is a durable reference page, not a product. The theme owns the
rendering contracts, components, design tokens, responsive behavior, navigation
chrome, metadata, and progressive enhancement shared by its consumers.

A consuming site owns its content hierarchy, vocabulary, editorial policy,
navigation entries, brand assets, deployment, URLs, redirects, and the choice of
which Cardhaus features to use. Cardhaus must not hard-code a consumer name,
subject domain, asset path, commerce state, inventory, or purchasing behavior.

Reusable source lives under `themes/cardhaus/`. Consumer content, configuration,
and assets live outside the theme.

## Canonical topic image

A page may select one canonical page-bundle image:

```yaml
hero:
  image: "filename.jpg"
  focal:
    x: 0.5
    "y": 0.35
  alt: "Accessible description"
```

`hero.image` is the durable editorial identity. Each focal coordinate is
optional, ranges from 0 through 1, and defaults independently to `0.5`.
`hero.alt` defaults to the page title. Quote `"y"` in YAML 1.1 front matter.

When `hero.image` is absent, the first compatible bundle image may be used.
A topic bundle without `carousel.yaml` renders that canonical image as its
detail hero. A page without an image remains valid and receives an intentional
placeholder where an image-bearing component such as a card is requested.
Explicit image names that cannot be resolved should fail validation rather than
quietly choose another image; this stricter behavior remains roadmap work.

`layouts/_partials/hero-resource.html` is the canonical resolver.
`hero-image.html` renders responsive derivatives, focal cropping, accessible
text, and stable topic identity. Gallery code uses the same resolver and places
the canonical resource first. Do not create a second card or detail-image
contract.

## Cards, grids, and child listings

`layouts/_partials/topic-card.html` is the only topic-list card. It contains the
canonical image or missing-image state and a restrained dark glass overlay with
the title, optional Hugo summary, and optional `topic_type`. The image remains
primary.

Cardhaus provides three author-facing shortcodes:

- `topic-card` embeds one card resolved from an absolute content reference.
- Paired `topic-card-grid` groups curated cards and may add a heading and
  introductory Markdown. Introductory content must precede all nested cards.
- `branch-children` lists a branch's immediate authored children in weight
  order. It defaults to the current branch and may accept an absolute branch
  reference and heading.

Unresolved references and structurally invalid shortcode use fail the build.
Shortcodes delegate to shared partials. `topic-card` may override the referenced
topic's image, title, or description. A relative image override resolves from
the referenced topic's bundle; an absolute image override resolves from the
bundle identified by the image's directory, then from the site's `assets`
directory. Omitted values retain the canonical page data, and `ref` remains the
internal destination. A canonical topic should appear only once per document,
because named View Transitions must be unique.

Generated section, taxonomy, and term lists use the same card. The structural
grid has three columns above the tablet breakpoint, two through tablet widths,
and one at mobile widths. Consumers may override `tablet` and `mobile` under
`params.cardhaus.breakpoints`; defaults are `1200px` and `850px`.

## Carousels

The homepage uses `content/carousel.yaml`. A branch or leaf page reads only the
`carousel.yaml` resource in its own bundle. Bundles without that file have no
carousel, even when they contain images. A configured carousel precedes the
page's normal article presentation. Opening scroll snap applies to the
homepage and to branch or leaf pages whose carousel appears at the top.

Every slide requires an explicit `image`, stable `hash`, and accessible `alt`.
A relative image path resolves from the bundle containing `carousel.yaml`; a
site-root-relative image path resolves from the page bundle named by that path.
Both local forms use Hugo's responsive image pipeline. Fully qualified HTTP(S)
images remain remote and may supply `width` and `height` to reserve their aspect
ratio. Slide-specific `focal.x` and `focal."y"` coordinates independently
default to `0.5`. Slides contain no visible title or caption.

An optional `link` navigates normally. Image-preserving same-site links support
two handoff forms:

- A carousel destination requires a fragment matching its destination slide
  hash. The source and destination entries use the same image and hash, and the
  clicked and selected images use the fixed `carousel-handoff` transition name.
- A destination bundle without `carousel.yaml` requires a fragmentless URL and
  a canonical hero that is the same local image as the source slide. The clicked
  slide adopts the hero's normal stable transition name; the detail hero keeps
  the same name it already shares with canonical cards.

A same-site carousel link into a bundle with neither a carousel nor a canonical
hero is a build error. Slides without `link` open the complete image in a modal
lightbox. Legacy `slug` input is a build error.

Carousel navigation uses complete clone sets before and after the originals so
centered, variable-width layouts do not expose an empty track. Previous and next
controls make one physical move; after entering a clone set, the implementation
resets without animation to the matching original. Logical state comes from
`data-carousel-index`. Manual pointer or keyboard navigation stops autoplay,
as does arriving with a fragment that selects a slide; a deep-linked image must
remain focused rather than advancing on the autoplay timer.
Cardhaus supplies control labels and arrow presentation rather than Swiper's
navigation or accessibility rewriting. The URL fragment tracks the focused
slide and initializes a matching slide on arrival.

Generated leaf carousel data explicitly lists every compatible non-AVIF sibling
image in the former gallery order, with the canonical image first. Its focal
coordinates preserve per-image metadata over canonical `hero.focal` over the
center default, coordinate by coordinate. Generated alt text is always the page
title and generated entries contain no `link`, so activation opens the complete
image in the shared carousel lightbox. Runtime rendering never infers carousel
membership from sibling images. Text-only topics and image-bearing topics with
no `carousel.yaml` remain valid.

## View Transitions

Canonical cards and detail heroes receive stable, filename-safe
`view-transition-name` values derived by
`layouts/_partials/view-transition-name.html`. A carousel-to-hero source adopts
that destination name when activated. Carousel-to-carousel handoffs retain the
fixed `carousel-handoff` name on both sides. Cardhaus opts into cross-document
transitions. The named image expands over 750ms while page roots fade.
`prefers-reduced-motion: reduce` makes the named transition effectively
instantaneous and disables root fades.

## Descriptions and social metadata

Cardhaus emits a canonical URL, description, Open Graph data, Twitter card data,
and RSS discovery where that output exists. Description precedence is page
description, explicit summary, generated summary, then site description.

Social image precedence is `social.image`, canonical hero, site social image,
then none. Focal coordinates and alt text inherit only from metadata belonging
to the selected resource; otherwise they fall back to center and page title.
Processable images use the largest in-bounds 40:21 crop around the focal point
and are reduced, never enlarged, toward 1200 by 630 pixels. Other formats are
linked without an attempted crop.

## Design and extension contracts

Image overlays share a subtle dark glass treatment with useful contrast. Topic
cards use a consistent `3 / 4` portrait crop; carousel images preserve their
source proportions.

Theme CSS is divided into neutral tokens, document base, layout, and components,
then concatenated, minified, and fingerprinted. Consumers may override semantic
custom properties through a site-owned stylesheet configured as
`params.cardhaus.customStylesheet`. Structural breakpoint values are build-time
Hugo parameters because CSS custom properties cannot define media-query limits.

The theme may provide generic header, menu, breadcrumb, footer, page, section,
taxonomy, topic, carousel, and card rendering. Consumer configuration
provides navigation and identity. Header markup must be valid without any
consumer asset; the current exception is tracked in `ROADMAP.md`.

## Conformance

A conforming change preserves canonical image identity, shared cards, graceful
text-only pages, responsive grids, accessible metadata, reduced motion, exact
carousel targeting, and continuous carousel navigation. Validate with a
production Hugo build, generated-page inspection, interaction checks for changed
UI behavior, and `git diff --check`.
