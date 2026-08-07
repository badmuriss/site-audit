<p align="center"><img src="docs/banner.png" width="720" alt="site audit wordmark on a near-black canvas, the word audit in signal amber, with the tagline: walk it, measure it, then decide if it ships"></p>

<p align="center"><b>Audit a running site the way a skeptical reviewer would, then refuse to pass it on vibes.</b></p>

<p align="center">
  <a href="LICENSE"><img src="https://img.shields.io/badge/license-MIT-green?style=flat-square" alt="MIT license"></a>
  <a href="https://skills.sh"><img src="https://img.shields.io/badge/agent-skill-black?style=flat-square" alt="agent skill"></a>
  <a href="https://github.com/badmuriss/site-audit/stargazers"><img src="https://img.shields.io/github/stars/badmuriss/site-audit?style=flat-square" alt="GitHub stars"></a>
  <a href="https://github.com/badmuriss/site-audit/commits/main"><img src="https://img.shields.io/github/last-commit/badmuriss/site-audit?style=flat-square" alt="last commit"></a>
</p>

<p align="center">
  <a href="#install">Install</a> ·
  <a href="#the-three-phases">The three phases</a> ·
  <a href="#hard-gates">Hard gates</a> ·
  <a href="#it-audits-its-own-audit">It audits its own audit</a> ·
  <a href="#pairs-with-incredibly-pretty-websites">Pairs with</a> ·
  <a href="#testing-this-skill">Testing</a>
</p>

## Install

```bash
npx skills add badmuriss/site-audit
```

Works with Claude Code, Codex, OpenCode, Cursor and anything else that reads the [skills](https://skills.sh) format. Manual alternative:

```bash
git clone https://github.com/badmuriss/site-audit ~/.agents/skills/site-audit
```

Then trigger it: "audit the site", "site audit", "qa sweep", "seo check", "lighthouse", "check performance", "is the site ready to ship".

**This audits a live target.** You need a reachable URL or a running local server (`http://localhost:5173` is fine). It is not a static code review, and it will stop and ask for a URL rather than read your source and guess.

## Why it exists

Ask any agent to "check my site" and you get a confident report it never earned: findings inferred from source files, Core Web Vitals quoted with no measurement behind them, a clean bill of health for a page nobody clicked. The failure is not the checklist, it is that nothing forces the agent to actually use the site before grading it.

So this skill is built around evidence rather than coverage. Every finding carries reproduction steps, a captured artifact and a suspected location. Every phase names the tool it measured with. And the last step is a meta-check that throws out the report when the run looks faked.

## The three phases

One workflow, three phases in order, one consolidated report at the end. Each phase reads its own reference file, does the work, and captures evidence before the next one starts.

| # | Phase | Reference | What it does |
|---|---|---|---|
| 1 | **UX walkthrough** | [`ux-walkthrough.md`](references/ux-walkthrough.md) | Locks a persona, inventories the routes, then actually uses the app: types into inputs, fires primary actions, opens panes, reads the console, screenshots before and after, runs axe-core per page, sweeps responsive widths, stresses the edges. Produces a timestamped interaction log. |
| 2 | **On-page SEO** | [`seo-onpage.md`](references/seo-onpage.md) | Title and meta quality plus length, canonical (markup **and** the HTTP header), OG and Twitter cards, heading hierarchy, image alts, structured-data validity, robots and sitemap reachability, internal linking, content depth, mobile viewport. Each area scored, fixes ranked P0/P1/P2. |
| 2 | **AI visibility (AEO/GEO)** | [`aeo-geo.md`](references/aeo-geo.md) | The same HTML through a second lens: fetch classification, bot-vs-user parity, raw-vs-rendered parity, AI-crawler directives, schema-to-visible-content parity, extractability, claim integrity, trust signals, thin-template detection. |
| 3 | **Performance budget** | [`perf-budget.md`](references/perf-budget.md) | Measures Core Web Vitals on the most representative route, then works the top-culprits checklist: render-blocking resources, unoptimized images, font loading, JS weight. Every breach gets diagnosed to a suspected source. |

### Deep dives, read on demand

Five references the phases pull in only when the situation calls for them. Nothing here loads by default, so a straightforward audit stays cheap.

| Reference | Read it when |
|---|---|
| [`stress-test-recipes.md`](references/stress-test-recipes.md) | A row in the Phase 1 stress table fires and you need the actual recipe: steps, what to watch, what counts as a failure |
| [`data-seasoning.md`](references/data-seasoning.md) | The app stores or renders user text. Full real-flavour battery: accents, apostrophes, RTL, emoji, zero-width, XSS canary, oversized files |
| [`multi-pane-stress.md`](references/multi-pane-stress.md) | More than one pane, tab or live region shares the screen. Cross-pane sync bugs never surface in a single-pane walkthrough |
| [`perfection-checklist.md`](references/perfection-checklist.md) | Layout holds at every width and you want the per-component pass on states, spacing, focus, loading and empty variants |
| [`project-adaptation.md`](references/project-adaptation.md) | The defaults do not fit: you cannot find the URL, cannot log in headless, need seed data, or the repo has its own path convention. Covers Next/Auth.js, Rails, Django, Laravel, WordPress, Prisma, Drizzle, raw SQL |

### Bring whatever browser tooling you have

Agent-neutral by design. Any tier is valid, and the skill tells you what the tier costs you:

| Tier | Tools | Trade-off |
|---|---|---|
| Best | Chrome DevTools MCP or Playwright MCP | Real interaction, console and network capture, traces |
| Good | `playwright` / `puppeteer` driven from a script | Same coverage, more manual |
| Fallback | `curl` + `lighthouse` CLI + `npx axe` | No live interaction, so the UX phase caps at "Incomplete". SEO and perf still run in full. |

## Hard gates

These cannot be downgraded, argued with, or averaged away. One red gate fails the audit.

| Gate | Threshold |
|---|---|
| Console errors during the walkthrough | 0 |
| Console warnings during the walkthrough | 0 |
| Network 5xx | 0 |
| Network 403/404 on authenticated pages | 0 |
| Layout collapse at any tested viewport | 0 |
| axe-core Critical / Serious violations | 0 |
| LCP / CLS / INP on the representative route | ≤ 2.5s / ≤ 0.1 / ≤ 200ms |
| Proof of interaction | required |

There is no such thing as a "Medium console error" here, that category does not exist. A 5xx is Critical automatically. If your target has genuinely noisy categories (Sentry info logs, extension chatter, an expected 401 on an auth probe), you declare them as an allowlist up front and the verdict block shows both raw and reportable counts.

### Four verdicts

| Verdict | Meaning |
|---|---|
| **Pass** | Critical 0, High 0, every hard gate green, proof of interaction present |
| **Conditional Pass** | Same, but Medium/Low findings remain |
| **Fail** | At least one Critical or High, or a red hard gate |
| **Incomplete** | No proof of interaction, a skipped phase, or the meta-check fired |

"It looked OK" is never a Pass.

## It audits its own audit

The last step before publishing is a meta-check that flips the verdict to **Incomplete** when the run itself looks unearned. It is the part that makes the rest trustworthy:

- No typed, clicked or navigated interactions logged, so the UX phase was a static read
- Interaction timestamps clustered under 0.5s apart, so the log was batch-emitted after the fact
- Fewer than 2 screenshots per route, or fewer than 1 console read per route
- Core Web Vitals reported with no measurement method shown, so the numbers were invented
- Findings phrased as "Suggested fix", "Consider X", "Improve Y", so nothing is committable
- AEO parity or claim findings with no fetch, diff or quoted string behind them
- A missing-title or thin-content finding on a route whose fetch was actually blocked, so a WAF challenge page got recorded as the page

A clean "Pass" with implausible timings is not legal. The weak phase gets redone with real interaction before anything ships.

### Every finding, or it does not count

ID, phase, severity, surface (route plus viewport), numbered reproduction steps, observed, expected, evidence (screenshot path, console line, network capture, measured metric), suspected location (`file:line` or resource URL), and a concrete committable patch. A finding missing reproduction, evidence or a suspected location is rejected before it reaches the report. The fix list closes grouped into Quick Wins (24 to 48h), Structural (1 to 2 weeks) and Polish (post-launch).

## Pairs with incredibly-pretty-websites

[**incredibly-pretty-websites**](https://github.com/badmuriss/incredibly-pretty-websites) builds. This skill verifies what got built.

```bash
npx skills add badmuriss/incredibly-pretty-websites
```

Neither one carries the other's rules, which is the point:

| | incredibly-pretty-websites | site-audit |
|---|---|---|
| Runs on | a brief, a repo, a blank page | a reachable URL or a local dev server |
| Owns | typography, color, layout, motion, components, copy tone, the AI-tells list | on-page SEO, AEO/GEO, axe-core, Core Web Vitals, the UX walkthrough |
| Output | a built site | one severity-ranked report with hard gates |

Build, deploy, then point this at the URL. Redesigning an existing site instead? Run this **first** too: the report is the baseline the redesign has to protect, and the after-run is what proves it did.

## Testing this skill

[`badseo.dev`](https://badseo.dev) is a live, public, no-auth regression target: 33 URLs in its sitemap, one broken rule per page, self-describing paths. Point the skill at it and check that each rule fires on the page named after it.

| Path | Exercises |
|---|---|
| `/head/missing-title`, `/head/heading-order-skip` | Title tag, heading hierarchy |
| `/content/duplicate-title-a`, `/content/duplicate-title-b` | Duplicate-group detection and its canonical exclusion rule |
| `/index/noindex-header`, `/index/canonical-conflict` | `X-Robots-Tag` and `Link:` header reading, which the HTML alone does not reveal |
| `/status/not-found`, `/status/server-error`, `/status/blocked` | Fetch classification. **`/status/blocked` is the important one**: exactly one "blocked" finding, and no cascade into missing-title, no-H1 or thin-content |
| `/links/broken-internal-link`, `/structure/orphan`, `/structure/no-outgoing-links` | Internal-linking checks and their suppression rules |
| `/redirect/trailing-slash` | Redirect-chain handling from the chain head |
| `/perf/slow-response` | TTFB and response-time diagnosis |
| `/kitchen-sink` | Everything at once |

A run that reports content findings against `/status/blocked` has the non-cascade rule wrong.

## Credits

Distilled and reworked, not copied, from three existing skills. Full attribution in [`NOTICE.md`](NOTICE.md):

- UX walkthrough phase, from [`jezweb/claude-skills`](https://github.com/jezweb/claude-skills) (MIT)
- On-page SEO phase, from [`aaron-he-zhu/seo-geo-claude-skills`](https://github.com/aaron-he-zhu/seo-geo-claude-skills) (Apache-2.0)
- Performance budget phase, from [`cloudflare/skills`](https://github.com/cloudflare/skills) (Apache-2.0), with the Cloudflare-product-specific guidance deliberately dropped

## License

MIT. See [`LICENSE`](LICENSE).

Built by [Murilo Moura](https://github.com/badmuriss).
