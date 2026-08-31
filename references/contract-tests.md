# Locking the contract into a test

A contract you verified once decays. The component's slot list is in
`node_modules`, the snippet key is in a JSON file, the class name is in a
template — nothing in the toolchain checks that they still agree, and every one of
these defects passes review because it renders *something*.

So the last step of any fix is a test that fails when the defect returns. In a
Shopware plugin these are cheap: read the files off disk and assert relationships
between them. No kernel, no browser, milliseconds.

## Where they go

Most Shopware plugins already have a Jest suite for the Administration
(`tests/Jest/...`) with `moduleNameMapper` entries for `.twig` and `.scss`, and
specs that mount templates read from disk. Follow the repo's existing shape
rather than inventing one. If a spec already mounts your template, extend it
instead of adding a parallel file.

Two setup traps worth knowing before you run anything:

- **The Administration build empties a plugin's `node_modules`.** After
  `bin/build-administration.sh`, `npx jest` fails with "jest-environment-jsdom
  cannot be found" until `npm ci` runs again. That is not a test regression.
- **Record the failure baseline before you start.** Most repos carry a few
  pre-existing failures. Note which suites they are in, so "my change broke
  nothing" is a comparison rather than a hope.

## The assertions worth writing

Each one below maps to an entry in
[silent-failures.md](silent-failures.md). Name the defect in a comment — a test
whose reason is written down survives refactoring.

### 1. Slot names, checked against the installed component

The highest-value assertion in this file, because it compares your markup to the
vendor tree rather than to a remembered name:

```ts
const MT_CARD_SLOTS = ['before-card','avatar','title','subtitle','headerRight',
  'context-actions','tabs','toolbar','default','grid','footer','after-card'];

it('pinned slot list still matches the installed mt-card', () => {
  if (!fs.existsSync(METEOR_CARD_PATH)) return;   // plugin checked out without the shop
  const declared = [...read(METEOR_CARD_PATH).matchAll(/<slot\s+name="([^"]+)"/g)]
    .map(m => m[1]);
  expect(declared.sort()).toEqual([...MT_CARD_SLOTS].sort());
});

// The defect: `<template #action>` on an mt-card. Vue drops content addressed to
// a slot that does not exist, so the card's Add button was absent, not misplaced.
it.each(templateNames)('%s addresses only slots mt-card declares', (name) => {
  const used = [...templates[name].matchAll(/<template\s+#([A-Za-z-]+)[\s=>]/g)]
    .map(m => m[1]);
  const foreign = ['actions','action-modals','button','label','modal-footer',
                   'smart-bar-header','smart-bar-actions','language-switch','content'];
  const cardSlots = used.filter(s =>
    !s.startsWith('column-') && !s.startsWith('preview-') && !foreign.includes(s));
  expect(MT_CARD_SLOTS).toEqual(expect.arrayContaining(cardSlots));
});
```

The pinned list plus the vendor cross-check is deliberate: the cross-check keeps
the list honest when Shopware changes, and the pin keeps the test meaningful in a
CI job that has no `vendor/` tree. Filter the slot families other components own
(`sw-data-grid`'s `column-*`/`preview-*`/`actions`, `sw-page`'s smart bar,
`sw-modal`'s footer) or the assertion will fight itself.

### 2. Every referenced snippet key resolves, in every locale

```ts
const referenced = [...new Set(
  allSourceText.match(/myPlugin\.[A-Za-z0-9.]+/g) ?? []
)].filter(k => !k.endsWith('.'));   // '…optionType.' is a concat prefix, tested below

it.each(['en-GB','de-DE'])('every referenced key resolves in %s', (locale) => {
  const missing = referenced.filter(k => typeof resolve(catalogues[locale], k) !== 'string');
  expect(missing).toEqual([]);
});
```

Merge the plugin catalogue with core's (`src/app/snippet/{en,de}.json` and every
`src/module/*/snippet/{en,de}.json`) before asserting, or every `global.default.*`
key reports as missing.

### 3. Dynamic key prefixes, covered against the source of truth

The concatenated-key case, which the assertion above deliberately skips:

```ts
// A type added to the list without a snippet renders the raw key in the dropdown.
it.each(['en-GB','de-DE'])('every option type has a label in %s', (locale) => {
  const types = [...typesArrayBlock.matchAll(/'([a-z]+)'/g)].map(m => m[1]);
  expect(types.length).toBeGreaterThan(5);          // the extraction still works
  const missing = types.filter(t =>
    typeof resolve(catalogues[locale], `myPlugin.optionType.${t}`) !== 'string');
  expect(missing).toEqual([]);
});
```

Read the value list from wherever it is actually defined — a PHP constant block,
a `types` array — so adding a value there and forgetting the snippet fails here.
The `toBeGreaterThan` guard matters: if your extraction regex breaks, an empty
list would otherwise pass silently.

### 4. No literal colours or geometry in inline styles

```ts
it.each(templateNames)('%s hard-codes no colours or geometry', (name) => {
  const inline = [...templates[name].matchAll(/\sstyle="([^"]*)"/g)].map(m => m[1]);
  inline.forEach(decl => {
    expect(decl).not.toMatch(/#[0-9a-fA-F]{3,8}\b/);
    expect(decl).not.toMatch(/\b\d+px\b/);
  });
});
```

Data-driven colour through `:style="{ backgroundColor: item.colorHex }"` stays
legal — the assertion targets literals, which are the ones that cannot follow the
design system.

### 5. Grid columns do not re-pin their widths

```ts
// Fixed widths pushed the table past the 60rem mt-card caps itself at, so the
// grid scrolled sideways at every window size. Core's own nested list sets none.
it.each([['detailPage','optionColumns'], ['modal','valueColumns']])
  ('%s %s pins at most one column width', (name, symbol) => {
    const block = sliceObjectLiteral(scripts[name], symbol);
    const widths = [...block.matchAll(/width:\s*'(\d+)px'/g)];
    expect(widths.length).toBeLessThanOrEqual(1);
  });
```

### 6. Every class you style is a class you use, and vice versa

Two directions, both cheap:

```ts
it('every class the stylesheet styles is one the template uses', () => {
  styledClasses.forEach(c => expect(template).toContain(c));
});

it('every BEM element class on a plain element has a rule', () => {
  expect(inertClasses).toEqual(INTENTIONALLY_UNSTYLED);
});
```

Keep `INTENTIONALLY_UNSTYLED` tiny and commented — a root scoping hook is a
legitimate entry, a `__toolbar` is not.

### 7. The specificity fix stays specific

When you had to beat a scoped style, assert the shape survives:

```ts
// mt-card's own rule is a scoped style (.mt-card[data-v-…]) and outranks a single
// class, so this selector must stay nested. Do not "simplify" it back.
it('the values card margin outranks mt-card scoped style', () => {
  const line = scss.split('\n').find(l => l.includes('__values.mt-card'));
  expect(line).toMatch(/^\s+\./);                       // nested, so three classes
  expect(scss).toContain('.my-modal {');
});
```

### 8. No tag-shaped text inside Twig comments

The trap that breaks every template-mounting spec at once:

```ts
it.each(templateNames)('%s has no HTML tags inside twig comments', (name) => {
  const comments = templates[name].match(/\{#[\s\S]*?#\}/g) ?? [];
  comments.forEach(c => expect(c).not.toMatch(/<[/a-zA-Z][^>\n]*>/));
});
```

## The baseline-allowlist technique

A repo-wide guard spec normally cannot land until every offender is fixed, which
means it lands never. Ship it with an explicit baseline instead, and assert the
baseline **from both sides**:

```ts
const INLINE_STYLE_BASELINE = [
  'component/order-configuration-modal/…twig',   // removed in phase 2
  'extension/order-line-items-grid/…twig',       // removed in phase 3
];

it('no new offenders', () => expect(offendersNotInBaseline).toEqual([]));
it('no stale baseline entries', () => expect(baselineEntriesWithNoOffender).toEqual([]));
```

The second assertion is what keeps this honest: an entry whose file was fixed
fails the build, so the allowlist shrinks as the work lands and cannot quietly
become a permanent exemption list. Every entry carries the phase that removes it.

## When a test encodes the wrong contract

If a spec asserts `toContain('<template #action>')`, it is pinning a defect in
place. Fix the assertion, and say so in the commit — a reviewer needs to see that
the test changed *because the contract was wrong*, not because the code stopped
satisfying it.

The same applies to convention tests that are stricter than their own stated
intent. A rule requiring an element directly after a template's root Twig block,
written to catch an inert root `<template>`, should not forbid a `{# … #}` comment
above the tag it explains — a comment is stripped before Vue compiles and cannot
produce a node. Relax the pattern and record why in the test.

## What not to test

- Rendered pixel values. They belong in the measurement pass, not in a spec that
  has no browser.
- A component library's own behaviour. Assert *your* usage of it.
- Anything you cannot name a defect for. A test with no story is noise a future
  reader will delete — correctly.
