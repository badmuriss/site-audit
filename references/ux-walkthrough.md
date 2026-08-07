# UX Walkthrough (Phase 1)

Walk the site AS a real user. Typing, clicking, watching, screenshotting. A static DOM read
cannot produce a verdict — if you never interacted, the verdict is **Incomplete**.

## 1. Lock a persona

Findings drift to generic "looks fine" without one. Source in order: (1) user-provided
("audit as a busy insurance broker"), (2) a persona file in the repo (`docs/personas/`,
`personas/`), (3) ask once: *"Who uses this and what are they trying to get done?"*

Capture role, tech comfort, time pressure, device. Write it at the top of the report; every
finding must be defensible from this persona. If you think "a developer would know..." — stop,
your persona doesn't.

**Also always run the first-time-user lens** (below) on every multi-page feature. It is the
single biggest blind spot in internal/AI tooling: what the builders find obvious is opaque to
someone landing cold.

## 2. Discover the surface

- **Routes** — read the router config (React Router, Next app dir, etc.) and click through
  every nav section. One line per route: `/app/clients — list, search, add new`.
- **Threads** — identify 3-5 real end-to-end tasks that make up a user's day. These are the
  spines of the audit (e.g. create a client, work today's queue, send a summary).
- **Elements** — per page, list every interactive control. Coverage is arithmetic:
  publish `tested 29 of 31 elements on /app/clients`.

## 3. Interaction log (MANDATORY)

Every walkthrough produces a timestamped log. Without it, verdict = Incomplete.

```
INTERACTION LOG — /dashboard/spaces/marketing
  Persona: SME owner, time-pressed, low tech comfort
  [x] 14:32:01 Typed "test message" into textarea[placeholder*="message"]
  [x] 14:32:05 Clicked Send (button[aria-label="Send"])
  [x] 14:32:06 Verified input cleared within 1000ms (value === "")
  [x] 14:32:08 Verified message appeared in transcript (count +1)
  [x] 14:32:12 Opened thread pane, verified main column width >= 200px
  [x] Console read after each step (0 warnings, 0 errors)
  [x] Screenshot before + after each step
  [x] Network inventoried (0 5xx, 0 403/404 on auth pages)
```

**Required per audited page:** >= 1 input typed into (real text), >= 1 primary action
triggered (Send/Save/Submit/Create), >= 1 modal or detail pane opened, >= 1 console read
after the action, >= 1 screenshot before AND after, and verification of expected post-action
state (input cleared, toast, route change, list updated).

## 4. Live interaction smoke

Code reading proves a button has an `onClick`. It does not prove clicking does anything. For
every interactive control:

1. **Click it** — pointer lands, element responds.
2. **Watch the network** — did a request fire? Right URL, method, body?
3. **Watch the DOM** — did something visibly change?
4. If nothing changed in (2) or (3), that is a bug.

Common silent failures: Approve/Deny cards that no-op, popup-blocked OAuth, optimistic delete
that never confirms, filter chips reading stale cached data, off-by-one pagination, forms whose
validation strips input silently.

## 5. First-time-user lens (mandatory per multi-page feature)

Adopt someone opening the app for the first time, no docs, no source access. Per screen ask:

- Could I complete the task without reading code or docs?
- Are labels plain language, not internal vocab (`agentClass`, `slug`, `webhook_id` leaking)?
- Do pickers show *what each option does*, not just an opaque ID/enum?
- Are defaults sensible enough to accept and move on?
- If I'd click "Skip" because I don't understand a setting, that's a UX bug.

Log a finding when the lens fires even if the screen technically works.

## 6. Responsive + layout sweep

Test widths 1440 / 1280 / 1024 / 768 / 375. For apps with sidebars/drawers/panes, test the
worst combos (all-open at 1024-1280 hides the nastiest bugs). At each width, scroll the longest
content and run layout-detection JS for overflow, clipping, and vertical-text stacks:

```js
[...document.querySelectorAll('*')].filter(el => {
  const r = el.getBoundingClientRect()
  return r.width > 0 && (el.scrollWidth > el.clientWidth + 2 || r.right > innerWidth + 2)
}).map(el => el.tagName + '.' + el.className).slice(0, 20)
```

Any collapse, overflow, or invisible text is a hard-gate High.

Layout holding at every width is the floor, not the finish. Once the sweep is clean, read
[perfection-checklist.md](perfection-checklist.md) and run it over the components that carry
the product: the per-component pass on states, spacing, focus, loading and empty variants that
separates "nothing is broken" from "this looks shipped".

## 7. Automated accessibility (axe-core, mandatory per page)

Manual keyboard walks catch focus traps; they miss ~80% of structural a11y bugs. axe-core
covers that in <1s/page. Inject once per page after it settles, re-run after opening a
modal/drawer (aria attributes shift):

```js
await page.evaluate(async () => {
  if (!window.axe) {
    await new Promise((res, rej) => {
      const s = document.createElement('script')
      s.src = 'https://cdnjs.cloudflare.com/ajax/libs/axe-core/4.10.0/axe.min.js'
      s.onload = res; s.onerror = rej; document.head.appendChild(s)
    })
  }
  const r = await window.axe.run()
  return r.violations.map(v => ({ id: v.id, impact: v.impact, help: v.help,
    nodes: v.nodes.length, sample: v.nodes[0]?.html?.slice(0, 200) }))
})
```

Severity map: axe `critical` -> Critical (hard-gate), `serious` -> High (hard-gate),
`moderate` -> Medium, `minor` -> Low. > 0 Critical or Serious on any page fails the audit.
axe is structural only — pair with a keyboard-only pass (tab order sensible, focus visible,
Escape closes, no focus trap in modals) for high-stakes apps.

## 8. Stress the edges

Run the ones that fit the app. Each catches a class of bug dev-clean data hides:

| Stress | Catches |
|--------|---------|
| Empty / 1 / 100 / 1000+ records | Edge layouts, missing virtualization, broken empty states |
| Interrupted workflow (close tab mid-form, refresh) | Lost state |
| Wrong-turn recovery | Dead ends, no back affordance |
| Round-trip A->B->A (mutate on B, return to A) | "It's just empty when I go back" — the biggest one |
| Race (double-click, fast-type-then-blur, slow network) | Optimistic-UI + debounce bugs |
| Reduced motion / high-contrast / print | Ignored media queries |
| Real-flavour data (apostrophes, accents, RTL, emoji, XSS canary, 50MB PDF, .heic) | Silent truncation, validation that strips input, unescaped injection |

The table is the checklist. When a row fires, or when the app is complex enough that the
one-liner is not enough, read the deep dive for that row:

| Deep dive | Read it when |
|---|---|
| [stress-test-recipes.md](stress-test-recipes.md) | You need the actual recipe for a row above: exact steps, what to watch, what counts as a failure |
| [data-seasoning.md](data-seasoning.md) | The app stores or renders user text. Full real-flavour battery: accents, apostrophes, RTL, emoji, zero-width, XSS canary, oversized files |
| [multi-pane-stress.md](multi-pane-stress.md) | The app has more than one pane, tab, or live region on screen at once. Cross-pane sync bugs never show up in a single-pane walkthrough |

## Notes

- **Every hesitation is a finding.** If you paused to figure out what to click, report it.
- Resize screenshots to <= 1440px longest side before referencing them (Retina 2x bloats context).
- Ask before destructive actions (delete, send, publish, pay) and anything that notifies real people.
