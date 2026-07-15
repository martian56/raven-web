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
"github.com/martian56/raven-web" = "v0.3.0"
```

Raven Web is tested with Raven 2.26.1 on Linux and Windows.

## The developer model

- `Node` is the fluent, XSS-safe HTML builder. Composition helpers `child_each`,
  `child_when`, `each`, and `when` build lists and conditional structure inline.
- `Component` lets a Raven struct expose `to_node()`.
- `Behavior` is a typed, composable sequence of DOM effects attached directly to
  an element event. No handwritten JavaScript, no stringly-typed action names.
- `Page` owns metadata, styles, client state, bindings, and output files.
- `Site` and `Layout` build multi-page sites with a shared shell, clean-URL
  routing, a sitemap, and a 404 page.
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

Declare page-level state, mutate it from behaviors, and bind it to the DOM. This
covers counters, tabs, toggles, and theme switches with no author JavaScript.

```rust
let page = Page.new("Counter", ui).state("n", "0").bind_text("#count", "n")
// a button that increments:
Node.button("+").on_click(Behavior.new().increment_state("n", 1))
```

Bindings: `bind_text`, `bind_class`, `bind_attr`, `bind_visible`. State effects:
`set_state`, `toggle_state`, `increment_state`. The runtime applies bindings on
load and re-applies them after every state change.

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
