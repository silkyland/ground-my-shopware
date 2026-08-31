# The silent failure catalogue

Every entry here has shipped to a real merchant. None of them throws, logs, or
fails a build — which is why hunting them deliberately beats waiting for a
symptom. Read this as a checklist when a screen looks wrong, and again before you
open a merge request.

Each entry: **what the merchant sees → what the code did → the probe that proves
it.**

## 1. A button, a toolbar or a whole section is missing

**Cause: content addressed to a slot the component does not declare.** Vue drops
it without a warning. The classic is `<template #action>` on `mt-card`, which has
no `action` slot — so a card's "Add" button is not misplaced, it is absent, and
with a non-empty list there is no other way to reach the action.

Case matters: `headerRight` is a slot, `header-right` is nothing.

```bash
# every slot the component really declares
grep -n '<slot' vendor/shopware/administration/Resources/app/administration/node_modules/@shopware-ag/meteor-component-library/src/components/layout/mt-card/mt-card.vue
# in the browser: is the slot wrapper empty?
document.querySelector('.mt-card__titles-right-slot').textContent   // '' == dropped
```

The same failure shape applies to `sw-page` (only named slots render; there is no
default slot, so a modal placed outside `#content` never mounts) and to
`sw-data-grid`'s `#column-*` / `#preview-*` families, where a typo in the property
name silently falls back to the raw cell value.

## 2. A section looks cramped, unstyled, or "off"

**Cause: class names that were written as if they were styled, with no stylesheet
behind them.** This is the single most common defect in plugin admin code — a
plugin can carry dozens of BEM class names and two stylesheets.

Distinguish two cases before filing anything:

- a **root** class on a page or component with no rule is *convention* — a scoping
  and test hook;
- a **BEM element** class (`__toolbar`, `__row`, `__filters`) on a plain
  `div`/`span`/`p` with no rule is the defect. It was written to be styled.

A class on a component that styles itself (`mt-card`, `sw-container`,
`mt-banner`) is also fine — the layout comes from props, not from your class.

```bash
# classes referenced in templates
grep -rho 'class="[^"]*"' src/Resources/app/administration/src --include='*.twig' \
  | tr ' ' '\n' | grep -o 'my-prefix[a-z0-9_-]*' | sort -u
# classes with a rule (remember SCSS &__ nesting when comparing)
grep -rho '\.my-prefix[a-z0-9_-]*' src/Resources/app/administration/src --include='*.scss' | sort -u
```

## 3. Your CSS override computes to nothing

**Cause: a Meteor *scoped* style outranks it.** `.mt-card[data-v-d982cb6d]` is a
class plus an attribute (0,2,0) and beats your single class (0,1,0), so
`margin-top` silently computes to `0px`. Load order does not save you, and a tie
depends on which bundle Vite emitted first.

Fix by owning an extra ancestor (0,3,0), never by `!important`, and comment the
selector so nobody shortens it. The probe that names the winning rule is in
[measuring.md](measuring.md).

## 4. A grid scrolls sideways and clips a header

**Cause: a fixed-width container, not a wide table.** `sw-data-grid`'s wrapper is
`overflow-x: auto`, so it scrolls rather than shrinks — which is correct
behaviour. The question is whether the constraint moves with the window:

- `sw-modal` without a variant caps at **700px** — a ~640px content box — at every
  window size.
- `mt-card` caps itself at **60rem** (~960px) at every window size.
- A page's content area grows with the viewport.

So measure at two widths. Overflow at 1120px that disappears at 1920px is the
component doing its job (with a sticky actions column and a gradient
affordance). Overflow that persists at 1920px is a defect: raise the modal
variant, drop a column, or unpin column widths.

## 5. A media field shows an empty box instead of the chosen image

**Cause: `sw-media-upload-v2 variant="small"` has no preview block at all**
(`sw-media-upload-v2.html.twig:130`), and a non-empty `source` hides the
pick/upload buttons — so the field renders as a dashed rectangle with a remove
cross, and the merchant cannot even re-pick without clearing first.

Related: passing a media **id** where the component prints
`mediaNameFilter(source, source.name)` yields a blank file name, because a string
has no `.name`.

## 6. A whole run of fields below one component vanishes

**Cause: a sibling component crashed the render.** `sw-price-field` calls
`Object.values(this.value)` unconditionally, so a `null` price throws and
everything after it in the template disappears — no visible error.

When two unrelated fields go missing at once, suspect the component above them
before you suspect the missing fields. Reload with the console open; the stack
names the component.

## 7. A raw snippet key appears on screen

**Cause: the key does not resolve — in that locale.** `sw-data-grid` runs column
labels through `tWithFallback()`, which prints the key it was given. Two common
routes:

- present in `en-GB`, missing in `de-DE`;
- assembled by concatenation (`$tc('…optionType.' + type)`), where a value with no
  matching snippet prints the prefix.

Only the active locale is loaded in the running admin, so verify the other one on
disk. Enumerate the backend's own list of values (PHP constants, a `types` array)
and assert coverage.

## 8. An empty state shows a stray blank line

**Cause: a required prop passed as a bare attribute.** `mt-empty-state`'s
`description` is required and rendered unconditionally; `description` with no
value passes `""` and renders an empty text block under the headline.

More generally: a bare attribute in a Vue template is the empty string, which
satisfies a `String` prop type and looks like an omission everywhere else.

## 9. A colour, a badge or a border does not match the rest of the admin

**Cause: a hard-coded hex, or the legacy palette instead of a semantic token.**
`#ccc` in an inline style is unreachable from any stylesheet and will not follow
the design system. `var(--color-gray-300)` is subtler: it resolves — the
Administration generates a custom property from every SCSS variable
(`global.scss:4-11`) — but it is the pre-Meteor set.

Beware the inverse mistake: `grep -- '--color-gray-300:'` finds nothing because
the property is generated, so a failed grep is **not** proof the token is
undefined. Check the computed value in the browser before calling it broken.

## 10. Template specs explode with "Codegen node is missing"

**Cause: an HTML-looking tag inside a `{# … #}` Twig comment.** Shopware strips
Twig comments before Vue compiles a plugin template, so the browser is fine —
but a Jest spec that reads the raw `.twig` off disk hands Vue a stray unclosed
`<h3>` and the compile assertion fires. Native tag names trigger it;
component-looking tags usually survive.

Write tag names without brackets in comments: `` `h3` ``, `` `div` ``.

```bash
grep -Pzo '\{#[\s\S]*?#\}' path/to/template.html.twig | grep -o '<[/a-zA-Z][^>]*>'
```

## 11. A grid in a card sits in a 24px band, or a toolbar strip is empty

`mt-card` renders its `default` and `grid` slots into the same
`.mt-card__content`. A grid left in the default slot keeps the card's
`--scale-size-24` inset, so the table floats in a band while a `#toolbar` strip
above it bleeds to the card edge — two different edges in one card. And a single
button dropped into `#toolbar` produces a 69px strip that is mostly empty,
because that slot is built for a control that fills it.

Neither throws, and both look plausible until you measure the table's inset
against a core grid-in-a-card screen. The slot decision table is in
[component-contracts.md](component-contracts.md#which-slot-does-my-content-belong-in).

## 12. Comparing against the wrong class of core screen

Not a rendering defect — a reasoning defect, and the one most likely to make you
defend something wrong with real evidence. `sw-data-grid` behaves differently as
a full-page grid (`sw-data-grid--full-page`) than inside a card, and `sw-modal`
and `mt-card` both cap their width **independently of the viewport** while a page
does not.

So before concluding "core does it this way", check that the core screen you
measured is the same kind of container as yours: in-card against in-card,
full-page against full-page, modal against modal. Measuring an in-card grid
against the product list is how a 700px modal cap gets excused as normal
responsive behaviour.

## 13. A test that pins the defect in place

Not a rendering failure, but the reason one survives: a spec asserting
`toContain('<template #action>')` locks in a slot that does not exist. When a test
encodes the wrong contract, fix the test and say so in the commit — and prefer
assertions that compare against the **installed** component over assertions that
quote a remembered name.

## 12. `create_thumbnails = 1` with no thumbnail sizes linked

The media folder asks for thumbnails and never gets any, because generation is
driven by `media_folder_configuration_media_thumbnail_size`. Everything looks
right in the admin and every upload stores one full-size file. The SQL probe and
the migration pattern are in
[component-contracts.md](component-contracts.md#media-folders-and-thumbnails-in-a-migration).

## A five-line pre-review pass

```bash
grep -rn '<template #' src/Resources/app/administration/src --include='*.twig' | grep -v '#column-\|#preview-'
grep -rn ':style=\|style="' src/Resources/app/administration/src --include='*.twig'
grep -rn '#[0-9a-fA-F]\{3,6\}\b' src/Resources/app/administration/src --include='*.twig'
grep -rn -- '--color-gray-\|--color-darkgray-' src/Resources/app/administration/src
grep -rn "width: '[0-9]*px'" src/Resources/app/administration/src --include='*.ts'
grep -rln '<mt-card' src/Resources/app/administration/src --include='*.twig' | xargs grep -l 'sw-data-grid' | xargs grep -L 'template #grid'
```

Each line maps to an entry above — the last one lists every card whose grid is still in the default slot. None of them is a substitute for reading the
component, but they find the cheap defects in seconds.
