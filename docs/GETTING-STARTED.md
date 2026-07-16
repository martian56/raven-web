# Getting started with raven-web

This is the walkthrough from an empty directory to a deployed site. The
[README](../README.md) is the API reference; this document is the path
through it.

## What you need

- Raven 2.26.1 or newer on PATH (`raven -V`), which also installs `rvpm`.
- Nothing else. No Node, no bundler, no JavaScript.

## A project in one file

Make a directory with two files. `rv.toml`:

```toml
[package]
name = "my-site"
version = "0.1.0"
edition = "v2"

[dependencies]
"github.com/martian56/raven-web" = "v0.12.0"
```

and `site.rv`:

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

`build` describes the page and writes it; `main` hands `build` to `cli`, which
turns the compiled program into a small CLI. That is the whole structure of
every raven-web project, however large it gets.

## The commands

```text
raven build site.rv -o site.exe    # compile the generator once
./site.exe                         # generate dist/ (same as ./site.exe build)
./site.exe dev                     # serve, watch, live reload
./site.exe serve                   # serve without watching
```

`./site.exe dev` is where you live while developing:

- Open http://localhost:8080 (or `.port(3000)` on the `cli` builder).
- Edit `site.rv` and save. The harness recompiles it with the `raven`
  compiler, regenerates `dist/`, and the browser reloads. You never restart
  anything by hand.
- Break the syntax and save. The compiler error appears in the browser as an
  overlay and stays until the next good build. The last good site keeps
  serving underneath.
- Assets outside the entry file's directory join the watch list with
  `.watch("assets")`.

## When input is dropped

raven-web's builders never emit unsafe output. An executable attribute, an
unsafe URL, an invalid tag or selector is dropped instead. Every drop is
recorded and printed when the site is written:

```text
raven-web warning: dropped attribute "onclick" on <div>: invalid or executable attribute name
```

Three escalation levels, use what fits:

- `Page.warnings()` returns the list, for asserting in your own checks.
- `Page.strict()` makes `write_static` return `Err` while any warning exists.
  Put it on the page in CI so a dropped input fails the build.
- Each builder has a `try_*` twin (`try_new`, `try_bind_value`, ...) that
  returns a `Result` when one call site wants to handle one rejection.

## State and interactions

Client state is a typed `Signal`. Declare it once, pass the handle around:
views bind to it, behaviors write to it, and a typo is a compile error rather
than a silent no-op.

```rust
let open = signal("open", "")

Node.button("Menu").on_click(Behavior.new().toggle(open))
Node.nav().visible_if(open)
```

Derived values are `Expr` trees (`text_expr`, `visible_when`, `class_when`,
`disabled_when`, `aria_when`); forms are controlled with `bind_value` and
validated with the same expressions. The README sections on expressions and
forms show the patterns.

## Components

A component is an ordinary Raven function whose props are typed parameters.
`ctx.local` gives each instance private state:

```rust
fun counter(ctx: Ctx, label: String, step: Int) -> Node {
    let n = ctx.local("n", "0")
    return Node.div().child(Node.span().child_text(label)).child(
        Node.span().text_of(n),
    ).child(Node.button("+").on_click(Behavior.new().increment(n, step)))
}

fun build_page() -> Page {
    let ui = app()
    let view = Node.div().child(counter(ui.instance(), "clicks", 1)).child(
        counter(ui.instance(), "by five", 5),
    )
    return ui.page("Counters", view)
}
```

## Styling

Everything is just CSS in the end, so pick any of:

- `Stylesheet` and `CssRule` for authored styles in Raven.
- `Theme` for design tokens with light and dark values.
- `page.css_file("styles/site.css", css)` to ship a plain CSS file you wrote.
- `page.link_stylesheet(url)` for a hosted framework build.

## Data without a fetch

`signal_of` seeds a signal with a typed Raven value, so a list is in the
document on first paint with no request and no spinner:

```rust
@derive(ToJson)
struct Task {
    id: Int,
    title: String,
}

let tasks = signal_of("tasks", board().to_json())
Node.ul().each_of(tasks, Node.li().child_text("{{title}}"))
Page.new("Board", body).signal(tasks)
```

For runtime data, `request(method, url)` builds a typed fetch a `Behavior`
sends; `store_in` puts the response where bindings read it.

## Routing and multi-page

- `Router` gives one page client-side routes with params, guards, and
  history. Mark the page `.spa()` so deep links survive a reload.
- `Site` and `Layout` write a multi-page static site with clean URLs, a shared
  shell, `sitemap.xml`, and `404.html`.

Both are in the README with full examples.

## Deploying

`dist/` is complete, static, and self-contained: `index.html`, `styles.css`,
`app.js`, plus your assets. Put it on any static host (GitHub Pages, Netlify,
Cloudflare Pages, an S3 bucket, nginx). There is no server component. For an
SPA, the emitted `404.html` is what makes deep links work; most static hosts
serve it automatically.

A CI build is the same two commands, plus strict mode on your pages:

```text
raven build site.rv -o site.exe && ./site.exe
```

## What raven-web is not

Raven does not run in the browser, so client behaviour is typed data
interpreted by a fixed, audited runtime. That covers components, state,
routing, forms, fetches, and accessibility, but a stateful component cannot be
repeated per row of runtime-fetched data (rows use `{{field}}` templates), and
there is no SSR or hydration. The full argument is in
[FRONTEND-GAP-ANALYSIS.md](FRONTEND-GAP-ANALYSIS.md); this library does not
pretend otherwise.
