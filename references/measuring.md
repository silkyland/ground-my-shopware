# Measuring a running Administration

A layout diagnosis from source alone is a guess. These probes turn "looks
cramped" into a number that names the fix, and they are the difference between
"the grid is too wide" and "the wrapper is 623px with a 902px table because the
modal has no `variant`".

## Ground rules

- **Never type credentials.** If the admin is not logged in, ask the user to log
  in, or work from source and mark every layout claim `UNVERIFIED`. Waiting costs
  less than a wrong assumption.
- **Reload before reading the console.** Buffers keep stale errors, and diagnosing
  a fixed error wastes a cycle.
- **Measure twice, at two widths.** 1920×1080 and ~1120px. What changes between
  them tells you whether a constraint is viewport-independent, which is what
  separates a defect from designed responsive behaviour.
- Anything you drive the admin with is inspection, not implementation. Fix the
  source, rebuild, re-measure — never patch the live DOM and call it done.

## Getting there

The admin route for a plugin module follows the module name: `dh-cx-warehouses`
becomes `#/dh/cx/warehouses/list`. So a list page is
`http://<shop>/admin#/dh/cx/warehouses/list`.

Inside a long session, navigating by hash and waiting is faster than reloading:

```js
location.hash = '#/dh/cx/warehouses/list';
await new Promise(r => setTimeout(r, 2500));
```

Watch out: an admin dev server (Vite, port 5173) is a **different origin** from
the shop, so a session on one is not a session on the other. Whichever origin you
measure has to be the one that is logged in — and the built bundle is only as
fresh as your last `bin/build-administration.sh`.

## Probe 1 — the box model of one element

The default report for "this looks wrong":

```js
const m = (s) => {
  const e = document.querySelector(s); if (!e) return null;
  const r = e.getBoundingClientRect(), c = getComputedStyle(e);
  return { w: Math.round(r.width), h: Math.round(r.height), left: Math.round(r.left),
           padding: c.padding, margin: c.margin, display: c.display,
           overflowX: c.overflowX, clientW: e.clientWidth, scrollW: e.scrollWidth };
};
({ dialog: m('.sw-modal__dialog'), body: m('.sw-modal__body'),
   card: m('.my-card'), wrapper: m('.sw-data-grid__wrapper'),
   table: m('.sw-data-grid__table') })
```

`scrollW > clientW` on the wrapper is the overflow, and the amount is the fix's
budget.

## Probe 2 — is the overflow viewport-independent?

The test that decides whether an overflow is a defect. Run it at 1920 wide, then
at ~1120:

```js
const routes = ['/dh/cx/warehouses/list', '/dh/cx/subscriptions/list'];
const out = { viewport: [innerWidth, innerHeight] };
for (const r of routes) {
  location.hash = '#' + r;
  await new Promise(x => setTimeout(x, 2200));
  const w = document.querySelector('.sw-data-grid__wrapper');
  out[r] = w ? w.scrollWidth - w.clientWidth : 'no grid';
}
out
```

Zero at 1920 and non-zero at 1120 → `sw-data-grid` scrolling as designed; leave it.
Non-zero at both → a fixed cap (`sw-modal` 700px, `mt-card` 60rem); fix it.

## Probe 3 — which CSS rule actually won

For the "my override does nothing" case. This lists every rule that matches the
element and touches the property, so the winner is visible:

```js
const el = document.querySelector('.my-card');
const out = [];
for (const ss of document.styleSheets) {
  let rules; try { rules = ss.cssRules } catch (e) { continue }
  for (const r of rules || []) {
    if (!r.selectorText || !/margin/.test(r.style?.cssText || '')) continue;
    try { if (el.matches(r.selectorText)) out.push({
      sel: r.selectorText, css: r.style.cssText.slice(0, 120),
      file: (ss.href || 'inline').split('/').pop() }) } catch (e) {}
  }
}
({ computed: getComputedStyle(el).margin, candidates: out })
```

A `[data-v-…]` in the winning selector means you lost to a scoped style and need
one more class of specificity.

## Probe 4 — do these tokens resolve, and to what?

The only reliable check, because the legacy palette is generated from SCSS and
never appears as a literal `--name:` in source:

```js
const cs = getComputedStyle(document.documentElement);
Object.fromEntries(['--scale-size-24','--color-border-secondary-default',
                    '--color-gray-300','--border-radius-xs']
  .map(n => [n, JSON.stringify(cs.getPropertyValue(n))]))
```

An empty string is genuinely undefined — that declaration is being dropped. A
value means the token exists, and any complaint about it is about semantics, not
breakage.

## Probe 5 — snippet keys, in the locale that is loaded

```js
const i18n = Shopware.Application.view.i18n.global;
const dig = (o, p) => p.split('.').reduce((a, k) => a && a[k], o);
({ active: i18n.locale?.value ?? i18n.locale,
   inEn: dig(i18n.getLocaleMessage('en-GB'), 'myPlugin.myModule.myKey') ?? '(absent)',
   inDe: dig(i18n.getLocaleMessage('de-DE'), 'myPlugin.myModule.myKey') ?? '(absent)' })
```

Expect the non-active locale to look empty — the admin loads only the active
one, so `(absent)` there is not evidence of a missing key. Check that locale's
JSON file on disk instead.

## Probe 6 — a grid's real column budget

When a table overflows, this says which column to cut or unpin:

```js
const heads = [...document.querySelectorAll('.sw-data-grid__cell--header')]
  .map(h => ({ label: h.textContent.trim() || '(actions)',
               w: Math.round(h.getBoundingClientRect().width),
               min: getComputedStyle(h).minWidth }));
({ heads, sum: heads.reduce((a, c) => a + c.w, 0),
   available: document.querySelector('.sw-data-grid__wrapper').clientWidth })
```

A header whose rendered width is far above its content is usually a long label or
an unresolved snippet key inflating the column — worth checking before you blame
the layout.

## Probe 7 — the focus ring, honestly

Programmatic `.focus()` often does not match `:focus-visible`, so a computed
`outline: none` proves nothing. Press a real Tab, then read:

```js
const a = document.activeElement, cs = getComputedStyle(a);
({ tag: a.tagName, cls: a.className.slice(0, 60), outline: cs.outline,
   offset: cs.outlineOffset, focusVisible: a.matches(':focus-visible') })
```

## What to record

For each defect, one line the fix can be measured against:

```
values grid: wrapper clientW 623 / scrollW 902  → 279px overflow
             (sw-modal--default caps the dialog at 700px, body padding 20/30)
after:       wrapper 821 / 821 → 0
```

Numbers before and after are what make a UI change reviewable. "Looks better" is
not a result.
