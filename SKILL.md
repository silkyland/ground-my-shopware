---
name: ground-my-shopware
description: >-
  Builds and repairs Shopware 6 Administration and Storefront UI by reading the
  INSTALLED vendor source first — `mt-card`'s real slot list, `sw-data-grid`'s
  preview slot, `sw-modal`'s variants, the Meteor token files, the snippet
  catalogues — and reusing the component core already ships instead of
  hand-rolling markup. Built for Shopware's silent failure class, where nothing
  throws: content addressed to a slot the component does not declare is dropped
  without a warning, a `variant="small"` media uploader renders no preview at
  all, a BEM class with no stylesheet leaves a section looking cramped, a scoped
  `.mt-card[data-v-…]` rule beats your single-class override and computes to
  `0px`, a required `description` prop passed as a bare attribute prints an empty
  block, a `<h3>` inside a `{# … #}` comment renders fine in the browser and
  crashes every template spec. Use this skill whenever the task touches a
  Shopware admin module, page, card, data grid, modal, media or colour picker,
  admin SCSS or snippets, a storefront Twig block or thumbnail — and especially
  when a merchant says a screen looks cramped, ugly, empty, misaligned, or "the
  picker does not show what I selected". Also use it before writing any new admin
  template, so the contract is read rather than remembered.
compatibility: >-
  Needs the Shopware source on disk (a composer install with `vendor/shopware/*`
  and the Administration's `node_modules`). Browser measurement steps need a
  logged-in Administration; skip them and mark findings UNVERIFIED when there is
  no session.
---

# Ground My Shopware

Shopware's Administration is a large design system wearing a thin Vue harness.
Almost everything you need is already built — and almost nothing tells you when
you have asked for it wrongly. A slot name that does not exist renders nothing.
A wrong variant renders unstyled. A class you invented styles nothing. The build
passes, the tests pass, and a merchant tells you the screen looks broken.

So this skill has one move, applied over and over: **read the installed
component before you write the markup that uses it.**

## The Prime Directive

> **No slot, prop, enum value, token, icon name, snippet key, or CSS assumption
> that you have not read in the installed source, at a `file:line` you can
> cite.** Not remembered from training. Not inferred from a sibling component.
> Not copied from the online docs, which describe a different patch version than
> the one that will run.

Shopware moves fast and deprecates in place. A component that took `#action` in
6.4, `sw-card` in 6.5 and `mt-card` in 6.7 is the normal case, not the exception.
Training-data memory of this framework is a hypothesis generator, never a source.

## Hard rules

1. **Reuse before you build.** Before writing a card, a grid, a picker, a
   definition list or a toolbar, check
   [references/component-contracts.md](references/component-contracts.md) for the
   component that already does it and the core file that demonstrates it. Copying
   a core screen's structure is the highest-value move available to you: it comes
   pre-styled, pre-tokenised, pre-accessible, and it survives upgrades.
2. **Silence is the enemy.** These defects do not throw. Hunt them with the
   catalogue in
   [references/silent-failures.md](references/silent-failures.md) rather than
   waiting for an error or trusting a green build.
3. **Measure, do not eyeball.** A layout diagnosis from source alone is a guess.
   The probes in [references/measuring.md](references/measuring.md) turn "looks
   cramped" into `scrollWidth 902 vs clientWidth 623`, which names the fix.
4. **Lock the contract into a test.** An audit that lives only in a commit
   message gets re-broken next sprint. Patterns in
   [references/contract-tests.md](references/contract-tests.md).
5. **Style with tokens you have grepped.** Two different packages define them and
   the legacy palette still resolves, so "it renders" is not evidence you used
   the right one. See the token section of `component-contracts.md`.
6. **Never reach for `!important`.** When an override loses, the cause is
   almost always a Meteor *scoped* style outranking you. Win on specificity with
   an ancestor class you own, and leave a comment saying why the selector is
   longer than it looks like it needs to be.

## Workflow

### 1. Locate the truth on disk

Paths differ per install; find them once and cite them from then on. The
canonical layout, and the greps that confirm it, are at the top of
[references/component-contracts.md](references/component-contracts.md). Pin the
version too — `composer.lock` or `vendor/shopware/core/composer.json` — because
a contract answer is only true for a version.

### 2. Read the contract for every component the screen touches

For each one, answer from source: which slots does it declare (exact spelling
and case — `headerRight` is not `header-right`), which props are required, which
enum values are actually styled, what does it do to layout and padding, and is
it deprecated with a due date. Record the answers with `file:line`.

The two mistakes that cost the most time here are assuming a slot name and
assuming a wrapper component is safer than the Meteor component it wraps. Both
are covered in `component-contracts.md` with the evidence.

### 3. Build on what you verified — and reuse the core screen that already does it

Never re-implement a layout primitive. The reuse table in
`component-contracts.md` maps each thing you might need to the component that
provides it *and* a core file that uses it well, because a working example
answers questions a prop list cannot: where the padding comes from, which slot
the grid belongs in, what the empty state looks like.

When you deviate, say why in a comment at the deviation. A future reader who
finds `variant="regular"` with no explanation will "simplify" it back.

### 4. Measure the result in a logged-in Administration

Screenshot it, then dump the numbers: widths, overflow, computed padding, which
rule won. Then read the console *after a reload*, so you are not diagnosing a
stale error. If you have no session, say so and mark every layout claim
`UNVERIFIED` — do not ask the user for credentials and do not type any.

### 5. Lock it in

Add or extend a contract spec so the defect cannot return. Shopware plugin repos
usually have a Jest suite that reads templates off disk; that is the right place,
and `contract-tests.md` has the assertions worth writing.

## The five-minute version

When the task is small and you only need the greatest hits:

```bash
# 1. what version am I actually targeting?
grep -m1 '"version"' vendor/shopware/core/composer.json || grep -A2 '"name": "shopware/core"' composer.lock

# 2. does this slot exist? (the single most common silent defect)
grep -n '<slot' vendor/shopware/administration/Resources/app/administration/node_modules/@shopware-ag/meteor-component-library/src/components/layout/mt-card/mt-card.vue

# 3. does this token exist, and which package defines it?
grep -rn -- '--scale-size-24:\|--color-border-secondary-default:' \
  vendor/shopware/administration/Resources/app/administration/node_modules/@shopware-ag/

# 4. does this snippet key resolve, in every locale?
grep -rn '"myKey"' src/Resources/app/administration/src/snippet/*.json

# 5. is a class I referenced actually styled anywhere?
grep -rn '\.my-class' src/Resources/app/administration/src --include='*.scss'
```

Four of those five are grep. That is the point: the expensive failures in this
framework are cheap to prevent and costly to diagnose.

## Reference files

Read the one the task needs; they are written to be skimmed by heading.

| File | Read it when |
|------|--------------|
| [references/component-contracts.md](references/component-contracts.md) | Any time you write or edit admin markup — slot lists, props, variants, tokens, the reuse table, and the core files to copy from |
| [references/silent-failures.md](references/silent-failures.md) | A screen looks wrong but nothing errors; or before review, as a checklist |
| [references/measuring.md](references/measuring.md) | You need numbers from a running Administration — overflow, box model, which CSS rule won, token values, snippet resolution |
| [references/contract-tests.md](references/contract-tests.md) | Closing the loop: guard specs, the baseline-allowlist technique, and the Twig-comment trap that breaks template specs |

## What this skill does not do

It is not a plugin architecture guide — DAL, DI, subscribers, migrations and
Store API belong to a Shopware plugin-development skill, and this one only touches
migrations where the UI depends on one (media folders and thumbnails). It also
does not decide product design. It makes sure that what you decided is what the
framework actually renders.
