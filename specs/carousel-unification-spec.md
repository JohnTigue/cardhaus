# Carousel unification specification

Status: implementation-ready proposal  
Scope: Cardhaus theme, Padonma content migration, and a portable migration utility  
Out of scope for this document change: implementation, generated content, deployment, and visual redesign

## Summary

Cardhaus must have one carousel renderer and one carousel behavior contract. The
existing `hero-carousel.html` implementation becomes that renderer for the
homepage, branch bundles, and leaf page bundles. The separate topic gallery is
removed.

A page displays a carousel only when its own bundle contains `carousel.yaml`.
Sibling images alone must never cause a carousel to appear. This makes image
selection and order explicit editorial data and removes the leaf layout's
implicit resource discovery.

Opening scroll-snap behavior applies to the homepage and to branch and leaf
pages with a carousel at the top. A leaf carousel snaps from the carousel to
the page's normal article presentation.

## Goals

- Replace the leaf-only topic gallery with the established hero carousel.
- Make `carousel.yaml` the sole opt-in and ordering mechanism at every page
  level.
- Preserve the former leaf galleries' image coverage, order, effective focal
  coordinates, title-based alt fallback, and lightbox activation during the
  migration.
- Preserve the existing homepage, Bandgrind, and Sport Coat carousels without
  rewriting their hand-authored data.
- Supply a deterministic, safe, portable Python utility for creating a leaf
  bundle's `carousel.yaml`.
- Remove code and documentation that exist only for the obsolete gallery.

## Non-goals

- Do not redesign the hero carousel.
- Do not alter its link, deep-link, autoplay, looping, responsive image,
  lightbox, or view-transition behavior except where needed to support leaf
  bundles.
- Do not infer carousel membership after migration.
- Do not add `link` to generated leaf entries.
- Add opening scroll snap around a top-of-page carousel on leaf pages.
- Do not migrate AVIF files into leaf carousels.
- Do not change unrelated page content, image files, or front matter.

## Current-state findings

The repository snapshot inspected for this specification contains:

- 47 leaf bundles (`index.md`).
- 44 leaf bundles with at least one sibling image recognized by the inventory
  scan.
- 166 non-AVIF sibling images in those image-bearing leaf bundles.
- One sibling AVIF, which the current topic gallery explicitly excludes.
- Three existing hand-authored carousel files:
  `content/carousel.yaml`,
  `content/topics/qr-codes/bandgrind/carousel.yaml`, and
  `content/topics/atelier-padonma/sport-coat/carousel.yaml`.

These counts are a planning baseline, not constants to encode. The migration
must rescan the working tree immediately before generation and report the
actual counts.

The current topic gallery's effective rules are:

1. Start with `.Resources.ByType "image"` and exclude resources whose media
   subtype is `avif`.
2. Resolve the canonical hero through `hero-resource.html`. An explicit
   `hero.image` wins; otherwise the first compatible image is the canonical
   image.
3. Put the canonical image first and retain Hugo's resource order for all
   supporting images.
4. Default every image's focal point to `(0.5, 0.5)`.
5. For the canonical image, inherit `hero.focal` coordinate by coordinate.
6. Apply matching `images[].focal` metadata coordinate by coordinate after the
   hero fallback, so per-image metadata wins.
7. Default alt text to the page title. Only the canonical image may replace
   that default with `hero.alt` in the old gallery.

The agreed migration intentionally standardizes generated `alt` to the page
title for every entry, including the canonical image. If any page currently
has `hero.alt` different from its title, the migration report must call out the
textual change for editorial review.

### Known hash blocker in the inspected snapshot

Hash derivation removes only the final extension, then replaces both
underscores and any remaining periods with hyphens. This produces valid hashes
for the inspected filenames containing multiple periods without requiring
source-file renames. For example,
`no_philanthropy.gemini_generated.png` becomes
`no-philanthropy-gemini-generated`, and `lotus_silk_economics.1.png` becomes
`lotus-silk-economics-1`.


## Unified carousel contract

### Opt-in and ownership

- The root homepage reads `content/carousel.yaml` as it does today.
- A branch or leaf page reads only the `carousel.yaml` resource in its own page
  bundle.
- No inherited, ancestor, or sibling-bundle carousel applies.
- A bundle without `carousel.yaml` renders no carousel even when it contains
  images.
- A bundle with `carousel.yaml` invokes `hero-carousel.html` before the normal
  article presentation.

### Entry schema

The established schema remains authoritative:

```yaml
- image: example_photo.jpg
  hash: example-photo
  alt: Example page title
  focal:
    x: 0.5
    "y": 0.5
```

- `image`, `hash`, and non-empty `alt` are required.
- `image` may retain all currently supported carousel forms. Generated leaf
  entries use only a bare sibling filename—never `./name.jpg`, an absolute
  path, or a page-relative URL.
- `hash` must remain unique within the carousel and satisfy the existing hash
  validator.
- `focal` remains optional for hand-authored files. Generated files always
  emit both coordinates.
- `link` remains optional in the general contract. When present, activation
  navigates. When absent, activation opens the full image in the existing
  hero-carousel lightbox.
- Generated leaf entries must never contain `link`; old leaf galleries did not
  navigate on activation.
- The removed legacy `slug` field remains an error.

### Leaf layout and presentation

Both the project override at `layouts/topics/single.html` and the theme's
`layouts/topics/single.html` must follow the same contract. If both continue to
exist, keep them behaviorally aligned.

The leaf page order is:

1. optional `hero-carousel.html`, when the bundle has `carousel.yaml`;
2. breadcrumbs;
3. page header and date;
4. article content;
5. any project-specific trailing navigation, such as Bandgrind session
   navigation in the project override.

Remove gallery-derived wrapper classes and the `topic-page--with-gallery`
condition unless a retained shared style demonstrably needs them. Carousel
presence must be tested with `.Resources.GetMatch "carousel.yaml"`, not with
`.Resources.ByType "image"`.

### Assets, preloads, and page classes

- Generalize carousel asset/preload detection only as far as required for a
  leaf bundle's local `carousel.yaml`.
- `hero-carousel-preloads.html` must work for home, section, and regular page
  bundle contexts.
- Carousel JavaScript must initialize every rendered `.hero-carousel` without
  depending on `.IsSection`.
- The root `<html>` classes that enable opening scroll snap cover the homepage,
  a branch landing page with a carousel, and a leaf page with a carousel.
- A leaf page with a carousel receives its own leaf opening-snap class. Its
  carousel and article container are the two opening snap targets.

## Python generator contract

### Placement and portability

Add a single Python 3 command-line program in the repository's utilities area
(recommended: `scripts/generate-carousel.py`). It must use only the Python
standard library unless the project explicitly adopts and documents a small
runtime dependency. It must be usable after copying the file into a page
bundle as well as from its repository location.

The default target is the current working directory. Provide an explicit
directory argument, for example:

```text
python3 scripts/generate-carousel.py [DIRECTORY] [--dry-run | --check]
```

The exact option spelling may vary, but default, explicit-directory, dry-run,
and check workflows are required and must be documented in `--help`.

### Input validation

For a target directory, the utility must:

- require a regular `index.md` file, because this utility targets leaf page
  bundles;
- refuse an existing `carousel.yaml` by default, without modifying it;
- enumerate only regular sibling files, never recurse;
- recognize the same processable raster image formats used by the former
  gallery/Hugo pipeline, case-insensitively;
- explicitly exclude AVIF;
- ignore non-image files and temporary/generated derivative files;
- fail if no supported images remain;
- parse the YAML front matter needed for `title`, `hero.image`, `hero.alt`,
  `hero.focal`, and `images[].file`/`images[].focal`, including quoted and
  unquoted `y` keys as encountered in the repository;
- report malformed or ambiguous metadata rather than silently falling back.

Because the old renderer delegates type detection to Hugo, implementation must
confirm the extension allow-list against the repository's Hugo version. The
current content inventory uses `.jpg`, `.jpeg`, `.png`, and `.webp` plus one
excluded `.avif`; support for other Hugo-processable raster types should be
tested rather than guessed. SVG must not be admitted unless the existing
responsive image pipeline is first proven to process it safely.

### Ordering

Output order must reproduce the old gallery, not the host filesystem's
directory enumeration:

1. Reproduce Hugo page-resource order deterministically for compatible sibling
   images. Confirm this with a temporary diagnostic template or rendered
   fixture against the repository's pinned/current Hugo version.
2. Resolve the canonical image exactly like `hero-resource.html`: explicit
   `hero.image`, otherwise the first compatible non-AVIF image.
3. Emit the canonical image first, followed by every other compatible image in
   its original Hugo resource order.

If Python sorting cannot be proven equivalent for all current filenames, the
migration must compare generator output with a Hugo-produced manifest before
writing content files.

### Field derivation

For each image:

- `image`: the filename exactly as stored beside `index.md`.
- `hash`: remove the final extension only, then replace every underscore and
  every remaining period with a hyphen. Perform no other normalization.
- `alt`: the page `title` as a string for every entry.
- `focal.x` and quoted `focal."y"`: effective old-gallery coordinates.
- `link`: never emitted.

Focal derivation is coordinate-wise:

1. Start at `x: 0.5`, `"y": 0.5`.
2. If the image is canonical, overlay present coordinates from `hero.focal`.
3. If a matching `images[]` entry exists, overlay its present coordinates.
4. Validate both final values as numeric and within `[0, 1]`.

Always serialize the vertical key as `"y"` to avoid YAML 1.1 interpreting an
unquoted `y` as boolean true. Stable output should use this shape and order:

```yaml
- image: filename.jpg
  hash: filename
  alt: Page title
  focal:
    x: 0.5
    "y": 0.5
```

Use a YAML-safe scalar representation for filenames, hashes, and titles. The
same input must produce byte-for-byte identical output.

### Errors and collisions

Before writing anything, validate the complete proposed document. A nonzero
exit with no output file is required for:

- an existing `carousel.yaml` in normal generation mode;
- a missing or invalid `index.md`/front matter/title;
- no supported images;
- a missing explicit `hero.image`;
- invalid or out-of-range focal metadata;
- an empty or invalid derived hash;
- two or more files deriving the same hash;
- an unsupported situation that would make ordering or metadata preservation
  uncertain.

Errors must name the target directory, offending file or field, derived value,
and corrective action. Never repair identifiers silently.

### Modes and exit behavior

- Normal mode validates, then creates `carousel.yaml` atomically. It never
  overwrites an existing file.
- `--dry-run` performs full validation and prints the exact proposed YAML to
  standard output without writing. Diagnostics go to standard error.
- `--check` validates an existing `carousel.yaml` against what the utility
  would generate and writes nothing. Exit zero only for an exact semantic
  match; report missing, extra, reordered, or changed fields clearly.
- Mutually incompatible modes produce a usage error.
- Successful normal mode reports the target and entry count.

An optional explicit overwrite mode is not required. If implemented later, it
must be opt-in and atomic; it is not used for this migration.

### Generator tests

Use temporary directories and standard-library test tooling unless the project
adopts another test runner. Cover at least:

- current-directory and explicit-directory operation;
- one image and multiple images;
- canonical-first ordering with explicit and implicit hero selection;
- uppercase extensions and deterministic ordering;
- AVIF exclusion;
- page-title quoting and Unicode;
- default focal coordinates;
- coordinate-wise hero focal inheritance;
- coordinate-wise per-image focal precedence;
- quoted and YAML-1.1-style unquoted `y` input;
- invalid focal values;
- final-extension-only hash derivation;
- underscore and remaining-period replacement, including filenames with
  multiple periods;
- invalid hashes caused by characters other than underscores or periods;
- derived-hash collision across different extensions;
- refusal to overwrite;
- dry-run output and check-mode pass/fail;
- absence of `link` in every generated entry;
- deterministic byte-for-byte output.

## Migration procedure

1. Record a clean/dirty working-tree baseline and preserve unrelated changes.
2. Add and test a temporary Hugo diagnostic that outputs, for every leaf
   bundle, the former gallery's ordered filenames, canonical image, effective
   focal coordinates, and alt text. Save the result outside committed output
   or as a deliberate test fixture.
3. Rescan all `index.md` bundles and all existing `carousel.yaml` files. Do not
   assume the 47/44/3 planning counts still hold.
4. Run the generator in dry-run mode for every image-bearing leaf without a
   carousel. Aggregate failures before writing any files.
5. Resolve invalid hashes and collisions editorially. Do not allow
   automatic suffixes or transformations beyond the specified
   underscore/period replacement.
6. Generate one new `carousel.yaml` for every eligible image-bearing leaf
   bundle. Never overwrite the root, Bandgrind, or Sport Coat carousel.
7. Run check mode across all newly generated leaf carousels.
8. Compare each generated file with the former-gallery manifest. Require exact
   image coverage and order and exact effective focal coordinates. Require
   title alt text per the agreed new rule and record any canonical `hero.alt`
   difference.
9. Implement template and asset changes.
10. Remove obsolete code only after the new leaf rendering passes functional
    checks.

The migration report must state the final number of scanned leaf bundles,
image-bearing leaf bundles, generated carousel files, generated entries,
excluded AVIF files, skipped existing carousels, and resolved blockers.

## Theme cleanup

After leaf pages use `hero-carousel.html`, remove:

- `themes/cardhaus/layouts/_partials/topic-gallery.html`;
- `themes/cardhaus/layouts/_partials/gallery-image-meta.html`;
- the `initTopicGallery` JavaScript block and all topic-gallery/lightbox-only
  event handling in `themes/cardhaus/layouts/baseof.html`;
- CSS selectors and responsive rules used only by `.topic-gallery*` and
  `.topic-lightbox*`.

Audit before deleting the generic `.gallery*` rules: remove them only if no
template, shortcode, Markdown output, or project override still uses them.
Preserve Swiper assets and all `.hero-carousel*` styling and script behavior.
Use repository-wide searches after cleanup to ensure there are no remaining
references to `topic-gallery`, `gallery-image-meta`, or the removed initializer.

## Documentation updates

Update theme documentation in the same implementation change:

- `themes/cardhaus/README.md`: rename “Add a branch carousel” to cover page
  bundles; explain leaf opt-in, no implicit discovery, generated leaf
  lightboxes, generator usage, and leaf opening scroll snap.
- `themes/cardhaus/specs/SPEC.md`: replace the topic-gallery contract with the
  unified carousel contract; describe home/branch/leaf ownership, focal rules,
  and optional-link behavior.
- `themes/cardhaus/specs/carousel-linking-update.md`: mark superseded gallery notes
  as historical or update them so they do not recommend reusing the removed
  gallery markup.
- `docs/content-authoring.md`: explain that sibling images do not render a
  carousel without `carousel.yaml`, and document safe generator invocation.
- Source comments: replace “branch bundle” wording where the code now supports
  any page bundle.
- Source map lists: remove `topic-gallery.html` and identify
  `hero-carousel.html` as the single carousel renderer.

Do not erase historical decision records unless they are presented as current
instructions. Prefer a clear supersession note where historical context is
valuable.

## Verification plan

### Static and automated checks

- Run the generator test suite.
- Run `python3 scripts/validate-carousel-handoffs.py` to validate every
  image-preserving carousel handoff, including shared image identity, hash, and
  explicit destination fragment.
- Build Hugo and run `python3 scripts/validate-carousel-handoffs.py
  --rendered-dir <destination>` to validate rendered handoff anchors and
  destination `data-hash` values.
- Run generator `--check` across every generated leaf carousel.
- Validate YAML parsing and carousel schema for all carousel files.
- Assert every generated leaf entry has exactly one sibling image, a unique
  valid hash, title alt text, two in-range focal coordinates, and no `link`.
- Assert every former compatible image occurs exactly once in its migrated
  bundle, in the same sequence.
- Assert the excluded AVIF occurs zero times.
- Preserve the three pre-existing carousel files except for deliberate,
  documented carousel-handoff hash and fragment normalization.
- Run `git diff --check`.
- Run `scripts/deploy.sh build` (or the equivalent production Hugo command)
  with warnings enabled, followed by `scripts/validate-build.py` if it is not
  already part of the deploy script.
- Search rendered HTML for obsolete topic-gallery classes and source for
  obsolete partial/initializer references.

### Representative rendered-page checks

Inspect desktop and mobile output for:

- a single-image leaf;
- a multi-image leaf;
- a leaf with explicit `hero.image` and non-centered focal coordinates;
- a leaf with per-image focal metadata;
- portrait and landscape images;
- a text-only leaf, which must have no carousel;
- a leaf containing sibling images but intentionally lacking
  `carousel.yaml`, which must have no carousel;
- the homepage;
- the Bandgrind branch carousel;
- the Sport Coat branch carousel.

For leaf carousels verify article content remains below the carousel, slide
activation opens the correct full image, closing restores focus, controls and
keyboard behavior work, and no slide navigates. Confirm a leaf with a
top-of-page carousel snaps from the carousel to its article, while a leaf
without a carousel retains ordinary document scrolling.

For the three established carousels verify links, deep links, view transitions,
autoplay stopping rules, looping, focal positioning, and responsive preloads
remain unchanged.

## Acceptance criteria

Implementation is complete only when all of the following are true:

- `hero-carousel.html` is the only carousel/gallery renderer.
- A leaf carousel renders if and only if its bundle has `carousel.yaml`.
- All eligible image-bearing leaf bundles have reviewed carousel data, with
  migration counts reported from the final working tree.
- Every migrated image preserves former coverage, order, and effective focal
  coordinates; every generated alt equals its page title.
- Generated leaf entries contain no `link` and open the hero-carousel
  lightbox.
- Homepage and branch opening scroll snap still work; leaf pages with a
  top-of-page carousel snap from the carousel to the article.
- The three hand-authored carousels preserve their editorial image sequences
  and pass regression checks; image-preserving handoffs use explicit fragments
  and matching source/destination hashes.
- The generator is deterministic, refuses overwrite, reports invalid hashes
  and collisions, and has passing tests for the required modes and edge cases.
- Every generated hash removes only the final extension, replaces underscores
  and remaining periods with hyphens, and applies no other normalization.
- Obsolete gallery template, metadata partial, JavaScript, exclusive CSS, and
  current documentation references are gone.
- The production build, validation, and whitespace checks pass without new
  warnings.

## Risks and implementation notes

- Hugo resource ordering is behavior to measure, not assume. A Python lexical
  sort is acceptable only after comparison against a Hugo-generated manifest.
- YAML parsing is deceptively subtle because YAML 1.1 may coerce `y` to boolean
  true. The writer must always quote `"y"`; the reader must preserve current
  effective values.
- The project and theme both define a topic single layout. Updating only one
  can hide a stale implementation until the override changes.
- Removing broad `.gallery` CSS without a usage audit could affect Markdown or
  project-specific content unrelated to the topic gallery.
- Carousel preloading on leaf pages can increase initial bandwidth. Preserve
  the existing preload policy exactly unless profiling justifies a separate
  change.
- Old `hero.alt` may differ from the agreed generated title alt. Treat this as
  an explicit, reported migration difference rather than an unnoticed
  regression.
