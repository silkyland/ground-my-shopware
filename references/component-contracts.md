# Component contracts — Shopware 6.7 Administration

Every fact below was read from an installed **6.7.13.0** tree and, where it
concerns layout, confirmed by measuring a running Administration. Line numbers
drift across patch releases: treat them as "look here", re-grep before you rely
on one, and update this file when a version proves it wrong.

## Contents

- [Where the truth lives](#where-the-truth-lives)
- [The reuse table](#the-reuse-table)
- [mt-card](#mt-card)
- [sw-data-grid](#sw-data-grid)
- [sw-modal](#sw-modal)
- [sw-description-list](#sw-description-list)
- [Media components](#media-components)
- [mt-empty-state](#mt-empty-state)
- [Colour picker, and the wrapper trap](#colour-picker-and-the-wrapper-trap)
- [sw-price-field, the render killer](#sw-price-field-the-render-killer)
- [Design tokens: two packages, one legacy palette](#design-tokens-two-packages-one-legacy-palette)
- [Snippets](#snippets)
- [Icons](#icons)
- [Storefront: sw_thumbnails and media](#storefront-sw_thumbnails-and-media)
- [Media folders and thumbnails in a migration](#media-folders-and-thumbnails-in-a-migration)

## Where the truth lives

Relative to the shop root:

| What | Path |
|------|------|
| Core PHP | `vendor/shopware/core/` |
| Administration source | `vendor/shopware/administration/Resources/app/administration/src/` |
| Meteor components | `…/administration/node_modules/@shopware-ag/meteor-component-library/src/components/**/mt-*.vue` |
| `--scale-size-*` steps | `…/node_modules/@shopware-ag/meteor-component-library/dist/index.css` |
| Semantic tokens | `…/node_modules/@shopware-ag/meteor-tokens/deliverables/administration/light.css` |
| Icon set | `…/node_modules/@shopware-ag/meteor-icon-kit/icons/{regular,solid}/*.svg` |
| Core admin snippets | `src/app/snippet/{en,de}.json` plus `src/module/*/snippet/{en,de}.json` |
| `sw-*` registration (async) | `src/app/component/index.ts` |
| `mt-*` registration | `src/app/adapter/view/vue.adapter.ts` |
| Storefront views | `vendor/shopware/storefront/Resources/views/storefront/` |

Confirm them, and find any component fast:

```bash
V=vendor/shopware/administration/Resources/app/administration
find $V/src/app -type d -name 'sw-data-grid'          # a core component
find $V/node_modules/@shopware-ag/meteor-component-library/src -name 'mt-card.vue'
grep -m1 '"version"' vendor/shopware/core/composer.json
```

**Read the component, not just its template.** A Shopware component is `index.js`
or `index.ts` (props, computed, defaults) plus `*.html.twig` (slots, structure)
plus `*.scss` (what it does to layout). Layout questions are usually answered in
the SCSS, and default values only in the index.

## The reuse table

Before building anything, check whether it is already here. The "core example"
column matters as much as the component: a working screen shows you which slot
the thing belongs in and where the padding comes from.

| You need | Use | Core example to copy from |
|----------|-----|---------------------------|
| A section with a title and a framed body | `mt-card` | anywhere; `sw-product-cross-selling-form.html.twig:4` |
| One action button in a card header | `mt-card` `#headerRight` | `sw-product-cross-selling-form.html.twig:8` |
| A search field + buttons above a list | `mt-card` `#toolbar` | `sw-settings-country-state.html.twig:6-9` |
| A grid filling a card edge to edge | `mt-card` `#grid` + `sw-data-grid plain-appearance` | `sw-property-option-list.html.twig` |
| A nested collection editor (values of an option, options of a group) | `mt-card` + grid + a detail modal | `sw-property-option-list` + `sw-property-option-detail` |
| A grid inside a modal | `sw-modal variant="full"` + a toolbar band + `plain-appearance` | `sw-import-export-edit-profile-modal` + `…-modal-mapping` |
| A thumbnail beside a row's primary cell | `sw-data-grid` `#preview-<property>` + `sw-media-preview-v2` | `sw-product-list.html.twig:152-153` |
| One image field with preview, upload and library browse | `sw-media-upload-v2 variant="regular"` + `sw-media-modal-v2` + `sw-upload-listener` | `sw-cms-el-config-image.html.twig:1-60` |
| A compact "pick existing media by id" field | `sw-media-field` (`v-model:value` on a `mediaId`) | `sw-dynamic-url-field.html.twig:79-86` |
| Label/value pairs (order data, summaries) | `sw-description-list` with a `grid` prop | `sw-review-detail.html.twig:115` |
| A colour swatch input | `mt-colorpicker` | `sw-property-option-detail.html.twig:33-39` |
| Gross/net price with currency and tax | `sw-price-field` | `sw-property-option-detail`, product pricing |
| An empty list state | `mt-empty-state` | `sw-property-option-list.html.twig` bottom |
| Muted helper text, visually-hidden labels | a shared utility class in your plugin's `style/`, imported from `main.ts` | — |

Two habits that follow from this table:

- **A `<div>` with a class you invented is the last resort, not the first.** If
  you are writing `display: flex` in a template, check whether `sw-container`,
  `mt-card`'s slots, or the grid's own cell handling already does it.
- **Grep core for the component you are unsure about.** `grep -rn '<sw-media-field'
  vendor/…/src/module/` tells you whether core trusts it, how, and with which
  props — in one command.

## mt-card

`node_modules/@shopware-ag/meteor-component-library/src/components/layout/mt-card/mt-card.vue`

**Slots — all twelve, and there is no `action`:**

```
before-card  avatar  title  subtitle  headerRight  context-actions
tabs  toolbar  default  grid  footer  after-card
```

`mt-card.vue:3-94`. Note `headerRight` is **camelCase**; `#header-right` renders
nothing at all. Vue drops content addressed to an undeclared slot silently, so a
button in `#action` does not move — it disappears, and with it any way for the
merchant to reach that action.

**Props:** `title`, `subtitle`, `isLoading`, `inheritance`, and a deprecated
`large` (`:107-112`). `position-identifier` is a leftover from the old `sw-card`
and is not a prop — core still passes it in places; do not copy that.

**Behaviour that decides your layout:**

| Fact | Source |
|------|--------|
| Header renders only when there is a `title`, `subtitle` or `avatar` slot | `:152-154` |
| Filling `#grid` adds `mt-card--grid`, which sets `.mt-card__content { display: grid; padding: 0 }` — that is how a grid spans the card | `:158`, `:303` |
| `#headerRight` is right-aligned by `.mt-card__titles-right-slot { margin-left: auto }` — no CSS of yours needed | `:368` |
| `.mt-card__toolbar` and `.mt-card__avatar` are `display: none` when empty | `:327-336`, `:338-352` |
| The card is a **scoped** style: `.mt-card[data-v-…] { max-width: 60rem; margin: 0 auto var(--scale-size-40) }` | `MtCard-*.css` |

That last row is the one that bites. A class plus an attribute selector (0,2,0)
outranks your single class (0,1,0), so `.my-card { margin-top: 32px }` computes to
`0px` with no warning. Win on specificity with an ancestor you own:

```scss
.my-modal {
    /* Three classes on purpose: mt-card's own rule is a scoped style, and a
       class+attribute selector outranks a single class. Do not "simplify". */
    .my-modal__values.mt-card { margin-top: var(--scale-size-32); }
}
```

`max-width: 60rem` also means a card is ~960px wide **at every window size**, so a
table that needs more will scroll sideways forever. That is a real defect, unlike
a grid that only overflows on a narrow window.

## sw-data-grid

`src/app/component/data-grid/sw-data-grid/`

**Slots:** `#column-<property>`, `#column-label-<property>` (`html.twig:109`),
`#preview-<property>` (`:317-322`), `#actions`, `#more-actions`, `#detail-action`,
`#pagination`, `#bulk`.

`#preview-<property>` is the one most often missed. It renders inside the cell
*before* the cell content, and the grid styles whatever media component it finds
there — 48px, 32px in compact mode, with the border and radius tokens
(`scss:419`, `:498`). That is how core attaches product images to the name column
(`sw-product-list.html.twig:152-153`) instead of spending a column on them.

**Props whose defaults surprise people** (`index.js:140-160`):

| Prop | Default | Effect |
|------|---------|--------|
| `compactMode` | **true** | 42px rows, 32px media previews |
| `plainAppearance` | false | when true, drops the grid's own cell borders — use it inside a card or modal that already draws a frame |
| `showPreviews` | true | the user-facing toggle for `#preview-*` slots |
| `showSelection`, `showActions` | true | |

**Column definitions** accept `property`, `label`, `primary`, `width`, `align`,
`inlineEdit`, `naturalSorting`, `routerLink`, `allowResize`, `sortable`. Two
things to know:

- `label` is run through `tWithFallback()` (`index.js:570-571`), so a snippet key
  works — and an unresolved key **prints itself as the column header**.
- `width` is a floor, not a target. Fixed widths plus flexible columns push the
  table's intrinsic width past its container, and the wrapper is
  `overflow-x: auto` (`scss:25`), so it scrolls rather than shrinks. Core's own
  nested list sets **no widths at all** (`sw-property-option-list/index.js:197`).
  Prefer that; pin only a genuinely narrow column.

Horizontal scroll is designed behaviour, not automatically a bug: the actions cell
is `position: sticky; right: 0` with a gradient affordance (`:362`, `:458`), and
`min-width: 88px` (`:250`). Judge it by whether the constraint moves with the
viewport — see [measuring.md](measuring.md).

## sw-modal

`src/app/component/base/sw-modal/`

| Variant | Max width | Source |
|---------|-----------|--------|
| `small` | 500px | `sw-modal.scss:50` |
| `default` (when omitted) | **700px** | `:42` |
| `large` | 900px | `:46` |
| `full` | 100% minus a 20px margin | `:80-85` |

Valid values are validated in `index.js:44-53`. Body padding is `20px 30px`
(`:153`), so a `default` modal gives you a **~640px content box** — reliably too
narrow for a data grid. Core sizes up for exactly this reason: the CMS layout
picker is `large`, and the import/export profile modal, which hosts a grid with a
toolbar, is `full`.

A section that should span the full modal width bleeds past the body padding with
negative margins — core writes `margin-left: -30px; margin-right: -30px`
(`sw-import-export-edit-profile-modal.scss:1-6`). A card inside a modal should
**not** do that; it is a framed box and belongs inside the gutter.

## sw-description-list

`src/app/component/base/sw-description-list/`

A `<dl>` whose `grid-template-columns` comes from the `grid` prop, default `1fr`
(`index.js:23-29`). `dt` and `dd` get `padding: 15px` and
`border-bottom: 1px solid var(--color-border-secondary-default)`, with the last
row's border transparent; `dt` is `--font-weight-semibold` (`scss:1-27`).

This is the component people most often rebuild by hand out of `div`s with
`display: flex`, `font-weight: 600` and a hard-coded border colour. If you need
three columns per row, emit `<dt>` + two `<dd>`s and set
`grid="max-content 1fr max-content"` — and always emit the trailing `dd`, empty
when there is no value, or the next row shifts.

## Media components

**`sw-media-upload-v2`** — `src/app/component/media/sw-media-upload-v2/`

The trap: the preview block is `v-if="variant === 'regular'"`
(`sw-media-upload-v2.html.twig:130`). Under `variant="small"` there is **no
thumbnail at all**, and because a non-empty `source` also hides the pick/upload
buttons, a field holding a media id renders as an empty dashed box with a remove
cross. Use `variant="regular"` for a single image field.

`source` accepts a media **entity** or an id string. Pass the entity when you
have it: the file-name line calls `mediaNameFilter(source, source.name)` and
prints nothing for a string. The thumbnail works either way, because
`sw-media-preview-v2` resolves an id itself (`sw-media-preview-v2/index.js:348`)
— so `:source="entity || mediaId"` shows the picture on first paint and the name
when the request lands.

`default-folder` sets where uploads go. Pair the field with `sw-upload-listener`
(`auto-upload`, `@media-upload-finish`) and `sw-media-modal-v2` for library
browsing; `@media-upload-sidebar-open` is the event that opens it.

**`sw-media-field`** — `src/app/component/media/sw-media-field/`

A compact picker bound to a `mediaId` string (`v-model:value`), which is exactly
right for a plain draft object. It loads the entity itself, shows it as a
`sw-media-media-item` row with thumbnail and file name, and its popover both
searches existing media and uploads new. It knows it may live in a modal:
`popoverConfig` retargets the popover at `.mt-modal` (`index.js:95-101`).

Its one hard limit: the suggestion list is filtered to
`mediaFolder.defaultFolder.entity = default-folder` (`index.js:131`). With a
`default-folder` set, it can only ever offer media already in that folder — a
merchant searching for an image that lives in Product Media gets an empty list
with no hint why. Omit `default-folder` for library-wide search, or use the
upload-v2 + modal pair when browsing matters.

**`sw-media-modal-v2`** — `src/module/sw-media/component/sw-media-modal-v2/`

Props: `initialFolderId`, `entityContext`, `defaultTab` (`library`|`upload`),
`allowMultiSelect`, `fileAccept`. Emits `media-modal-selection-change` with the
loaded entities — keep the first one instead of re-fetching it.

## mt-empty-state

`node_modules/@shopware-ag/meteor-component-library/src/components/layout/mt-empty-state/mt-empty-state.vue`

`headline`, `description` and `icon` are **required**; `description` is rendered
unconditionally (`:47-61`, template `:12-14`). A bare `description` attribute
passes `""` and leaves an empty text block under the headline — the prop is a
question the component is asking, and the answer belongs in your snippet file.
`centered` exists but is deprecated for Meteor v5; `true` is the behaviour that
survives it. There is no `absolute` prop — that was the old `sw-empty-state`.

## Colour picker, and the wrapper trap

Use **`mt-colorpicker`** with `v-model` and a `:z-index` (core does exactly this
in `sw-property-option-detail.html.twig:33-39`). Its `modelValue` is null-safe
(`|| ""`, `mt-colorpicker.vue:464`).

The trap is reaching for `sw-colorpicker` because a Meteor component "might not
be registered yet". Verify that reasoning before acting on it:
`MtColorpicker` is a lazy async component (`vue.adapter.ts:485`) — but
`sw-colorpicker` is registered lazily too (`app/component/index.ts:266`) **and
renders `mt-colorpicker` inside itself**, so the wrapper can only ever be one
async hop slower, never more reliable. It is also deprecated for removal in 6.8.

Generalise the lesson: an `sw-*` wrapper around an `mt-*` component adds a hop
and a deprecation, not safety. When a Meteor field "never appears", the cause is
usually a sibling component that crashed the render — see the next section.

## sw-price-field, the render killer

`src/app/component/form/sw-price-field/index.js:163` calls
`Object.values(this.value)` unconditionally. Pass `null` and the whole component
tree throws: **everything after `sw-price-field` in the template disappears
too**, with no error visible in the UI. A colour picker and a media field
vanishing at once is the signature.

Normalise the price before render:

```js
if (!Array.isArray(draft.surchargePrice)) {
    draft.surchargePrice = [{
        currencyId: Shopware.Context.app.systemCurrencyId,
        gross: 0, net: 0, linked: true,
    }];
}
```

## Design tokens: two packages, one legacy palette

Three families, and knowing which file defines what saves an hour:

| Family | Defined in | Notes |
|--------|-----------|-------|
| `--scale-size-*` | `meteor-component-library/dist/index.css` | **Not** in the token package. Steps: 0,1,2,4,6,8,10,12,14,16,18,20,22,24,26,28,30,32,36,40,48,56,64,72,80,128,160,192,224,256. There is no 44, 52 or 60. |
| `--color-*`, `--border-radius-*`, `--font-size-*`, `--font-weight-*`, `--font-line-height-*` | `meteor-tokens/deliverables/administration/light.css` | the semantic set; use these |
| `--color-gray-*`, `--color-darkgray-*`, and every other legacy variable | generated from `src/app/assets/scss/variables.scss` by `src/app/assets/scss/global.scss:4-11` | pre-Meteor palette |

That generator is worth understanding, because it defeats the obvious check:

```scss
:root {
    @each $name, $value in meta.module-variables("variables") {
        --#{$name}: #{meta.inspect($value)};
    }
}
```

Every SCSS variable becomes a custom property, so `grep -- '--color-gray-300:'`
finds **nothing** while the property resolves perfectly at runtime. Do not file
"undefined token" from a failed grep — check the computed value in the browser
(see [measuring.md](measuring.md)). The finding is still real, just different:
legacy palette, not broken palette. Prefer the semantic token.

A dark token file ships (`dark.css`) but nothing in the 6.7 Administration
imports it, and the running admin sets no `data-theme`. Do not argue for tokens
on dark-mode grounds in this version; argue consistency and upgrade safety.

## Snippets

Plugin admin snippets live in `src/Resources/app/administration/src/snippet/{en-GB,de-DE}.json`
and are read from disk at runtime, so an added key resolves without a rebuild.
Only the active locale's catalogue is loaded into the running admin, which means
you can check `en-GB` in the browser but must check `de-DE` on disk.

Two ways a raw key reaches the screen, both silent:

- A key present in one locale and missing in the other.
- A key assembled by concatenation — `$tc('…optionType.' + type)`,
  `$tc('…movement.type' + suffix)`. Every value the backend can produce needs a
  snippet, and the fallback prints the bare prefix. Enumerate the source of truth
  (a PHP constant list, a `types` array) and assert coverage in a test.

## Icons

Names are `regular-*` or `solid-*` plus the file name in
`node_modules/@shopware-ag/meteor-icon-kit/icons/{regular,solid}/`. `products.svg`
means `regular-products` is valid. A wrong name renders an empty box, so confirm
with `ls` rather than intuition.

## Storefront: sw_thumbnails and media

`{% sw_thumbnails %}` is a Twig tag, not a function
(`vendor/shopware/storefront/Framework/Twig/TokenParser/ThumbnailTokenParser.php`).
Two details decide whether your call works:

- The **first argument becomes the template variable `name`** (block scoping),
  *not* a CSS class (`:20-35`). Put the class in `attributes`.
- `sizes.default` short-circuits the breakpoint machinery and emits a plain
  `sizes="…"` — right for a fixed-size chip
  (`views/storefront/utilities/thumbnail.html.twig:112-114`).

```twig
{% sw_thumbnails 'my-swatch-thumbnails' with {
    media: value.media,
    sizes: { default: '48px' },
    attributes: { class: 'my-swatch', alt: label, loading: 'lazy' }
} %}
```

With no thumbnails it degrades to a plain `<img src>` (`:110`), so it is safe to
merge before the thumbnails exist. And you do **not** need to add a
`media.thumbnails` association: `MediaLoadedSubscriber::unserialize()` fills the
collection from the denormalised `thumbnails_ro` blob whenever the association was
not loaded (`vendor/shopware/core/Content/Media/Subscriber/MediaLoadedSubscriber.php:31-43`).

`sw_thumbnails` needs a media **entity**. An `<img>` pointing at your own
controller route — a private-upload proxy, a signed download — cannot use it and
is not a defect.

## Media folders and thumbnails in a migration

A plugin that owns a media folder needs three inserts (`media_default_folder`,
`media_folder_configuration`, `media_folder`) — and a fourth step that is easy to
miss: **`create_thumbnails = 1` does nothing on its own.** Generation is driven by
rows in `media_folder_configuration_media_thumbnail_size`. Without them the flag
is inert and every upload stores exactly one full-size file.

Core's own pattern, worth copying line for line
(`vendor/shopware/core/Migration/V6_5/Migration1687462843ProductManufacturerMediaThumbnails.php:25-53`):

1. declare the sizes you want;
2. `ensureThumbnailSizes($sizes, $connection)` from
   `core/Migration/Traits/EnsureThumbnailSizesTrait.php` — it reuses existing
   `media_thumbnail_size` rows and creates only what is missing;
3. `REPLACE INTO media_folder_configuration_media_thumbnail_size`;
4. `registerIndexer($connection, 'media_folder_configuration.indexer')`.

Verify with SQL rather than by reading the migration back:

```sql
SELECT d.entity, c.create_thumbnails,
       (SELECT COUNT(*) FROM media_folder_configuration_media_thumbnail_size m
         WHERE m.media_folder_configuration_id = c.id) AS sizes
FROM media_default_folder d
JOIN media_folder f ON f.default_folder_id = d.id
JOIN media_folder_configuration c ON c.id = f.media_folder_configuration_id;
```

A folder with `create_thumbnails = 1` and `sizes = 0` is the defect. Existing
files also need `bin/console media:generate-thumbnails --folder-name="<folder>"`
— configuration only governs future generation.
