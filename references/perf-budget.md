# Performance Budget (Phase 3)

Measure Core Web Vitals on the **most representative route** (the page real users hit most —
usually the dashboard, main list, or the landing page), then diagnose any breach to a source.
State your measurement method in the report; numbers with no method shown are rejected by the
audit-the-audit check.

Your knowledge of specific thresholds/APIs may be stale — prefer current docs (web.dev/vitals,
Lighthouse scoring) when citing exact numbers.

## Thresholds (hard gates)

Core Web Vitals "Good" tier. Any breach is a hard-gate **High**.

| Metric | Good (gate) | Needs work | Poor |
|--------|-------------|------------|------|
| LCP — Largest Contentful Paint | <= 2.5s | <= 4.0s | > 4.0s |
| CLS — Cumulative Layout Shift | <= 0.1 | <= 0.25 | > 0.25 |
| INP — Interaction to Next Paint | <= 200ms | <= 500ms | > 500ms |

Context metrics (report but don't hard-gate): FCP (< 1.8s good), TTFB (< 800ms good),
TBT (< 200ms good), Speed Index (< 3.4s good).

> For app *interiors* real users sit inside (internal tools, dashboards behind auth), a
> pragmatic floor of LCP < 4.0s / CLS < 0.25 / INP < 500ms is defensible if the user says CWV
> isn't a priority — but the default gate is the "Good" tier above. Landing pages, signup
> funnels, and e-commerce checkout always use the strict gate.

## Measurement — pick what the host has

**Chrome DevTools MCP** (best): `navigate_page(url)`, then
`performance_start_trace(autoStop: true, reload: true)` to capture a cold load with built-in
throttling, then `performance_analyze_insight` for `LCPBreakdown`, `CLSCulprits`,
`RenderBlocking`, `DocumentLatency`, `NetworkRequestsDepGraph`. Insight names vary by version —
if one 404s, inspect the trace response for available insights.

**Playwright / browser eval**: capture inline via PerformanceObserver (~1s, no Lighthouse
per-page overhead). Runs unthrottled in headless — LCP/INP paint optimistically, so document
it; a page at LCP=3.5s unthrottled is likely 6s+ on real 4G.

```js
await page.evaluate(() => new Promise((resolve) => {
  let lcp = 0, cls = 0, inp = 0
  new PerformanceObserver(l => { for (const e of l.getEntries()) lcp = e.startTime })
    .observe({ type: 'largest-contentful-paint', buffered: true })
  new PerformanceObserver(l => { for (const e of l.getEntries()) if (!e.hadRecentInput) cls += e.value })
    .observe({ type: 'layout-shift', buffered: true })
  new PerformanceObserver(l => { for (const e of l.getEntries()) inp = Math.max(inp, e.duration) })
    .observe({ type: 'event', buffered: true, durationThreshold: 16 })
  setTimeout(() => {
    const nav = performance.getEntriesByType('navigation')[0]
    resolve({ lcp: Math.round(lcp), cls: Math.round(cls * 1000) / 1000, inp: Math.round(inp),
      ttfb: nav ? Math.round(nav.responseStart - nav.requestStart) : null,
      loadComplete: nav ? Math.round(nav.loadEventEnd) : null })
  }, 1500) // let LCP + CLS settle
}))
```

INP only fires after a real interaction — trigger a click/type first, then read the worst seen.

**Fallback — Lighthouse CLI** (no MCP, no browser driver):

```bash
npx lighthouse https://site.com --only-categories=performance \
  --output=json --output-path=/tmp/lh.json --chrome-flags="--headless"
# read metrics: jq '.audits["largest-contentful-paint"].numericValue' /tmp/lh.json
```

Lighthouse applies mobile throttling by default (closer to real-world than raw
PerformanceObserver). Slower (~30-60s) but a solid single-route number.

## Top culprits checklist

For each breach, find the cause — be specific ("compress hero.png 450KB to WebP", not
"optimize images"). Skip anything with ~0ms estimated impact; a site at LCP 200ms / CLS 0 is
already excellent, say so.

| Symptom | Likely cause | Check |
|---------|--------------|-------|
| **LCP > 2.5s** | Unoptimized hero image; render-blocking CSS/JS; synchronous font; slow TTFB | Is the LCP element an image? Its weight + format? Is it preloaded? Blocking resources in `<head>`? |
| **CLS > 0.1** | Images/embeds without width+height; late font swap; content injected above the fold; ads with no reserved space | Which element shifts (CLSCulprits)? Do media have explicit dimensions? `font-display`? |
| **INP > 200ms** | Heavy click/keystroke handlers re-rendering; long tasks > 50ms; sync JSON parse; hydration cost | Interact, watch for long tasks in the trace. What runs on the interaction? |
| Slow to interactive | JS bundle too large; too many sync imports; SSR on every nav | Total JS transferred? Any single chunk > 200KB gzipped? |

### The four weight offenders (audit these directly)

1. **Render-blocking** — JS/CSS in `<head>` without `async`/`defer`/`media`. List requests
   (`list_network_requests(resourceTypes:["Script","Stylesheet"])` or the Network tab) and
   flag blocking resources on the critical path. Missing preloads for critical fonts/hero/key
   scripts count here too.
2. **Unoptimized images** — oversized or legacy-format images, especially the LCP element and
   anything above the fold. Recommend WebP/AVIF, correct dimensions, and `loading="lazy"`
   below the fold. Verify before recommending removal of anything.
3. **Font loading** — synchronous/blocking web fonts cause both LCP delay and CLS (swap). Want
   `font-display: swap` (or `optional`), preloaded/self-hosted critical fonts, and a matched
   fallback to minimize the swap shift.
4. **JS weight** — large or unused bundles. Look for whole-library imports (lodash, moment),
   barrel-file re-exports pulling everything, missing tree-shaking/minification, and eager
   loading where a dynamic import would defer. Confirm code is actually unused before flagging.

## Output

A CWV summary table (metric / value / rating + measurement method), then the prioritized
fixes with estimated impact (high/medium/low) and concrete code/config changes. Every breach
becomes a finding in the consolidated report with its suspected source resource.
