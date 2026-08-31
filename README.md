# ground-my-shopware

**Shopware ships almost every component you need — and tells you nothing when you
ask for one wrongly. Read the installed contract, reuse the core screen, measure
the result.**

`ground-my-shopware` is an agent skill for building and repairing Shopware 6
Administration and Storefront UI. It is the Shopware-specific counterpart to
[`ground-my-ui`](../ground-my-ui): where that skill teaches the stack-agnostic
method, this one carries the verified contracts — `mt-card`'s twelve slots,
`sw-data-grid`'s preview slot, `sw-modal`'s four width caps, which package defines
`--scale-size-*`, and the core file to copy for each of them.

It exists because of how this framework fails. Nothing throws:

- content addressed to `#action` on an `mt-card` is **dropped silently** — Vue has
  no such slot, so a card's "Add" button is not misplaced, it is gone, and with a
  non-empty list there is no other way to reach the action;
- `sw-media-upload-v2 variant="small"` has **no preview block at all**, so an image
  field holding a valid media id renders as an empty dashed box;
- a BEM class with no stylesheet leaves a section looking cramped, and a plugin can
  carry dozens of them against two `.scss` files;
- `.mt-card[data-v-…]` is a *scoped* style, so your single-class `margin-top`
  computes to `0px`;
- `sw-price-field` throws on a `null` price and **everything after it in the
  template disappears** with no visible error;
- `grep -- '--color-gray-300:'` finds nothing, yet the token resolves — the
  Administration generates a custom property from every SCSS variable;
- an `<h3>` inside a `{# … #}` comment renders perfectly and crashes every
  template spec.

Each of those cost real debugging time before it was written down here.

## How it works

```
1. Locate the truth on disk      vendor/, node_modules/@shopware-ag/, snippet JSON
2. Read the contract             slots, required props, styled enums, defaults
3. Build on core                 reuse table → the component + a core file using it
4. Measure in a real admin       overflow, box model, which rule won, token values
5. Lock it in                    a spec that fails when the defect returns
```

## What is in the box

| File | Contents |
|------|----------|
| `SKILL.md` | The directive, the workflow, and a five-minute grep version |
| `references/component-contracts.md` | Verified contracts for `mt-card`, `sw-data-grid`, `sw-modal`, `sw-description-list`, the media components, `mt-empty-state`, the colour picker, `sw-price-field`; the token packages; snippets; icons; `sw_thumbnails`; media-folder migrations — plus the reuse table mapping each need to a core example |
| `references/silent-failures.md` | Twelve failures that raise no error, each as *what the merchant sees → what the code did → the probe that proves it*, and a five-line pre-review pass |
| `references/measuring.md` | Browser probes for a logged-in Administration: box model, viewport-independent overflow, which CSS rule won, token resolution, snippet catalogues, focus rings |
| `references/contract-tests.md` | Guard-spec patterns that compare your markup to the vendor tree, and the baseline-allowlist technique that lets a repo-wide guard land green on day one |

## When it triggers

Anything touching a Shopware admin module, page, card, data grid, modal, media or
colour picker, admin SCSS or snippets, a storefront Twig block or thumbnail — and
especially a merchant report that a screen looks cramped, empty, misaligned, or
that "the picker does not show what I selected". Also before writing a new admin
template, so the contract gets read instead of remembered.

## Scope

It is not a plugin architecture guide: DAL, DI, subscribers, Store API and
migrations belong elsewhere, and this skill touches migrations only where the UI
depends on one (media folders and thumbnail sizes). It does not decide product
design either — it makes sure what you decided is what the framework renders.

Contracts are pinned to **Shopware 6.7.x** with `file:line` citations. Line
numbers drift across patch releases; the skill's first instruction is to re-verify
against the copy on your disk, and every claim is written to be checkable in one
grep.

## Licence

MIT — see [LICENSE](LICENSE).
