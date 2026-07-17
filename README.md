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

2. **On-page SEO** (`references/seo-onpage.md`) — title and meta quality + length, canonical,
   OG/Twitter cards, heading hierarchy, image alts, structured-data (schema.org) validity,
   robots.txt + sitemap reachability, internal linking, and the mobile viewport tag. Each area
   scored, fixes ranked P0/P1/P2.

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
metrics, or filler "consider X" fixes).

## License

MIT — see `LICENSE`. Distilled in part from `jezweb/claude-skills` (MIT),
`aaron-he-zhu/seo-geo-claude-skills` (Apache-2.0), and `cloudflare/skills` (Apache-2.0);
see `NOTICE.md` for attribution.
