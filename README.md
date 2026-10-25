# Cardhaus

Cardhaus is an image-first Hugo theme for topic-centered knowledge sites. Its
stable rendering contracts are defined in `specs/SPEC.md`; this file covers
current installation, configuration, and authoring usage. The roadmap for
unfinished theme work is `specs/ROADMAP.md`.

## Install

For local development, place this directory at `themes/cardhaus` and select it
in the consumer's Hugo configuration:

```toml
theme = "cardhaus"
```

The directory is also a self-contained Hugo Module. After Cardhaus is extracted
and published at the module path declared in `go.mod`, a consumer can install it
from its own module-enabled site:

```sh
hugo mod init example.com/my-site
hugo mod get github.com/Padonma/cardhaus
```

```toml
[module]
  [[module.imports]]
    path = "github.com/Padonma/cardhaus"
```

The import path is extraction scaffolding and will resolve only after that
standalone repository is published. The embedded Padonma consumer continues to
use the local `theme = "cardhaus"` setting.

## Configure the consumer

All identity parameters are optional unless the chosen consumer layout depends
on them:

```toml
[params]
  description = "Site description"
  subtitle = "Site subtitle"
  logo = "brand/wordmark.png"
  billboardImage = "brand/background.jpg"
  billboardLogo = "brand/mark.png"
  billboardTitle = ["Site", "Name"]
  topicsHeading = "Topics"
  footerImage = "brand/footer.jpg"

  [params.cardhaus]
    customStylesheet = "css/brand.css"
    # themeColor = "darkgreen"

    [params.cardhaus.breakpoints]
      tablet = 1200
      mobile = 850

    [params.cardhaus.card]
      landscapeMaxInlineSize = 584

```

Image paths resolve through the consumer's global `assets/` directory. The
custom stylesheet is processed and fingerprinted after the theme bundle; use it
to override semantic CSS properties such as `--accent`, `--primary`, and
`--font-display`. Breakpoints are unitless pixel values compiled into media
queries. `landscapeMaxInlineSize` is the shared card-width threshold in CSS
pixels. The grid uses it as its minimum track width, and each card uses the
same value in its own container query. At the 584px default, the grid adds a
second column when it has room for two 584px tracks plus the 1.5rem gap; a card
switches to landscape when its own width is 584px or less. Column count and
card layout stay independent while sharing one configurable value.

### Favicons and installable app icons

Favicon support is optional and configured under `[params.cardhaus.favicons]`:

```toml
[params.cardhaus.favicons]
  source = "brand/site-icon.png"
  ico = "brand/site-icon.ico"
  # name = "Site name for installation"
  # shortName = "Short name"
  # startURL = "/"
  # scope = "/"
  # display = "standalone"
```

`themeColor` is configured under `[params.cardhaus]`. When set, Cardhaus uses
the value for both the page's `theme-color` metadata and the manifest's
`theme_color` member. If unset, Hugo logs a warning and omits both. It does not
set the manifest's `background_color` member.

`source` is optional and names a square raster image in the consumer's global
`assets/` directory. When set, Cardhaus uses Hugo's image pipeline to generate
16, 32, and 48 pixel browser icons, a 180 pixel Apple touch icon, and 192 and
512 pixel app icons. It also creates padded maskable variants of the app icons
and publishes a `site.webmanifest` with both regular and maskable icons. Keep
the important artwork centered within the maskable safe area. The manifest
defaults its name to the site title, its launch URL and scope to the site root,
and its display mode to `standalone`.

`ico` is optional and names an ICO file beneath the consumer's `static/`
directory; Cardhaus emits a link to it. If no `source` is set, Cardhaus does
not generate PNG variants or a web manifest. These settings provide the icons
and manifest metadata for installation; offline behavior requires a separate
service worker.

### Glass rendering compatibility

Cardhaus treats its SVG glass reflection as a progressive enhancement. Every
browser receives the translucent tint, rim, and ordinary backdrop blur. The SVG
reflection is enabled only when the browser supports both an SVG backdrop
filter and `color-mix()`.

The `color-mix()` check is deliberately a conservative modern-engine floor, not
a dependency of the visual effect itself. Some older Chromium-derived browsers,
including older Amazon Silk releases, accept SVG backdrop-filter syntax but
incorrectly apply its displacement to the entire composited card. That can make
the image, overlay, border, and text appear wavy. Syntax detection for the
filter alone therefore produces a false positive on those engines. Requiring a
newer broadly supported CSS feature leaves them on the stable ordinary-blur
fallback without user-agent or device detection, while current mobile browsers
remain eligible for the enhanced reflection. Keep this compatibility gate with
the SVG filter rules in `assets/css/components.css` unless the rendering issue
is replaced by a reliable direct capability test.

## Author a topic

Topics are normally page bundles below `content/topics/`. Select a canonical
bundle image and crop in front matter:

```yaml
hero:
  image: "filename.jpg"
  focal:
    x: 0.5
    "y": 0.35
  alt: "Description of the image"
topic_type: "Optional classification"
```

Coordinates range from `0.0` to `1.0`. Quote `"y"` because YAML 1.1 may
otherwise interpret it as a boolean. If `hero.image` is omitted, Cardhaus uses
the first compatible bundle image when available. The Hugo summary supplies
optional card copy.

Supporting images may use entries in the page's `images` front matter array for
focal data. The leaf-carousel generator preserves those values: per-image
coordinates take precedence, and missing coordinates on the canonical image
inherit from `hero.focal`.

## Configure social previews

The canonical hero is the default page social image. Add a `social` block only
when the preview should differ:

```yaml
social:
  image: "social-preview.jpg"
  alt: "Description for the social preview"
  focal:
    x: 0.55
    "y": 0.25
```

Each property is optional. Hero alt and focal data are inherited only when the
selected social resource is the hero. A distinct image defaults to the title
and a centered crop. Explicit unresolved social images fail the build.

Consumers may provide a global fallback:

```toml
[params.cardhaus.social]
  image = "images/social-default.jpg"
  alt = "Site name"

  [params.cardhaus.social.focal]
    x = 0.5
    y = 0.5
```

## Embed topic cards

Embed one existing canonical card with an absolute content reference:

```go-html-template
{{</* topic-card ref="/topics/example-topic" */>}}
```

Override the referenced page's card image, title, or description when needed:

```go-html-template
{{</* topic-card
  ref="/topics/example-topic"
  image="alternate-card-image.jpg"
  title="Alternate card title"
  description="A short **Markdown-enabled** description."
*/>}}
```

`ref` remains the internal page destination. Override images are resolved from
that page's bundle when `image` is relative. An `image` beginning with `/` may
identify a resource in any page bundle (for example,
`/topics/example-topic/alternate-card-image.jpg`) or a file beneath the site's
`assets` directory (for example, `/images/alternate-card-image.jpg` resolves
`assets/images/alternate-card-image.jpg`). This lets a card use another internal
image without changing the referenced page's hero. Every override is optional;
omitted values continue to use the referenced page's canonical card data.

Group curated cards with an optional heading, custom CSS class (or
space-separated classes), and introductory Markdown:

```go-html-template
{{</* topic-card-grid heading="Related topics" class="related-topics featured-grid" */>}}

These topics provide useful context.

{{</* topic-card ref="/topics/example-topic" */>}}
{{</* topic-card ref="/topics/another-topic" */>}}

{{</* /topic-card-grid */>}}
```

A grid requires a nested card. Its wrapper always has `topic-card-grid` (and
`topic-card-grid--home` on the homepage); `class` appends the supplied classes.
Put all introductory Markdown before its first card. Missing or relative
references fail the build. Do not repeat one topic on a rendered page because
canonical cards carry document-unique transition names.

Card titles, summaries, and labels in a grid share fluid sizes computed from
that grid's inline size. Separate grids scale independently. Each card still
uses its own container size for overlay spacing and layout. Each card
independently switches at the shared breakpoint, including cards in
multi-column grids. Landscape cards use a 16:9 side-by-side layout with a 60%
image and transparent 40% text column. In multi-column grids, card height is
capped at 90% of the small viewport height. Long titles wrap, and summaries
use the remaining card height before fading at the clipped lower edge.
The typed, inherited CSS properties that provide this shared scale follow the
technique in
[Fixing fluid typography, by Kevin Powell](https://www.youtube.com/watch?v=q-_cIlttYBc).

## List immediate children

In a branch bundle, list immediate authored children using the branch title:

```go-html-template
{{</* branch-children */>}}
```

Or select a branch and heading explicitly:

```go-html-template
{{</* branch-children ref="/topics" heading="Introduction" */>}}
```

The reference must resolve to a branch bundle. Children use `.Pages.ByWeight`;
deeper descendants and loose Markdown files are excluded. Empty branches render
nothing. Hugo's draft, future, expiry, `build.list`, and `build.render` behavior
still applies. `render: link` requires the destination to be supplied elsewhere.

For a page that contains this listing, set an explicit front-matter `summary`
when its own card needs copy. Cardhaus omits automatic summaries for such pages
to avoid recursively rendering the listing inside the card.

## Add a page-bundle carousel

Place `carousel.yaml` beside a branch bundle's `_index.md` or a leaf bundle's
`index.md`. The homepage uses `content/carousel.yaml`. A bundle without that
file has no carousel: sibling images are never discovered implicitly. Homepage,
branch landing pages, and leaf pages with a top-of-page carousel receive
opening scroll snap.

```yaml
- image: supporting-image.jpg
  hash: example-topic-supporting
  link: /knowledge/example-topic/
  alt: Description of the selected image
  focal:
    x: 0.4
    "y": 0.6
```

`image`, `hash`, and accessible `alt` text are required. A bundle-relative
`image` path such as `supporting-image.jpg` selects a resource beside
`carousel.yaml`. A site-root-relative path such as
`/knowledge/example-topic/supporting-image.jpg` selects a resource from that
page bundle. Both local forms use Hugo's image pipeline; a fully qualified
HTTP(S) URL remains remote. Remote slides may declare positive
`width` and `height` values to reserve their aspect ratio. `focal` controls the
carousel crop.

For an existing image-bearing leaf bundle, generate deterministic entries from
its title, hero, and per-image focal metadata:

```sh
python3 scripts/generate-carousel.py content/topics/example-topic
python3 scripts/generate-carousel.py content/topics/example-topic --check
```

Run the program without a directory from inside a copied bundle. Use
`--dry-run` to print the proposed file. Generation refuses to overwrite an
existing carousel, excludes AVIF, emits no links, and gives every generated
slide a lightbox action. See `--help` for all modes.

`link` is optional. An ordinary external link, or a fragmentless link into a
carousel that does not share the source image, navigates normally. A same-site
image-preserving link supports either a carousel or canonical-hero destination.
A slide without `link` opens its complete, uncropped image in an accessible
lightbox. Legacy `slug` input is rejected with a build error.

To link one carousel to the same image in another carousel, give both entries
the identical `image` and `hash`. On the source entry, set `link` to the
destination carousel page followed by `#` and that hash:

```yaml
# Source carousel
- image: /knowledge/weaving/session-13/hero.png
  hash: weaving-session-13
  link: /knowledge/weaving/#weaving-session-13
  alt: Weaving experiment session 13

# Destination carousel at /knowledge/weaving/
- image: /knowledge/weaving/session-13/hero.png
  hash: weaving-session-13
  link: /knowledge/weaving/session-13/
  alt: Weaving experiment session 13
```

The destination entry's `link` remains its ordinary activation destination;
it does not point back to the source carousel. Loading the destination URL with
the fragment focuses the matching slide and stops autoplay so it remains
selected.

To hand off to a topic bundle without `carousel.yaml`, use that bundle's
canonical hero as the source slide image and link to the fragmentless page URL:

```yaml
- image: /knowledge/weaving/session-13/hero.png
  hash: weaving-session-13
  link: /knowledge/weaving/session-13/
  alt: Weaving experiment session 13
```

The destination resolves its hero through the normal `hero.image`/implicit-image
contract and renders it as the detail hero. The source slide adopts the hero's
stable transition name when clicked. A fragment is reserved for selecting a
destination carousel slide, so carousel-to-hero links must not include one.
The build rejects a same-site carousel link when the destination has neither
`carousel.yaml` nor a valid hero, or when the source slide is not that hero.

The focused carousel slide is capped at `70vw` by default, leaving about
`15vw` of each neighboring slide visible for capped landscape images. At the
configured mobile breakpoint (850px by default), that cap becomes `100vw`,
leaving no neighboring-slide peek for capped landscape images. The
`--hero-slide-max-width` custom property
controls both caps. Narrow portrait images retain their aspect-ratio-derived
width until they reach the cap, so they can show more of their neighbors.

## Source map and validation

Key reusable source:

- `layouts/_partials/hero-resource.html` resolves canonical images.
- `layouts/_partials/hero-image.html` renders responsive canonical imagery.
- `layouts/_partials/topic-card.html` renders every shared card.
- `layouts/_partials/hero-carousel.html` is the single carousel and image-sequence renderer.
- `layouts/_partials/favicons.html` emits configured browser icons and the optional app manifest.
- `layouts/_partials/social-metadata.html` emits sharing metadata.
- `layouts/shortcodes/` contains the author-facing card APIs.
- `assets/css/` contains tokens, base rules, layouts, and components.

Cardhaus does not define print-specific styles. Print rendering is out of scope.

From the Padonma consumer repository, run its production build and validation:

```sh
scripts/deploy.sh build
git diff --check
```
