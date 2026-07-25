---
name: site-audit
description: |
  Audit a RUNNING website or web app end to end: UX walkthrough, on-page SEO + AI visibility (AEO/GEO), and Core Web Vitals performance, in one consolidated severity-ranked report. Needs a live URL or a local dev server (e.g. http://localhost:5173) — this is not a static code review. Interaction-first: an audit that never typed, clicked, or navigated ends with verdict "Incomplete". Hard gates: console errors/warnings = 0, network 5xx = 0, layout collapse = 0, axe-core Critical/Serious = 0, perf budget green (LCP <= 2.5s, CLS <= 0.1, INP <= 200ms). Agent-neutral — works with Claude Code, Codex, or any host using whatever browser tooling is available: Chrome DevTools MCP, Playwright MCP, or plain curl + lighthouse CLI as fallback. Trigger with "audit the site", "site audit", "qa sweep", "seo check", "lighthouse", "check performance", "is the site ready to ship", "walkthrough", "find what's broken", "aeo audit", "geo audit", "ai visibility", "will chatgpt cite my site", "llm seo".
license: MIT
---

# Site Audit

One workflow, three phases, one report. Audits a **live** site or app — you need a
reachable URL or a running local server. If you have neither, stop and ask for one.

Run the phases **in order**. Each phase says "read its reference now" — read it, do the
work, capture evidence, then move on. Do not skip ahead. The consolidated report at the
end ranks every finding by severity with reproduction steps and a suspected location.

## Before you start — pick your tooling

Use whatever the host has, in this preference order (any tier is valid):

| Tier | Tools | Notes |
|------|-------|-------|
| Best | Chrome DevTools MCP or Playwright MCP | Real interaction, console + network capture, traces |
| Good | `playwright` / `puppeteer` CLI driven from a script | Same coverage, more manual |
| Fallback | `curl` + `lighthouse` CLI + `npx axe` | No live interaction — caps verdict at "Incomplete" for the UX phase, but SEO + perf still run |

Pin the viewport at 1440x900 to start (mobile sweep uses 375). Do not go above 2000px wide.

## Verdict states

The audit ends in exactly one:

- **Pass** — Critical = 0, High = 0, all hard gates green, proof-of-interaction present.
- **Conditional Pass** — same, but Medium/Low findings remain.
- **Fail** — at least one Critical or High finding, OR a hard gate red.
- **Incomplete** — no proof of interaction, a phase was skipped, or the audit-the-audit
  meta-check fires (see last section). "It looked OK" is never a Pass.

## Hard gates (auto-fail, cannot be downgraded)

| Gate | Threshold | Severity |
|------|-----------|----------|
| Console errors during walkthrough | > 0 | Critical |
| Console warnings during walkthrough | > 0 | High |
| Network 5xx | > 0 | Critical |
| Network 403/404 on authenticated pages | > 0 | High |
| Layout collapse at any tested viewport | > 0 | High |
| axe-core Critical violations on any page | > 0 | Critical |
| axe-core Serious violations on any page | > 0 | High |
| LCP on representative route | > 2.5s | High |
| CLS on representative route | > 0.1 | High |
| INP on representative route | > 200ms | High |
| No proof of interaction (typing/clicking/navigating) | n/a | Incomplete |

A console warning is High *minimum*. A 5xx is Critical *automatically*. There is no
"Medium console error" — that category does not exist here. If the target has known-noisy
categories (Sentry info logs, extension chatter, expected 401 on auth probes), note them
as an allowlist up front and show both raw and reportable counts in the verdict block.

The Phase 2 bot-UA parity probes (`aeo-geo.md` §1) intentionally provoke 403/429 responses.
They are AEO findings, not network hard-gate hits — never count them in the 403/404 row.

## Phase 1 — UX walkthrough

**Read `references/ux-walkthrough.md` now.** Then walk the site as a real user: pick a
persona, inventory the routes, and for each key page type into an input, trigger a primary
action, open a modal/detail pane, read the console, screenshot before and after, and verify
the expected post-action state. Run axe-core per page, sweep responsive widths, and log
every interaction with a timestamp and selector. No interaction log = verdict Incomplete.

## Phase 2 — On-page SEO + AI visibility

**Read `references/seo-onpage.md` now.** For each significant page, check the on-page
signals a page controls: title and meta description quality + length, canonical, OG/Twitter
cards, heading hierarchy (single H1, no skipped levels), image alt text, structured data
(schema.org) validity, robots.txt + sitemap.xml reachability, internal linking basics, and
the mobile viewport tag. Score each area, flag P0 (blocks ranking/indexing) first.

Then **read `references/aeo-geo.md` now** and run the AI-visibility layer over the same
fetched HTML/DOM: classify every fetch before reading it, then bot-vs-user parity,
raw-vs-rendered parity, AI-crawler directives, schema-to-visible-content parity,
extractability, claim integrity, and thin-template detection. Rank on the same P0/P1/P2
scale. Ecommerce-only checks are marked **N/A with a reason** on non-ecommerce targets —
N/A is not a skipped phase.

## Phase 3 — Performance budget

**Read `references/perf-budget.md` now.** Measure Core Web Vitals on the most representative
route (LCP, CLS, INP; plus FCP, TTFB, TBT as context), then run the top-culprits checklist:
render-blocking resources, unoptimized images, font-loading strategy, and JS weight. Any CWV
threshold breach is a hard-gate High. Diagnose each breach to a suspected source.

## Consolidated report

One report. Lead with the verdict block, then findings ranked by severity.

```
============================================================
VERDICT: [Pass / Conditional Pass / Fail / Incomplete]

Persona: [locked persona]
Surfaces audited: N / M routes
Proof of interaction: [complete / missing] — [N] logged interactions

Hard gates: console errors [N], warnings [N], 5xx [N], 403/404 auth [N],
  layout-collapse [N], axe Critical [N], axe Serious [N]   (all must be 0)
Performance (/[route]): LCP [N]s / CLS [N] / INP [N]ms  — budget 2.5s / 0.1 / 200ms
SEO: [N] P0, [N] P1, [N] P2   (title/meta/canonical/headings/alt/schema/robots/sitemap/viewport)
AI visibility: [N] P0, [N] P1, [N] P2   (bot-parity/render-parity/ai-robots/schema-parity/extractability/claims/trust/templates)

Findings: Critical [n]  High [n]  Medium [n]  Low [n]

TOP 5 (ranked by impact x ease):
  1-5. [F-id] Title — one-sentence reason it edges out the rest
============================================================
```

### Every finding must include

**ID** (severity-letter + number), **Phase** (UX / SEO / AEO / Perf), **Severity**, **Surface**
(route + viewport), **Reproduce** (numbered steps), **Observed**, **Expected**, **Evidence**
(screenshot path / console line / network capture / measured metric), **Suspected location**
(`file:line` or resource URL), and a **concrete patch** (committable, not "consider X" /
"improve Y"). A finding without reproduction + evidence + suspected location is rejected.

Group the fix list into Quick Wins (24-48h), Structural (1-2 weeks), and Polish (post-launch).

## Audit-the-audit (final step, mandatory)

Before publishing, reject the report if any of these fire — flip the verdict to **Incomplete**:

| Signal | Implies |
|--------|---------|
| No typed/clicked/navigated interactions logged | UX phase was a static read, not a walkthrough |
| Interaction timestamps clustered < 0.5s apart | Log batch-emitted, no real interaction happened |
| Screenshots fewer than 2 x routes, or console reads fewer than 1 x route | Pages weren't actually checked |
| Perf phase reports CWV with no measurement method shown | Numbers invented, not measured |
| Findings say "Suggested fix" / "Consider X" / "Improve Y" | Filler-shaped, not committable |
| AEO phase asserts parity/claims findings with no fetch, diff, or quoted string shown | Verified by vibes, not by request |
| Missing-title / no-H1 / thin-content findings on a route whose fetch was blocked or non-HTML | Challenge HTML recorded as if it were the page |
| Top 5 missing or padded with filler | Discipline broken |

A clean "Pass" with implausible timings is not legal. If the meta-check fires, redo the
weak phase with real interaction and measurement before publishing.
