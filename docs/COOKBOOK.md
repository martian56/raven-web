# Component cookbook

Ready-to-copy recipes for the components a site needs, using the current
API. Every recipe here is in `examples/cookbook.rv` as one buildable page, so
the code is proven to compile and run; this page explains it.

```text
raven build examples/cookbook.rv -o cookbook.exe
./cookbook.exe        # writes dist-cookbook/
```

All state is a typed `Signal`, so a typo in a binding is a compile error,
not a silent no-op. Register every signal on the page with `.signal(sig)`.

## Accordion

A signal per panel. The button toggles it, `aria_if` keeps the ARIA state
correct, and `transition()` animates the body (pair with
`transition_styles()`).

```raven
fun accordion_item(open: Signal, heading: String, body: String) -> Node {
    return Node.section().child(
        Node.button(heading).on_click(Behavior.new().toggle(open)).aria_if(open, "expanded"),
    ).child(Node.div().visible_if(open).transition().child_text(body))
}
```

## Tabs

One signal holds the active index. Each button sets it; each panel is
`visible_when` the index matches; the active button styles itself with
`class_when`.

```raven
let active = signal("tab", "0")

Node.button("First").on_click(Behavior.new().set(active, "0")).class_when(
    Expr.eq(Expr.of(active), Expr.text("0")), "active",
)
Node.div().child_text("First panel").visible_when(Expr.eq(Expr.of(active), Expr.text("0")))
```

## Modal

The native `dialog`, opened and closed by behaviors. The browser supplies
the focus trap, backdrop, and Escape-to-close.

```raven
Node.button("Open").on_click(Behavior.new().open_dialog("#confirm"))

Node.dialog().id("confirm").child(Node.h3().child_text("Sure?")).child(
    Node.button("Close").on_click(Behavior.new().close_dialog("#confirm")),
)
```

Style the backdrop with `.modal::backdrop { ... }`.

## Responsive nav

A menu signal toggles the links on small screens; CSS shows them inline on
wide ones. The toggle button is hidden by a media query above the breakpoint.

```raven
let menu = signal("menu", "")

Node.button("Menu").on_click(Behavior.new().toggle(menu)).aria_if(menu, "expanded")
Node.ul().class_when(Expr.of(menu), "open").child(Node.li().child(Node.link("Home", "#")))
```

```css
@media (min-width: 640px) { .nav-toggle { display: none; } }
@media (max-width: 639px) {
  .nav-links { display: none; }
  .nav-links.open { display: flex; flex-direction: column; }
}
```

## Card grid

Static structure from a list, no client state. Build the cards in a loop and
lay them out with CSS grid.

```raven
let grid = Node.div().class_name("grid")
for item in items {
    grid.child(Node.article().child(Node.h3().child_text(item.title)).child(
        Node.p().child_text(item.blurb),
    ))
}
```

```css
.grid { display: grid; gap: 1rem; grid-template-columns: repeat(2, 1fr); }
```
