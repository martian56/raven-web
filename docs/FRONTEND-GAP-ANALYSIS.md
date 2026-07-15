# What raven-web needs to be a real choice for frontend apps

An honest assessment, 2026-07-14, written against raven-web v0.2.2 by the person
who built it. The goal set for it is "replace React": best DX, best performance,
professional designs, complex apps, reusable components.

## Progress against this document

Option A (grow the library, no compiler change) is the chosen path. Status:

- **Tier 2.4/2.5 (destructive global re-render): fixed in v0.3.0.** The
  dependency graph is resolved at build time, so a write updates only the
  bindings that read that key, and lists reconcile by key and update rows in
  place. Verified in a browser: an input inside a row keeps its value, focus,
  and DOM identity across both an unrelated update and a full list refetch.
- **Tier 3.6 (stringly-typed wiring): largely fixed in v0.4.0.** `signal()`
  declares state once; nodes declare their own bindings (`text_of`,
  `visible_if`, `each_of`, ...) and raven-web marks the element, so no selector
  is written by hand. Behaviors and requests take the handle too. Remaining
  strings are the ones raven-web genuinely cannot check: `Signal.at(field)` and
  `{{field}}` both address JSON whose shape comes from a server.
- **Tier 1.2 (no logic): partly fixed in v0.5.0.** `Expr` gives conditionals,
  comparisons, boolean logic, arithmetic, `len`/`is_empty`, `concat`, and
  `ternary`, so an empty state and singular/plural text are now expressible. It
  is a DSL, not Raven: no arbitrary functions, no sharing domain logic with the
  server. That limit is structural and only Option B removes it.
- **Tier 1.1/1.3 (client components): partly fixed in v0.6.0.** A component is a
  Raven function whose props are typed parameters, and `Ctx.local` gives each
  instance private state, so a component is reusable, nestable, and independent
  of its siblings. Verified by driving the emitted page in a real DOM: three
  accordions, four counters, and two cards each held their own state under
  clicks. What remains unfixed is the dynamic half: instances are allocated
  while the tree is built, so a stateful component still cannot be repeated per
  row of a runtime-fetched list, and rows stay `{{field}}` templates. That half
  needs Option B.
- **Tier 4.8 (no client router): fixed in v0.7.0.** `Router` gives patterns,
  `:param` captures read as signals, guards, a fallback, real links, and
  history, with no reload, so state survives navigation. `spa()` and the dev
  server both serve `404.html` for an unowned path, so a deep link resolves in
  dev exactly as on a static host. Verified by driving the emitted app in a real
  DOM at a real URL: deep links, param re-binding, back, guard redirects
  (including on a direct deep link), and a ctrl-click left to the browser.
- **Tier 4.9 (no forms story): fixed in v0.8.0.** `bind_value` makes a field
  controlled, so validation is an `Expr` over the field's own signal, errors are
  `visible_when` rules, `disabled_when` gates submit, and dirty tracking is a
  signal set on blur. `Expr.matches` adds patterns. Verified by typing into the
  emitted form in a real DOM: errors stayed quiet until blur then cleared live,
  the email rule accepted `a@b.co` and rejected four near-misses, cross-field
  rules held, and unchecking a box re-disabled submit.

- **Tier 3.7 (`Page.state` takes only a String): fixed in v0.9.0.** `signal_of`
  seeds a signal from a typed Raven value via `@derive(ToJson)`, so the domain
  crosses into the client as structured data and renders on first paint with no
  request. Verified in a real DOM with `fetch`/`XHR` stubbed to fail: three rows
  rendered, zero requests made, and the state was a real array.
- **A security defect found and fixed in v0.9.0, not in the original list.** The
  README claimed the public API never accepts a raw JS string. It did:
  `Node.new("script").child_text(js)` emitted a live `<script>` that executed
  (confirmed in a DOM), and the raw `attr()` escape hatch skipped the URL
  checking that `link()`/`image()` do, so `attr("href", "javascript:...")`,
  `action`, `formaction`, `<base href>`, `<meta http-equiv=refresh>`, and
  `<iframe srcdoc>` all went out verbatim. Executing elements and attributes are
  now refused subtree-and-all, and every URL is judged by scheme wherever it is
  set. The lesson worth keeping: the typed constructors were safe and the escape
  hatch beside them was not, so the safety claim was only ever true of the path
  the tests took.

**Where that leaves the goal.** Every Tier 2, 3, and 4 item that Option A can
reach is now done, and Tier 1 is half done: components are reusable, typed, and
stateful, but only where the tree is known at build time. What is left is the
dynamic half of Tier 1 (a stateful component per row of runtime data, arbitrary
logic, sharing domain code with the server), and that is the part this
architecture cannot reach. It needs Option B.

## 1. What raven-web actually is today

Not a React alternative. Today it is:

- A **build-time static site generator**. Raven components are structs that
  render to HTML strings at build time. Nothing of the component survives into
  the browser.
- Plus a **12 KB fixed JavaScript interpreter** that reads serialized data
  (behaviors, bindings, state) out of `data-rw-*` attributes and a config blob.

The honest comparison is **Astro (build-time rendering) + Alpine.js (declarative
sprinkles)**, not React. That is a real and useful category. It is simply not the
category the goal names.

## 2. The load-bearing constraint

Everything below follows from one fact:

**Raven code cannot run in a browser.** `src/codegen` targets `Triple::host()`
and emits native objects through `cranelift-object`. Cranelift is a wasm
*consumer*, not a wasm emitter. There is no JavaScript backend and no wasm entry
in the v2 roadmap.

So every piece of client behavior must be expressible as **data interpreted by a
fixed runtime**. That is the ceiling. It is not a missing feature; it is the
architecture. React's model (your code runs in the browser, components hold
state, the framework re-renders) is unreachable from here by adding effects.

## 3. Gaps, ordered by how badly they hurt

### Tier 1: architectural blockers (cannot be fixed by adding opcodes)

1. **No client components.** A `Component` is build-time only. A task card
   cannot hold local state (inline editing, an open/closed menu, a hover
   popover), cannot have lifecycle, cannot own logic. React's central primitive
   does not exist. "Reusable components" today means "reusable HTML templates".
2. **No arbitrary logic.** A `Behavior` is a linear list of effects. There is no
   if/else, no loop, no expression, no computed or derived value, no formatting.
   You cannot express "if the list is empty show an empty state", "show
   `3 tasks` vs `1 task`", "disable the button while the field is blank", or any
   validation rule. Every new need becomes a new opcode: that is reinventing a
   programming language inside JSON, badly.
3. **No composition on the client.** `bind_list` templates are flat and
   stringly-typed (`{{field}}`). No nesting, no per-item components, no slots, no
   props, no children. A board column can render a row; it cannot render a row
   that itself renders a component.

### Tier 2: correctness and performance

4. **Re-render is global and destructive.** `rw_apply()` re-runs *every* binding
   on *every* state change, and each list binding does `innerHTML = ''` and
   rebuilds all rows. Consequences:
   - An input inside a list loses focus and caret on any unrelated state change.
   - Scroll position, text selection, CSS transitions, and `<details>` open state
     are destroyed on every update.
   - Cost is O(all bindings x all rows) per keystroke, not O(what changed).
   This alone makes editable lists, live-filtered tables, and most real app UI
   infeasible. React reconciles; Solid/Svelte update the exact node. raven-web
   nukes the subtree.
5. **No keyed reconciliation.** Rows have no stable identity, so list animations,
   drag-and-drop, and per-row state are impossible.

### Tier 3: the typing paradox (the DX pitch is the weakest part)

6. **The wiring is stringly-typed.** `bind_text("#org-id", "auth.org_id")`,
   `{{title}}`, `.field("email", "#email")`, `bind_list_where(..., "status",
   "todo")`. A typo in a selector, a state path, or a placeholder is a **silent
   no-op**: exactly the failure class Raven's type system exists to eliminate. I
   hit this myself during work-prj (a wrong asset path silently served no CSS,
   and a mis-set binding silently showed two cards at once). The "typed" claim
   covers the *builder* API, not the *wiring*, which is where the bugs live.
   React with TypeScript type-checks props from parent to child; raven-web checks
   nothing across the binding boundary.
7. **`Page.state(key, value)` takes only a `String`.** Structured initial state
   is impossible; arrays can only enter via `fetch`. The typed domain in
   `packages/core` cannot cross into the client as typed data.

### Tier 4: missing app infrastructure

8. **No client router.** Multi-page means full page loads. No nested routes, route
   params, guards, or transitions.
9. **No forms story.** No validation, error display, dirty tracking, or
   controlled inputs.
10. **No SSR or hydration.** Static only. No dynamic server rendering, no islands
    with a hydration boundary.
11. **No transitions or animation primitives.**
12. **No component library, no preview/storybook, no devtools**, no way to test a
    component's behavior (only its rendered HTML string).
13. **No code splitting or lazy loading.**

### What is actually good today

The design layer is the strongest part and worth keeping: `Theme` design tokens
with light/dark, the `Stylesheet`/`CssRule` builders, the safety model (validated
tags/attrs/URLs, escaping, no raw JS accepted, template data injected via
`textContent`/`setAttribute` and never `innerHTML`), the static output, the dev
server with live reload, and `Site`/`Layout` multi-page. "Professional designs"
is the one goal that is largely reachable today.

## 4. The strategic fork: how does Raven reach the browser?

There are only three coherent answers.

### Option A: stay a data-driven VM and grow it

Add opcodes for conditionals, computed values, components, and so on.

- Ceiling: Alpine/HTMX class. Good for content sites plus light interactivity.
- Cost of each step is low; the total is a bad language expressed as JSON.
- Will **never** be a React alternative. Tier 1 stays unsolved by construction.
- Honest framing: this makes raven-web the best *static site* tool, not the best
  *app* tool.

### Option B: a Raven to JavaScript backend (recommended if the goal stands)

A new codegen backend that emits JavaScript from HIR/MIR, the way Svelte, Gleam,
ReScript, and Elm reach the browser.

- Unlocks Tier 1 completely: real Raven components with local state and real
  logic run in the browser, type-checked end to end.
- Enables Tier 2 properly: build a signals/fine-grained reactivity layer, so an
  update touches one text node instead of wiping a list.
- Kills Tier 3: props and state become typed Raven values, not selector strings.
- Cost: this is a **compiler project**, not a library project. A JS backend for a
  GC'd language means mapping Raven's value model, closures, traits/generics
  (monomorphized or dictionary-passed), and `Result`/`Option` onto JS, plus a
  runtime shim. Then the framework (components, signals, router) on top.
- This is the only path where "replace React" is a truthful claim.

### Option C: Raven to WebAssembly

- Cranelift cannot emit wasm, so this needs a second backend (LLVM or binaryen).
- Raven is garbage-collected, so it needs wasm-gc or must ship a GC in the
  payload; either way the bundle is far larger than JS output.
- Every DOM call crosses the wasm/JS boundary, which is slower than JS for
  DOM-heavy work. This is why Leptos/Dioxus (Rust, no GC) accept it and most
  GC'd languages do not.
- More work than Option B for a worse frontend result.

## 5. Recommendation

Pick the product, then the work follows:

**If the goal is "the best way to build content-heavy sites in Raven"** (a real,
winnable niche): take Option A. The highest-value work is then, in order:
fine-grained updates instead of the global re-apply (fix Tier 2.4), typed
selector/state handles to kill the silent-no-op class (Tier 3.6), a conditional
and computed-value opcode set (Tier 1.2, partially), a client router, and a forms
story. That is weeks of work and produces something genuinely good.

**If the goal is "replace React"**: Option B is the only honest path, and the
first deliverable is not in raven-web at all: it is `raven build --target js`.
raven-web then becomes the framework layer on top (components, signals, router),
and most of the current VM is deleted.

The worst outcome is drifting: adding opcodes forever, calling it a React
alternative, and being neither.

## 6. If Option B is chosen: rough shape

1. **Compiler**: a JS backend behind `--target js` emitting ES modules from
   HIR/MIR; a small runtime shim for Raven's value model, closures, and traits;
   `extern "js"` for DOM/browser APIs.
2. **Reactivity**: signals (`signal`, `computed`, `effect`) in Raven, so updates
   are fine-grained and keyed.
3. **Components**: Raven structs with typed props and local state, returning a
   view; compile-time-checked composition, no selector strings.
4. **Router**: typed routes, params, nested layouts, code splitting.
5. **Keep from today**: the safety model, `Theme`/design tokens, the CSS
   builders, `Site`/`Layout`, the dev server and live reload, static export (as
   SSG/SSR output rather than the only mode).
