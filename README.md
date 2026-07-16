# raven-web

[![CI](https://github.com/martian56/raven-web/actions/workflows/ci.yml/badge.svg)](https://github.com/martian56/raven-web/actions/workflows/ci.yml)

raven-web is a static-first frontend library for Raven where you never write
JavaScript. Build components from structs and impl blocks, describe interactions
as typed values, and emit a deployable directory of HTML, CSS, JavaScript, and
static files. raven-web owns 100% of the generated JavaScript; the public API
never accepts a raw JS string.

It stays browser-framework-agnostic. Tailwind, UnoCSS, handwritten CSS, a
design-system bundle, or a raven-web `Theme` are all just CSS.

## Install

Add the latest release to `rv.toml`:

```toml
[dependencies]
"github.com/martian56/raven-web" = "v0.18.0"
```

Raven Web is tested with Raven 2.26.1 on Linux and Windows.

## Quickstart

One file is a whole project. Save this as `site.rv` next to the `rv.toml`
above (or copy `examples/starter.rv`):

```rust
import std/io { println }
import "github.com/martian56/raven-web" { Behavior, Node, Page, signal }
import "github.com/martian56/raven-web/dev" { cli }

fun build() {
    let count = signal("count", "0")
    let page = Page.new(
        "Hello",
        Node.main_node().child(Node.h1().child_text("Hello from Raven")).child(
            Node.button("+1").on_click(Behavior.new().increment(count, 1)),
        ).child(Node.strong().text_of(count)),
    ).signal(count)
    match page.write_static("dist") {
        Ok(out) -> println("wrote ".concat(out.file_count().to_string()).concat(" files")),
        Err(error) -> println(error.to_string()),
    }
}

fun main() {
    cli("site.rv", "dist").run(fun() -> Unit {
        build()
    })
}
```

Two commands from there:

```text
raven build site.rv -o site.exe
./site.exe dev
```

Open http://localhost:8080. Edit `site.rv` and save: the harness recompiles
it, regenerates the site, and the browser reloads. Break the syntax and save:
the compiler error appears in the browser until you fix it. When you are done,
`./site.exe` writes `dist/`, which deploys to any static host.

The long-form walkthrough is [docs/GETTING-STARTED.md](docs/GETTING-STARTED.md).

## The developer model

- `Node` is the fluent, XSS-safe HTML builder. Composition helpers `child_each`,
  `child_when`, `each`, and `when` build lists and conditional structure inline.
- `Component` lets a Raven struct expose `to_node()` for a stateless view.
- `app()` and `Ctx` give reusable components typed props and per-instance local
  state, so the same component can be used many times on one page.
- `Signal` is a typed handle to client state; `signal_of` seeds one with a typed
  Raven value, so data reaches the client with no fetch. `Expr` computes over
  signals for conditionals, derived text, and form validation. `bind_value`
  makes a field controlled, and `disabled_when` gates a submit button on a rule.
- `Behavior` is a typed, composable sequence of DOM effects attached directly to
  an element event. No handwritten JavaScript, no stringly-typed action names.
- `Page` owns metadata, styles, client state, bindings, and output files.
  Head metadata is typed too: `meta`, `meta_property` (Open Graph, Twitter),
  `canonical`, `theme_color`, `preconnect`, and JSON-LD through
  `structured_data(JsonValue)`, which escapes every angle bracket inside the
  JSON so the payload can never close its script element.
- `Router` routes on the client: patterns, params, guards, and history, with no
  reload. `Site` and `Layout` build multi-page sites with a shared shell,
  clean-URL routing, a sitemap, and a 404 page.
- `Stylesheet`, `CssRule`, and `Theme` build authored CSS and design tokens.
- The `dev` module wraps a project in a standard CLI (`cli`): build, serve,
  and a dev mode that recompiles your source on save, live-reloads the
  browser, and shows compile errors as a browser overlay.

Builders are infallible and reject unsafe input (an invalid tag, URL,
attribute, or selector is dropped rather than emitted). Every drop is
recorded: `write_static` prints each one as `raven-web warning: ...`,
`Page.warnings()` returns them, and `Page.strict()` makes `write_static`
refuse to write while any exists, which is what a CI build wants. Each builder
also has a `try_*` twin that returns a `Result` when one call site wants the
rejection as a value.

```rust
import "github.com/martian56/raven-web" { Behavior, Component, Node, Page, Stylesheet }

struct Hero {
    title: String,
}

impl Hero {
    fun new(title: String) -> Hero {
        return Hero { title }
    }
}

impl Component for Hero {
    fun to_node(self) -> Node {
        return Node.main_node().class_name("hero").child(Node.h1().child_text(self.title)).child(
            Node.button("Say hi").on_click(Behavior.new().set_text("#msg", "Hello from Raven")),
        ).child(Node.p().id("msg"))
    }
}

fun main() {
    let page = Page.new("Home", Hero.new("Hello from Raven").to_node()).lang("en").description(
        "A Raven-generated static page.",
    ).with_styles(Stylesheet.new().raw(".hero { padding: 4rem; }"))
    match page.write_static("dist") {
        Ok(_) -> print("wrote dist"),
        Err(e) -> print(e.message()),
    }
}
```

`Page.new` takes a `Node` explicitly; pass `MyComponent { ... }.to_node()` at the
package boundary.

## Typed interactions, no JavaScript

Attach a `Behavior` to an element event. raven-web serializes it into a data
attribute and interprets it with a fixed, audited runtime it generates for you.

```rust
let form = Node.form().on_submit(
    Behavior.new().prevent_default().set_value("#title", "").add_class("#title", "is-saved").focus(
        "#title",
    ),
).child(Node.input_field("Title").id("title")).child(Node.submit_button("Save"))
```

Events: click, input, change, submit, keydown, keyup, focus, blur, mouseenter,
mouseleave, scroll. Effects: set text/value/attribute, add/remove/toggle class,
focus, show/hide/toggle-visible, navigate (safe URLs only), scroll-to,
copy-value, copy-to-clipboard, open/close a dialog, and the client-state
effects below.

Polish for real interfaces:

- `Node.on_key("Escape", behavior)` fires only for the named key, so an
  unmatched key keeps its browser behaviour.
- `Behavior.debounce(300)` runs after the event has been quiet for the
  window, so a search field fires once per pause, not once per keystroke.
- `Node.dialog()` with `Behavior.open_dialog("#id")` / `close_dialog` uses
  the native modal: the browser supplies the focus trap, Escape-to-close,
  and the backdrop, with a graceful fallback where `showModal` is missing.
- `Page.every(5000, behavior)` runs a behavior on an interval, starting
  after boot: a carousel advance, a clock, or a polling `fetch` that keeps a
  list fresh. Intervals under 100 ms are refused.
- `Node.visible_if(sig).transition()` animates a show/hide instead of
  snapping it: the element keeps its box and toggles the `rw-hidden` class,
  so author CSS owns the animation. `transition_styles()` is a
  reduced-motion-safe fade default; a visitor who asked for less motion gets
  an instant swap.

## Client state and bindings

State is a typed `Signal`. Declare it once, then pass the handle around: nodes
bind to it and behaviors write to it. No selector and no state string is written
by hand, so a typo is a compile error instead of a silent no-op.

```rust
let n = signal("n", "0")

let ui = Node.div().child(Node.span().text_of(n)).child(
    Node.button("+").on_click(Behavior.new().increment(n, 1)),
)
let page = Page.new("Counter", ui).signal(n)
```

Node bindings: `text_of`, `class_of`, `attr_of`, `visible_if`, `hidden_if`,
`each_of`, `each_where`. Behavior writes: `set`, `toggle`, `increment`, `clear`.
raven-web resolves the dependency graph at build time, so a write updates only
the bindings that read that signal, and lists reconcile by key rather than being
rebuilt (an input inside a row keeps its value and focus).

## Typed data, without a fetch

`signal_of` seeds a signal with structured data, so a typed Raven value reaches
the client as data rather than as a quoted string. The list is in the document
on first paint: no request, no spinner, no empty flash.

```rust
@derive(ToJson)
struct Task {
    id: Int,
    title: String,
    status: String,
}

let tasks = signal_of("tasks", board().to_json())   // board() -> List<Task>
Node.ul().each_of(tasks, row())
Page.new("Board", body).signal(tasks)
```

It stays live state: behaviors can clear or replace it, and `fetch` can refill
it later. `Page.data(key, json)` does the same for a key you address directly,
and `ctx.local_of` gives a component structured local state.

## Accessibility

The pieces a screen reader needs are bindings like any other.

```rust
fun accordion(ctx: Ctx, heading: String, body: String) -> Node {
    let open = ctx.local("open", "")
    let panel = ctx.id_for("panel")          // unique to this instance
    return Node.section().child(
        Node.button(heading).attr("aria-controls", panel).aria_if(open, "expanded"),
    ).child(Node.p().id(panel).child_text(body).visible_if(open))
}
```

- `aria_if(signal, name)` and `aria_when(expr, name)` keep an ARIA state
  correct. Use these rather than `attr_of`: ARIA states are the words `true` and
  `false`, while raven-web's truthiness is `"1"` and `""`, so
  `attr_of(open, "aria-expanded")` writes `aria-expanded="1"`, which is invalid
  and leaves the control announcing as collapsed with nothing reporting it.
- `ctx.id_for(name)` gives an id unique to the component instance, which is what
  makes `aria-controls`, `aria-describedby`, and `label for` work in a component
  used more than once.
- `Router` moves focus to the matched route on a navigation, so the new view is
  announced. The container is `tabindex="-1"`, so it takes focus without joining
  the tab order, and focus is not stolen on first paint.
- `visible_if`/`hidden_if` hide with `display:none`, so hidden content is out of
  the accessibility tree too, and `disabled_when` sets the real `disabled`
  property rather than a class.

`examples/form.rv` shows a field wired up properly: label tied by id,
`aria-describedby` pointing at the error, `aria-invalid` tracking the rule, and
`role="alert"` so the message is read out when it appears.

## Performance

The runtime is a fixed ~3.6 KB gzipped whatever the page does, and the HTML is
complete at build time, so content never waits on JavaScript. The dependency
graph is resolved at build time, so a write wakes only the bindings that read
that key: on a page with 401 bindings and a 500 row list, clicking one counter
re-applies 2 bindings, scans the document 0 times, and touches no rows.

Measured, with the method and the caveats, in
[docs/PERFORMANCE.md](docs/PERFORMANCE.md).

## Safety, and what is enforced

raven-web owns all of the page's JavaScript: the public API accepts no raw JS
string, and the runtime is a fixed constant carrying zero author input. What
backs that up:

- **No element can execute.** `script`, `style`, `base`, `meta`, `link`,
  `iframe`, `frame`, `frameset`, `object`, `embed`, `applet`, `portal`,
  `noembed`, and `noscript` are refused, subtree and all. Head metadata is
  `Page`'s job and CSS is `Stylesheet`'s, so nothing legitimate needs them.
- **No attribute can execute.** `on*`, `style`, `srcdoc`, and `http-equiv` are
  refused.
- **Every URL is judged by scheme**, whether it is set through `link`/`image` or
  through the raw `attr` escape hatch. `http`, `https`, `mailto`, `tel`, and
  scheme-less relative paths are allowed; everything else, including
  `javascript:`, `vbscript:`, and `data:`, is refused however it is spelled
  (mixed case, or with the control characters browsers strip out of a scheme).
- **Text is escaped**, and template data is injected with `textContent` and
  `setAttribute`, never `innerHTML`.
- **Nothing is `eval`'d.** Expressions and behaviors are data the runtime walks;
  `Expr.matches` compiles an authored pattern as a `RegExp`, which is data too.
- The generated JS is always an external file, so page data cannot break out of
  a `<script>` tag.

A rejected element or URL renders as nothing rather than as an unsafe
approximation, and every rejection is recorded as a build warning:
`write_static` prints them, `Page.warnings()` returns them, `Page.strict()`
turns them into a failed build, and each builder's `try_*` twin reports one
rejection as a `Result` at the call site.

## Expressions

`Expr` computes over signals, so a view can express a condition or derived text
without JavaScript. It evaluates as data; nothing is ever `eval`'d.

```rust
// "1 row" vs "N rows"
Node.span().text_expr(
    Expr.ternary(
        Expr.eq(Expr.len(Expr.of(rows)), Expr.int(1)),
        Expr.text("1 row"),
        Expr.concat(Expr.len(Expr.of(rows)), Expr.text(" rows")),
    ),
)
// an empty state
Node.p().child_text("Nothing yet").visible_when(Expr.is_empty(Expr.of(rows)))
```

Available: `of`, `text`, `int`, `flag`, `not`, `len`, `is_empty`, `eq`, `ne`,
`gt`, `lt`, `and`, `or`, `add`, `sub`, `concat`, `matches`, `ternary`. This is a
DSL, not Raven: it has no arbitrary function calls, and it cannot share domain
logic with your server code.

## Components

A component is an ordinary Raven function: its props are typed parameters, so
the compiler checks every use. `ctx.local` gives an instance private state
without inventing a global key, so one component can be reused on a page and
each copy behaves independently.

```rust
fun counter(ctx: Ctx, label: String, step: Int) -> Node {
    let n = ctx.local("n", "0")
    return Node.div().child(Node.span().child_text(label)).child(
        Node.span().text_of(n),
    ).child(Node.button("+").on_click(Behavior.new().increment(n, step)))
}

fun build() -> Page {
    let ui = app()
    let view = Node.div().child(counter(ui.instance(), "clicks", 1)).child(
        counter(ui.instance(), "by five", 5),
    )
    return ui.page("Counters", view)   // registers every local that was declared
}
```

Call `ui.instance()` once per instance; components nest by passing
`ctx.instance()` to a child. Two components handed the *same* `Ctx` share its
state deliberately, which is how a parent hands a child a signal it owns.

Components are expanded while the tree is built, so a stateful component cannot
yet be repeated per row of a runtime-fetched list. List rows stay `{{field}}`
templates. See [docs/FRONTEND-GAP-ANALYSIS.md](docs/FRONTEND-GAP-ANALYSIS.md)
for why, and what it would take to lift it.

## Forms

`bind_value` makes a field controlled: the signal *is* the value, so typing
writes it and writing it updates the field. Validation is then just an `Expr`
over that signal, and `disabled_when` keeps submit off until the form is valid.

```rust
let email = signal("email", "")
let touched = signal("touched", "")
let bad = Expr.not(Expr.matches(Expr.of(email), "^[^@\\s]+@[^@\\s]+\\.[^@\\s]+$"))

Node.input_field("Email").bind_value(email).on_blur(Behavior.new().set(touched, "1"))
// an error that waits until the field has been left
Node.p().child_text("Enter a valid email.").visible_when(Expr.and(Expr.of(touched), bad))
Node.submit_button("Save").disabled_when(bad)
```

- A checkbox or radio models as `"1"` or `""`, so it reads as truthy state.
- `Expr.matches(value, pattern)` compiles the pattern as a `RegExp`. The pattern
  is authored, never taken from user input, and is data rather than code: still
  nothing is `eval`'d. An invalid pattern matches nothing instead of throwing.
- Dirty/touched tracking needs no primitive: set a signal on blur, as above.
- Only a whole signal can be modelled. A field path (`auth.at("token")`) is a
  read-only view, so `bind_value` refuses it rather than silently failing to
  write; `try_bind_value` reports it.

See `examples/form.rv` for a signup form with cross-field rules.

## Client router

`Router` renders every route's view once at build time; the runtime shows the
one matching the address and hides the rest. Navigation changes no document and
reloads nothing, so state survives it: a fetched list stays fetched.

```rust
let router = Router.new().route("/", home).route("/tasks/:id", task_detail).route(
    "/login", login,
).route("/settings", settings).requires(auth, "/login").not_found(missing)

Page.new("App", Node.main_node().child(nav).child(router.to_node())).spa()
```

- A `:name` segment captures one path segment, read as `router.param("name")`
  like any other signal. Params live under the reserved state key `route`.
- `requires(signal, path)` guards the route added most recently: if the signal
  is falsy the runtime redirects instead of showing the view, so a guarded view
  never renders, including on a direct deep link. It is a convenience, not a
  security boundary. The server still has to protect the data.
- `Node.route_link(text, path)` keeps a real `href`, so it still opens in a new
  tab, middle-clicks, and is crawlable; only a plain left click is intercepted.
- `Behavior.go(path)` navigates from any behavior. `navigate` still does a full
  page load.
- Declaration order breaks ties between two matching patterns, and `not_found`
  is used only when nothing else matched.
- `Page.spa()` also writes `404.html`. Static hosts serve that for a path they
  have no file for, which is how a deep link survives a reload. The dev server
  does the same, so dev matches production.

## Multi-page sites

```rust
import "github.com/martian56/raven-web" { Layout, Node, Site }

let layout = Layout.new().header(Node.header().child_text("My site")).footer(
    Node.footer().child_text("(c) 2026"),
)
let site = Site.new("https://example.com").page("/", layout.wrap("Home", home_body)).page(
    "/about",
    layout.wrap("About", about_body),
)
site.write_static("dist")
```

Each route becomes a clean-URL directory (`/about` to `about/index.html`).
`write_static` also emits `sitemap.xml` and `404.html`.

## Design tokens and themes

```rust
import "github.com/martian56/raven-web" { Theme }

let theme = Theme.new().color("bg", "#ffffff", "#0b1020").color("fg", "#111111", "#e6edf6").scale(
    "space-3",
    "1rem",
)
page.with_styles(theme.stylesheet())
```

Tokens are emitted as CSS custom properties (`var(--color-bg)`). Dark values
apply under `prefers-color-scheme` and under an explicit `data-theme` attribute,
which a `Behavior` can set for a manual toggle.

## The dev workflow

`cli` wraps a site generator in the standard commands, so every raven-web
project works the same way:

```rust
import "github.com/martian56/raven-web/dev" { cli }

fun main() {
    cli("site.rv", "dist").run(fun() -> Unit {
        build()
    })
}
```

```text
./site.exe          # generate the site (prints file count, time, warnings)
./site.exe dev      # serve with live reload, rebuild on every source change
./site.exe serve    # build once and serve, without watching
./site.exe help     # usage
```

Dev mode watches the entry file's directory plus any `.watch(path)` extras,
excluding the output directory, `target`, `.git`, `node_modules`, and compiled
executables, so the build's own output never retriggers it. On a change it
recompiles the entry file with the `raven` compiler, regenerates the output by
running the fresh build, and reloads the browser. A compile error appears in
the browser as a full-screen overlay (injected through `textContent`, never
markup) and clears on the next good build. `.port(3000)` changes the port.

The `raven` compiler must be on PATH for dev mode; plain `build` and `serve`
never invoke it. The lower-level pieces (`reloader`, `serve`, `watch`,
`inject_reload`) stay available for a custom loop, and the dev module is
separate so a build-only project does not pull in the HTTP machinery.

## Output

`write_static("dist")` writes `index.html`, `styles.css`, `app.js`, and any
files added by `css_file` or `file`. Paths are validated as safe relative paths,
HTML is escaped, executable-looking attributes and unsafe URLs are rejected, and
all interaction code is structured data interpreted by a fixed runtime rather
than string-injected script.

## Multi-language sites

Declare a locale set once, then localize each page. `localize` sets the
`lang`, the canonical URL, and the full hreflang alternate set (every locale
plus `x-default`) in one call, so a multilingual site stops hand-wiring SEO
per page. Each locale is served as its own path prefix, which is what
`Site.page` expects.

```rust
let langs = i18n("https://example.com").default_locale("az").locale("en", "en")

let site = Site.new("https://example.com")
for locale in langs.all() {
    let page = langs.localize(Page.new(title, view(locale)), locale, "")
    site.page(langs.route(locale, ""), page)   // "/" for az, "/en" for en
}
```

`langs.url(locale, "about")` gives the clean canonical URL for a sub-path,
matching what `Site` emits.

## Content: markdown, frontmatter, collections, feeds

A content site is first-class. Markdown renders through the same safe
builders as everything else: text is escaped, URLs are judged by scheme, and
raw HTML in a document is text, not markup.

```rust
let posts = sort_documents_desc(load_documents("content"), "date")

for post in posts {
    let body = Node.new("article").child(post.html())   // markdown, rendered
    site.page("/posts/".concat(post.slug), layout.wrap(post.get("title"), body)).lastmod(
        post.get("date"),
    )
}
page.file(StaticFile.new("feed.xml", atom_feed(title, url, feed_url, updated, entries)))
```

- `markdown(source)` renders a deliberate CommonMark subset: ATX headings,
  paragraphs, fenced code (with `language-x` classes), blockquotes, flat
  bullet and numbered lists, horizontal rules, emphasis, strong, inline
  code, links, images, and backslash escapes. No raw HTML passthrough, by
  design; no tables, footnotes, reference links, or nested lists yet.
- `parse_document(source)` splits `--- key: value ---` frontmatter from the
  body; `load_documents(dir)` parses every `.md` under a directory with the
  file name as the slug; `sort_documents_desc(docs, "date")` orders them
  newest first; `chunks(items, size)` splits for pagination.
- `atom_feed(...)` builds an Atom feed from the same ISO dates frontmatter
  uses, escaped like everything else.
- Markdown headings get a GitHub-style slug `id`, so in-page anchors work.
  `headings(source)` extracts every heading (`level`, `text`, `slug`), and
  `table_of_contents(source, min, max)` builds a `<nav class="toc">` of
  anchor links for a level range.
- Fenced code blocks are syntax-highlighted at build time (raven, rust,
  js/ts, go, python, c/c++/java, bash) into classed spans, no client JS;
  `highlight(code, lang)` does the same for a standalone snippet, and
  `syntax_styles()` is a dark token theme you can override. Highlighting
  runs through the escaping builders, so it can never inject markup.

`examples/blog.rv` is the whole pattern: a directory of markdown becomes a
site with an index, clean post URLs, a dated sitemap, and a feed.

## Production output

- `page.optimize()` minifies the emitted CSS and content-hashes the asset
  names (`styles.7bdddcdb.css`, `app.42c4b729.js`), so far-future cache
  headers can never serve a stale deploy; older hashed copies are removed on
  rebuild. Off by default, so dev output keeps stable names.
- `page.base_path("/repo")` makes the site work under a subpath host (a
  GitHub Pages project site): asset references become absolute under the
  prefix, and the client router matches, navigates, and rewrites route-link
  hrefs beneath it.
- `Node.image` is `loading="lazy" decoding="async"` by default. `image_sized`
  adds the intrinsic width and height that prevent layout shift, `eager()`
  opts a hero image out of lazy loading and raises its fetch priority, and
  `srcset_add(url, width)` plus `sizes(...)` build responsive candidates.
- `page.alternate("en", url)` emits hreflang alternate links for
  multilingual sites, and `site.page(...).lastmod("2026-07-16")` dates the
  sitemap entry.
- `router.query("tab")` (or `query_param("tab")` without a router) reads
  `?tab=` as a signal under the reserved `query` key; the runtime re-parses
  it on load and on every navigation.

## Starters

Copyable templates to begin a new site from working code, not a blank file.
Each is one self-contained file; copy it out, rename it, and run.

- `examples/starters/landing.rv`: a marketing landing page (hero, toggled
  feature grid, CTA).
- `examples/starters/portfolio.rv`: a personal portfolio (intro, project
  cards, contact).
- `examples/starters/docs.rv`: a multi-page docs site with a shared sidebar
  and markdown content.
- `examples/starters/blog.rv`: a self-contained blog with inline markdown
  posts, a sitemap, and an Atom feed.

```text
raven build examples/starters/landing.rv -o site.exe
./site.exe dev        # serve with live reload
./site.exe            # write the dist directory
```

## Examples

- `examples/blog.rv`: a markdown-driven blog: frontmatter, collections, clean URLs, a dated sitemap, and an Atom feed.
- `examples/starter.rv`: the canonical single-file project (view, styles,
  build, and the standard CLI). Copy it to start a new site.
- `examples/landing.rv`: a self-contained 2.0 showcase (components, composition,
  inline behaviors, expanded effects).
- `examples/components.rv`: reusable components with typed props and
  per-instance local state, nested two deep.
- `examples/reactive.rv`: typed signals, targeted updates, keyed list
  reconciliation, and expressions (pluralized text, an empty state).
- `examples/router.rv`: client routing with params, a guard, a fallback, and
  state surviving navigation.
- `examples/form.rv`: a validated signup form with controlled fields,
  cross-field rules, and errors that wait until a field is left.
- `examples/typed_data.rv`: a typed `List<Task>` rendered on first paint with no
  request.
- `examples/bench.rv`: the page the performance numbers are measured on (200
  component instances plus a 500 row list).
- `examples/dev.rv`: the standard dev loop via `cli` (build, dev, serve).
- `examples/landing_v1_dashboard.rv.bak`: the earlier dashboard, kept for
  reference (uses the pre-2.0 API).

```text
raven build examples/landing.rv -o landing.exe
./landing.exe
```

## Checks

```text
rvpm fmt --check
rvpm test
```

Testing a site you build with raven-web is covered in
[docs/TESTING.md](docs/TESTING.md): golden-HTML assertions over `page.html()`,
`page.warnings()` as a lint gate, and `strict()` in CI. The runnable version
is `examples/site_test.rv`.
