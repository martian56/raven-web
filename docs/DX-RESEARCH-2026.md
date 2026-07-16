# What frontend developers expect in 2026, and where raven-web stands

Written 2026-07-16, against raven-web v0.13.1. This is the answer to one
question: is raven-web offering everything a frontend developer can ask for,
and if not, what is missing and in what order should it be built?

Short answer: no, not yet. raven-web has an unusually strong interactivity
and safety story for its class, and a genuinely differentiated toolchain
story, but it is missing most of the content pipeline that defines the static
site generator category it lives in, and several production-output table
stakes. The gaps split cleanly into reachable and structural, and the
reachable list is long enough to fill several releases.

## 1. Method and sources

Two investigations, merged:

- External: developer surveys (State of JS 2025, State of HTML 2025, Stack
  Overflow 2025), framework documentation and credible comparisons for the
  build-time class (Astro, Eleventy, Hugo) and the lightweight-interactivity
  class (Alpine.js, htmx), plus practitioner postmortems on both classes.
  Key claims were adversarially verified against the primary sources; the
  survey numbers cited below were confirmed against the survey pages
  directly.
- Internal: a code audit of lib.rv and dev.rv at v0.13.1, checking every
  suspected gap against the source rather than memory.

## 2. The market raven-web competes in

raven-web is one tool spanning two established categories: build-time HTML
generation (the Astro, Eleventy, Hugo class) and attribute-driven client
interactivity (the Alpine.js, htmx class). The evidence says this combined
position is a good one to be in:

- Astro holds the top meta-framework satisfaction ranking in State of JS
  2025, leading Next.js by 39 points, and the survey's number one
  meta-framework pain point is "excessive complexity" (303 mentions).
  Developers are actively moving toward simpler, content-first, build-time
  tools. (Verified against 2025.stateofjs.com directly.)
- Practitioners report the hidden costs of the incumbent: a clean Astro
  install is roughly 100 MB of node_modules, and its dev mode injects about
  1.75 MB of JavaScript into every page. raven-web is one native binary, no
  Node, and its whole production payload is smaller than 4 KB gzipped.
- One comparison put it plainly: "build speed is the single most reliable
  predictor of developer satisfaction at scale." Hugo builds 10,000 pages in
  under a second; Eleventy takes 14 s, Astro 31 s. raven-web generates its
  own site in milliseconds and its bottleneck is the Raven compile (about
  2 s), which only dev mode pays.
- The Alpine/htmx class caps out exactly where raven-web does: local UI
  state, forms, toggles, filtering yes; complex client state, offline,
  drag-and-drop, rich text no. That boundary is a defensible category line,
  not a raven-web defect. htmx is about 14 KB, Alpine about 15 KB; raven-web
  ships 3.9 KB including the page's own data.
- State of HTML 2025 says the single biggest developer pain is styling and
  customizing form controls (987 of about 2,041 pain mentions), with form
  validation second (527). raven-web's typed controlled fields and Expr
  validation sit directly on the category's sorest spot.

## 3. Scorecard

"Table stakes" below means: the leading tools in the class ship it first
party or via a blessed plugin, and comparisons treat its absence as a reason
to pick another tool.

| Capability | Expectation | raven-web v0.13.1 |
|---|---|---|
| Typed, componentized templating | table stakes | yes: Node builders, Component, Ctx instance state; compiler-checked props |
| Client interactivity (toggles, tabs, modals-lite) | table stakes | yes: signals, behaviors, Expr; targeted updates, keyed lists |
| Forms and validation | table stakes | yes: bind_value, Expr rules, disabled_when, aria states |
| Client router / SPA option | table stakes for app-ish sites | yes: patterns, params, guards, history, 404 fallback |
| Route query parameters | table stakes | no: router reads pathname only |
| Styling (authored CSS, tokens, framework-agnostic) | table stakes | yes: Stylesheet, CssRule, Theme light/dark, any CSS file |
| Scoped/component styles | nice-to-have | no |
| SEO head (meta, OG, canonical, JSON-LD) | table stakes | yes as of v0.13.0 |
| Sitemap | table stakes | partial: loc only, no lastmod/changefreq/priority |
| RSS/Atom feed | table stakes for content sites | no |
| hreflang alternates | table stakes for multilingual sites | no |
| Markdown content pipeline | the defining feature of the SSG class | no |
| Frontmatter + content collections | table stakes in class | no |
| Syntax highlighting | table stakes for dev-facing content | no |
| Pagination / taxonomies | table stakes in class | no |
| CMS/data-source integration | table stakes in class | partial: any Raven code can fetch at build time, but no helpers or recipes |
| Image discipline (dimensions, lazy, srcset) | table stakes | no: Node.image is src+alt only |
| Image resizing/optimization | expected first party (Astro, Hugo) | no, and binary processing needs shelling out |
| Asset fingerprinting / cache busting | table stakes for production | no: app.js and styles.css are unversioned |
| CSS/HTML minification | table stakes | no: authored CSS ships as written |
| Base-path deployment (subpath hosting) | table stakes | no: absolute asset paths assume root |
| i18n | table stakes in class (Hugo first party) | no helper; hand-rolled per-language pages work (proven on fuadalizada.com) |
| Accessibility primitives | table stakes | yes: aria_if/aria_when, id_for, router focus, visible focus |
| Dialog/focus management | expected | no focus trap; details/summary only |
| Dev server + live reload + error surface | table stakes (Vite at 98% satisfaction set the bar) | yes: cli dev, recompile on save, browser error overlay |
| Scaffolding (create/new) | table stakes | no; copy examples/starter.rv |
| Testing story for site authors | expected | partial: html()/warnings() are assertable, but no guide or helpers |
| Performance by default | table stakes | yes: fixed 3.9 KB runtime, build-time dep graph, no hydration |
| Transitions/animations API | nice-to-have | no (gap analysis Tier 4.11); CSS-only works today |
| Keyboard specifics (key filters), debounce, timers | expected for polish | no: keydown fires for any key, no debounce, no intervals |
| WebSocket/SSE/polling refresh | nice-to-have (htmx has extensions) | no; fetch on events only |
| Per-row stateful components, arbitrary client logic | expected of app frameworks only | structural no (Option B territory) |
| SSR / hydration / islands with real code | expected of app frameworks only | structural no |
| Plugin ecosystem, themes | treated as table stakes by comparisons | no ecosystem; single library |

## 4. What raven-web already does that the class leaders do not

Worth naming, because the roadmap should protect these:

- Typo-proof wiring. A binding references a Signal value the compiler
  checks. Alpine and htmx are stringly-typed attributes; the class's own
  postmortems complain about untraceable hx-* webs at scale.
- The safety model. No element, attribute, or URL can execute, enforced at
  the same depth on the escape hatches as on the typed paths, with build
  warnings and strict mode. No tool in either class does this.
- The toolchain weight. One binary against 100 MB of node_modules is a real
  differentiator the incumbents' own users complain about.
- Honesty as a feature. The gap analysis and the README state the ceiling
  plainly. Keep doing that; the surveys say complexity fatigue and
  overpromising are what developers are fleeing.

## 5. The roadmap, prioritized

### P0: production output table stakes (small, high leverage)

1. Asset fingerprinting: content-hash app.js and styles.css into their
   filenames and rewrite references, so far-future cache headers are safe.
   Without this every deploy risks stale-asset bugs; every tool in the class
   solves it (Hugo calls it fingerprinting, first party).
2. Minify emitted CSS and HTML (whitespace and comments; no risky
   transforms). The runtime JS is already compact.
3. Base-path support: a Site/Page setting that prefixes emitted asset and
   link paths, so GitHub Pages style subpath deployments work.
4. Image discipline: Node.image gains width/height (CLS), loading=lazy by
   default with an opt-out, decoding=async, and a srcset/sizes builder for
   caller-provided variants. Actual image resizing stays out of scope until
   a shell-out story is designed; emitting correct markup does not need it.
5. Router query parameters: expose location.search the way params are
   exposed today. Small runtime addition, closes a visible hole.
6. Sitemap lastmod (author-supplied per route) and hreflang alternate links
   on Page/Site for multilingual sites. Both were wanted while building
   fuadalizada.com.

### P1: the content pipeline (the biggest absence; the class-defining work)

7. Markdown module: CommonMark to Node tree, rendered through the existing
   safe builders so the safety model holds for authored content. This is
   pure string processing, well inside Raven's reach, and it is the single
   feature whose absence keeps raven-web out of the Astro/Eleventy/Hugo
   conversation entirely.
8. Frontmatter parsing and content collections: walk a directory, parse
   frontmatter into typed values, hand back an ordered collection; feed
   Site pages from it. With Layout and Site already in place, this makes
   blogs and docs sites first-class.
9. RSS/Atom feed generation from a collection.
10. Syntax highlighting for code blocks at build time (classed spans plus a
    theme stylesheet, no client JS), since dev-facing content is the likely
    first audience.
11. Pagination helpers over collections.
12. i18n: a typed translations pattern (the Copy-struct approach proven on
    fuadalizada.com) promoted into a documented helper, plus per-locale
    routes and the hreflang wiring from P0.

### P2: interactivity polish, inside the Option A ceiling

13. Key-filtered keyboard events (on_key("Escape", ...)) and modifier
    support.
14. Debounced input behaviors (search-as-you-type without spamming) and
    interval/timer effects (carousel advance, clock), as serialized data
    like every other effect.
15. Dialog support: focus trap plus Escape-to-close as a typed behavior
    pair, building on the native dialog element where possible (52.8% of
    State of HTML respondents already use dialog).
16. Outside-click dismissal for menus.
17. Transitions API (gap analysis Tier 4.11): class-based enter/leave
    transitions on visibility bindings, honoring prefers-reduced-motion.
18. A polling/SSE refresh effect for live-ish data, mirroring htmx's
    extension surface without new authored JS.

### P3: developer experience and ecosystem substitutes

19. A scaffolder story: since rvpm has no template command, ship
    "copy this directory" starters (blog, docs, portfolio, landing) in the
    repo and reference them from the README.
20. A testing guide for site authors: golden-HTML assertions over
    page.html(), warnings() as a lint gate, strict() in CI.
21. A patterns/cookbook page (component recipes: accordion, tabs, modal,
    nav, cards) as the ecosystem substitute, since comparisons treat theme
    and component availability as table stakes and a one-library project
    cannot grow a plugin ecosystem quickly.
22. Dev-server QoL: port-in-use retry, and a --port flag on the dev command.

## 6. What stays structurally out of reach, restated

Per-row stateful components over runtime data, arbitrary client logic,
shared server/client domain code, SSR and hydration. These need Raven code
in the browser (Option B), which the user has rejected. The external
evidence makes this an easier position to hold than it sounds: the
Alpine/htmx class draws its boundary in the same place, its practitioners
ship real products inside it, and the complaints that push people out of it
(dashboards, collaborative editors, offline) describe apps raven-web should
not claim. The right response to those workloads is the honest one already
in the README: use an app framework.

## 7. Verdict

Is raven-web offering everything a frontend developer can ask for? No.
Today it is a typed, safe, unusually well-tooled Alpine-class interactivity
layer attached to a static site generator that is missing the content half
of its job. Nothing on the P0 or P1 list is architecturally hard; they are
mostly string processing and output discipline, and each one closes a gap
that comparisons in this class treat as disqualifying. The strongest order
of attack is P0 first (small, unblocks production credibility), then the
markdown/content pipeline (largest single absence), then interactivity
polish. The differentiators worth protecting along the way: compiler-checked
wiring, the safety model, the single-binary toolchain, and the honest
ceiling.
