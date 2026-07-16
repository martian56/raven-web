# raven-web developer experience design

Status: implemented in v0.12.0.

The goal of this release is the developer's minute-to-minute experience: what
happens between saving a file and seeing the result, and what happens when the
input was wrong. Three things ship together because each one covers a failure
mode the other two cannot.

## 1. Build warnings: silent drops become loud

raven-web's builders are infallible by design. An invalid tag, an unsafe URL,
an executable attribute, or a malformed class token is dropped rather than
emitted, and the page stays safe. The cost has always been that the drop is
silent: the author typed something, the page ignored it, and nothing said so.
Every significant bug in this library's history shipped exactly that way.

The fix keeps the infallible API and adds a record. Every drop site now pushes
a one-line reason onto the value it dropped from:

- `Node` carries warnings for its own drops (invalid tag, refused attribute,
  unsafe URL, invalid binding argument). A dropped element still enters the
  tree as an inert node, so its warning travels with it.
- `Behavior` and `FetchSpec` carry warnings for dropped effects (an unsafe
  navigate URL, an invalid class token). Attaching the behavior to a node
  merges them into the node.
- `Stylesheet` carries warnings for rejected CSS, merged into the page by
  `with_styles` and `with_css`.
- `Page` carries its own (invalid language, unsafe favicon, rejected state
  key, rejected selector, rejected static file path).

`Page.warnings()` walks the tree and returns every message. `write_static`
prints each one, prefixed `raven-web warning:`, and still succeeds.
`Page.strict()` opts into failure: with it, `write_static` returns `Err` when
any warning exists, which is what a CI build wants.

The `try_*` twins are unchanged. They remain the right tool when one call site
wants to handle one rejection; warnings are for everything nobody thought to
check.

## 2. The dev command: save, and the browser is already right

Until now the dev loop had a hole. `dev.serve` and `dev.watch` could rebuild
the output when watched files changed, but the watcher runs inside the compiled
site generator, so it could never pick up a change to the generator's own Raven
source. Editing your site meant rebuild, restart, refresh, by hand.

`dev.cli` closes the loop using `std/process`. The author writes one main:

```rust
import "github.com/martian56/raven-web/dev" { cli }

fun main() {
    cli("site.rv", "dist").run(fun() -> Unit {
        build()
    })
}
```

and the compiled program is now a small CLI:

- `site.exe` or `site.exe build`: generate the site once, print the file count
  and the elapsed time.
- `site.exe dev`: build, serve with live reload, and watch the project source.
  When a `.rv` file changes, the harness recompiles the author's entry file
  with `raven build` into a scratch executable, runs it with `build` to
  regenerate the output, and bumps the reload token. The browser reloads with
  the new code. Nothing is restarted by hand.
- `site.exe serve`: build once and serve, no watching.
- `site.exe help`: usage.

Options are builder methods: `.port(3000)` and `.watch("assets")` for paths
outside the entry file's directory. The default watch root is the directory
containing the entry file, with `dist`, `target`, `.git`, and `node_modules`
excluded so the build's own output can never retrigger the build.

When the recompile fails, the compiler's error appears in the browser: the
reload endpoint carries the error text alongside the token, and the injected
client renders it in a full-screen overlay instead of reloading. Fixing the
file clears the overlay and reloads. The error text reaches the DOM through
`textContent`, never markup, and the overlay client follows the same string
rules as the rest of the owned runtime (single quotes, no backslash literals,
no template literals).

Known limits, stated plainly:

- The recompile runs `raven` from PATH. If the compiler is not installed, dev
  mode reports it in the overlay and keeps serving the last good build.
- The scratch executable lives at `target/.rw-dev.exe`. On Windows a crashed
  child can hold a lock on it; the next rebuild reports the link error in the
  overlay, and killing the stale process recovers.
- Watching is polling (400 ms) by content signature. It is simple and correct;
  it is not designed for very large trees.

## 3. Documentation for the first hour

- The README opens with a quickstart that goes from empty directory to a
  live-reloading page in two commands.
- `docs/GETTING-STARTED.md` is the long-form walkthrough: install, first page,
  the dev loop, state and bindings, components, deploying `dist` to a static
  host.
- `examples/starter.rv` is the canonical single-file project: copy it, rename
  it, run it. `examples/dev.rv` now demonstrates `cli` instead of hand-wiring
  `serve` and `watch`.

## What this release does not claim

The ceiling from `FRONTEND-GAP-ANALYSIS.md` is unchanged: Raven does not run
in the browser, list rows remain `{{field}}` templates, and there is no SSR.
The dev loop is faster and honest about failure; it does not make the library
more capable than it was.
