# raven-web 2.0 design

Status: DELIVERED, 2026-07-14. All five workstreams landed and tested (16 tests
green, fmt clean, examples build, dev server verified live). This document
remains the north star for the static-first frontend library where the author
never writes JavaScript.

## Principles

1. Static first. The output is a directory of HTML, CSS, JS, and assets that any
   static host can serve. Rendering happens at build time.
2. raven-web owns 100% of the emitted JavaScript. The public API never accepts a
   raw JS string. Interactivity is expressed through typed Raven values that
   compile to a fixed, audited client runtime plus serialized behavior data.
3. Safe by construction. Tags, attributes, URLs, CSS, and text are validated and
   escaped. Executable attribute names (on*), javascript: URLs, and style
   injection are rejected, as they are today.
4. Best DX. Infallible chainable builders, inline typed behaviors (no
   stringly-typed action indirection), real components, multi-page sites,
   a design-system CSS layer, and a dev server with live reload.

## API redesign (approved: free to redesign for best DX)

The current model wires interactivity through named string actions: create a
`Handler("save")`, register it on the `Page`, then reference the string "save"
on an element. Typos silently do nothing, and the wiring is spread across the
file. raven-web 2.0 replaces this with inline typed behaviors attached directly
to element events.

Before (today):

    let page = Page.new(...).on("toggle", fun(h) -> Handler = h.toggle_class("#m", "open"))
    // elsewhere:
    Node.button("Menu").on_click("toggle")   // string must match, or nothing happens

After (2.0):

    Node.button("Menu").on_click(Behavior.toggle_class("#m", "open"))

The element declares its own behavior. raven-web collects every behavior at
render time, deduplicates, and emits the runtime plus a JSON behavior table. No
action names to keep in sync, no separate registration step.

### Module layout

Raven constraint (verified on 2.26.1): a type that `lib.rv` only imports is not
re-exported to consumers importing from the package root, and Raven does not
support circular imports, so a function operating on a public type cannot live in
a different file from that type. Therefore every public type and the functions
that operate on it stay in `lib.rv`, which keeps the consumer's import story to a
single `import "github.com/martian56/raven-web" { ... }`. This is the best
available DX under the current module model.

- `lib.rv`: the full public surface. `Node`, `Behavior`, `Event`, `Effect`,
  `Page`, `Site`, `Layout`, `Stylesheet`, `CssRule`, `Theme`, `StaticFile`,
  the `Component` trait, and their impls and factory functions.
- Leaf utility files (optional) may hold only dependency-free helpers that take
  primitives and never reference a public type, avoiding circular imports. Used
  by `lib.rv` internally, not part of the consumer surface.

Note for the ecosystem: a re-export mechanism (a `pub use` equivalent, or making
`lib.rv`'s selective imports re-exportable) would remove this single-file
constraint and improve every multi-file Raven library. Tracked as a separate
language-level improvement, out of scope for this design.

## Workstream 1: foundation (components + unified error model)

Goal: a consistent, infallible, chainable core that everything builds on.

- Error model. Every structural builder constructor becomes infallible and
  follows the existing silent-reject convention: invalid tags, attribute names,
  URLs, CSS, and selectors are ignored rather than returning `Result`.
  `Node.new`, `CssRule.new`, and `StaticFile.new` lose their `Result` returns.
  For callers who want to detect rejection, add `try_*` variants that return
  `Result` (mirroring the existing `CssRule.try_set`). One rule: convenience is
  infallible, `try_*` is fallible, and both share one validator. The interactivity
  constructors (today's `Handler.new`) are not patched here: workstream 2 removes
  the named-action model entirely in favor of typed `Behavior` values.
- Components. Keep the `Component` trait (`to_node`). Add ergonomic helpers:
  `Node.children(list)`, `Node.when(cond, node)` for conditional children,
  `Node.each(list, fun(T) -> Node)` for mapped children, and text helpers so
  authors stop writing `.child(Node.text(...))`.
- Elements and controls. Fill out the semantic set (figure, figcaption, details,
  summary, blockquote, hr, etc.) and form controls (radio, fieldset, legend,
  number/email/password inputs, textarea rows, select/optgroup). Every control
  has a typed constructor.
- Tests. Rework `lib_test.rv` for the new infallible API and cover each new
  helper and control.

## Workstream 2: interactivity VM (the "replace JS" core)

Goal: express real static-site interactivity with zero author JavaScript.

- Events (typed): click, input, change, submit, plus keydown, keyup, focus,
  blur, mouseenter, mouseleave, and scroll. Each carries prevent-default and
  stop-propagation options.
- Effects (typed, composable into a `Behavior`): today's set (set_text,
  set_value, set_attribute, add/remove/toggle_class, focus) plus show/hide,
  toggle-visibility, navigate (safe URL only), scroll-into-view, set-css-var,
  copy-to-clipboard, and append-from-template.
- Client state: a small key to value store. Effects set, toggle, and increment
  state keys. Bindings reflect state to the DOM: `bind_text(sel, key)`,
  `bind_class(sel, key, class)`, `bind_attr(sel, key, attr)`,
  `bind_visible(sel, key)`. This is what unlocks counters, tabs, accordions,
  theme toggles, and filters with no author JS.
- Data flow: effects can read an element value or a state key and write it
  elsewhere, so form UX (live preview, validation display) works declaratively.
- Emission: a single fixed runtime function (audited once) interprets a JSON
  table of behaviors, bindings, and initial state. Values are escaped through
  the same JSON string path used today. The author never sees or writes JS.

Boundary (documented): computation heavy interactivity (arbitrary fetch, complex
logic) is out of scope for the static VM. The VM targets the common 90 percent
of static-site interactivity.

## Workstream 3: multi-page site and layouts

Goal: real multi-page sites, not just one page.

- `Layout`: shared shell (head defaults, header, footer, nav) that wraps a
  page body. Pages opt into a layout.
- `Site`: a set of routes (path to `Page`). `Site.write_static(dir)` emits every
  page at its route (`/about` to `about/index.html`), a `sitemap.xml`, and a
  `404.html`. Relative links between routes are computed so output works from any
  subdirectory.
- Nav helpers that mark the active route.

## Workstream 4: CSS design system

Goal: authors stop hand-writing raw CSS strings.

- Design tokens: color, space, radius, and type-scale tokens exposed as typed
  values and emitted as CSS custom properties.
- `Theme`: light and dark token sets with a `prefers-color-scheme` block and an
  optional state-driven toggle (built on the interactivity VM).
- Style helpers: higher-level builders (stack, cluster, grid, container) and a
  small, safe utility-class set, all validated like the existing CSS path.

## Workstream 5: dev server and live reload

Goal: edit, save, see it instantly.

- `serve(dir, opts)`: an `std/http` server that serves the built directory with
  correct content types and injects a live-reload client (generated JS).
- `watch(paths, rebuild)`: poll `std/fs.walk` and content-hash via `std/hash`;
  on change, run the `rebuild` closure and bump a reload token that the injected
  client polls at a reserved endpoint. Content-file changes rebuild in process;
  `.rv` source changes are handled by re-invoking the compiler through
  `std/process` (documented workflow).
- No new author JS: the live-reload client is part of raven-web's owned runtime.

## Testing strategy

- Unit tests per module in `*_test.rv`, run by `rvpm test`, kept green at every
  slice.
- Golden output: a representative page and a small multi-page site whose emitted
  HTML, CSS, and JS are snapshotted so regressions in generated output are
  caught.
- The `landing.rv` example is migrated to the 2.0 API and doubles as an
  integration smoke test.

## Delivery

Build in the order 1 to 5. Each workstream is a reviewable slice that keeps
`rvpm test` green and `rvpm fmt --check` clean. The library stays usable at every
step.
