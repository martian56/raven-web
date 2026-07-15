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
"github.com/martian56/raven-web" = "v0.8.0"
```

Raven Web is tested with Raven 2.26.1 on Linux and Windows.

## The developer model

- `Node` is the fluent, XSS-safe HTML builder. Composition helpers `child_each`,
  `child_when`, `each`, and `when` build lists and conditional structure inline.
- `Component` lets a Raven struct expose `to_node()` for a stateless view.
- `app()` and `Ctx` give reusable components typed props and per-instance local
  state, so the same component can be used many times on one page.
- `Signal` is a typed handle to client state; `Expr` computes over signals for
  conditionals, derived text, and form validation. `bind_value` makes a field
  controlled, and `disabled_when` gates a submit button on a rule.
- `Behavior` is a typed, composable sequence of DOM effects attached directly to
  an element event. No handwritten JavaScript, no stringly-typed action names.
- `Page` owns metadata, styles, client state, bindings, and output files.
- `Router` routes on the client: patterns, params, guards, and history, with no
  reload. `Site` and `Layout` build multi-page sites with a shared shell,
  clean-URL routing, a sitemap, and a 404 page.
- `Stylesheet`, `CssRule`, and `Theme` build authored CSS and design tokens.
- The `dev` module (`serve`, `watch`) runs a local server with live reload.

Builders are infallible and silently reject unsafe input (an invalid tag, URL,
attribute, or selector is dropped). Each has a `try_*` twin that returns a
`Result` when you want the rejection signalled.

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
copy-value, copy-to-clipboard, and the client-state effects below.

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

## Dev server with live reload

```rust
import "github.com/martian56/raven-web/dev" { reloader, serve, watch }

fun main() {
    let state = reloader()
    build()
    spawn(fun() -> Unit {
        watch(["src"], state, fun() -> Unit { build() })
    })
    serve("dist", 8080, state)
}
```

`serve` hosts the built directory and injects a live-reload client (generated,
never handwritten). `watch` rebuilds on source change and refreshes the browser.
The dev module is separate so a build-only project does not pull in the HTTP
machinery.

## Output

`write_static("dist")` writes `index.html`, `styles.css`, `app.js`, and any
files added by `css_file` or `file`. Paths are validated as safe relative paths,
HTML is escaped, executable-looking attributes and unsafe URLs are rejected, and
all interaction code is structured data interpreted by a fixed runtime rather
than string-injected script.

## Examples

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
- `examples/dev.rv`: the build, watch, and serve dev loop.
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
