# site-audit

An agent skill that audits a **running** website or web app end to end and returns one
consolidated, severity-ranked report. It runs three phases in order — UX walkthrough,
on-page SEO, and Core Web Vitals performance — enforces hard quality gates, and rejects its
own rushed output.

This audits a live target. You need a reachable URL or a running local server
(e.g. `http://localhost:5173`). It is not a static code review.

Agent-neutral: works with Claude Code, Codex, or any skill host, using whatever browser
tooling is available — Chrome DevTools MCP, Playwright MCP, or plain `curl` + `lighthouse`
CLI as a fallback.

## Install

```bash
npx skills add badmuriss/site-audit
```

Then trigger it with prompts like "audit the site", "site audit", "qa sweep", "seo check",
"lighthouse", "check performance", or "is the site ready to ship".

## The three phases

1. **UX walkthrough** (`references/ux-walkthrough.md`) — walk the app as a locked persona:
   type into inputs, trigger primary actions, open panes, read the console, screenshot before
   and after, run axe-core per page, sweep responsive widths, and stress the edges. Produces a
   timestamped interaction log. No interaction = the UX verdict is "Incomplete".

2. **On-page SEO + AI visibility** (`references/seo-onpage.md`, `references/aeo-geo.md`) —
   title and meta quality + length, canonical, OG/Twitter cards, heading hierarchy, image alts,
   structured-data (schema.org) validity, robots.txt + sitemap reachability, internal linking,
   content depth, and the mobile viewport tag. Then the AI-visibility layer: fetch
   classification, bot-vs-user HTML parity, raw-vs-rendered parity, AI-crawler directives,
   schema-to-visible-content parity, answer extractability, claim integrity, trust signals, and
   thin-template detection. Each area scored, fixes ranked P0/P1/P2.

3. **Performance budget** (`references/perf-budget.md`) — measure Core Web Vitals on the most
   representative route, then work the top-culprits checklist: render-blocking resources,
   unoptimized images, font loading, JS weight.

## Hard gates (auto-fail, cannot be downgraded)

- Console errors = 0, console warnings = 0
- Network 5xx = 0, 403/404 on authenticated pages = 0
- Layout collapse at any tested viewport = 0
- axe-core Critical = 0, Serious = 0
- Core Web Vitals green: **LCP <= 2.5s, CLS <= 0.1, INP <= 200ms**
- Proof of interaction required — an audit that never typed, clicked, or navigated ends with
  verdict **Incomplete**

The final step is an audit-the-audit meta-check that flips the verdict to Incomplete if the
run looks rushed (no real interaction, clustered timestamps, too few screenshots, invented
metrics, filler "consider X" fixes, or content findings recorded against a blocked fetch).

## Testing this skill

`https://badseo.dev` is a live, public, no-auth regression target: 33 URLs in its sitemap, one
broken SEO rule per page, with self-describing paths. Point the skill at it and check that each
rule fires on the page named after it.

| Path | Exercises |
|------|-----------|
| `/head/missing-title`, `/head/heading-order-skip` | Title tag, heading hierarchy |
| `/content/duplicate-title-a`, `/content/duplicate-title-b` | Duplicate-group detection + its canonical exclusion rule |
| `/index/noindex-header`, `/index/canonical-conflict` | `X-Robots-Tag` and `Link:` header reading — the HTML alone does not reveal either |
| `/status/not-found`, `/status/server-error`, `/status/blocked` | Fetch classification. **`/status/blocked` is the important one** — it must yield exactly one "blocked" finding and must not cascade into missing-title / no-H1 / thin-content |
| `/links/broken-internal-link`, `/structure/orphan`, `/structure/no-outgoing-links` | Internal-linking checks and their suppression rules |
| `/redirect/trailing-slash` | Redirect-chain handling from the chain head |
| `/perf/slow-response` | TTFB / response-time diagnosis |
| `/kitchen-sink` | Everything at once |

A run that reports content findings against `/status/blocked` has the non-cascade rule wrong.

## License

MIT — see `LICENSE`. Distilled in part from `jezweb/claude-skills` (MIT),
`aaron-he-zhu/seo-geo-claude-skills` (Apache-2.0), and `cloudflare/skills` (Apache-2.0);
see `NOTICE.md` for attribution.
