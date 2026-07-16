# Testing a raven-web site

A raven-web site is a Raven program, so you test it the way you test Raven:
with `std/test`, run by `rvpm test`. There is no browser or headless runner
to stand up. A page's `html()`, `css()`, `js()`, and `warnings()` are plain
values you assert over.

`examples/site_test.rv` is a complete, runnable version of everything below.

## Golden-HTML assertions

Build the page and assert over its emitted markup: what should be there, and
what should not.

```raven
fun test_home_renders_expected_markup() {
    let html = home().html()
    assert_true(html.contains("<title>Home</title>"))
    assert_true(html.contains("<h1>Welcome</h1>"))
    assert_true(html.contains("data-rw-click="))
    assert_false(html.contains("onclick"))     // nothing executable leaked in
}
```

Keep the page builder (`home()` here) as an ordinary function that returns a
`Page`, so a test can call it directly.

## warnings() as a lint gate

Every input raven-web drops is recorded. A clean page drops nothing, so a
single assertion guards against a whole class of silent mistakes (an unsafe
URL, an executable attribute, an invalid binding):

```raven
fun test_home_has_no_warnings() {
    assert_true(home().warnings().len() == 0)
}
```

And a test can assert that a known-bad input *is* caught:

```raven
let page = Node.div().child(Node.link("Bad", "javascript:alert(1)"))
let built = Page.new("X", page)
assert_false(built.html().contains("javascript:alert"))
assert_true(built.warnings().len() >= 1)
```

## strict() in CI

`Page.strict()` turns any warning into a failed build. Put it on your pages
and a dropped input fails the build instead of shipping a page that silently
does less than the source says. Your CI is then just:

```text
rvpm fmt --check
rvpm test
rvpm build      # builds the generator; run it to fail on strict warnings
./site          # writes the site; exits non-zero under strict() if anything dropped
```

## Where the tests live

In your project, put assertions in a `*_test.rv` file next to your source;
`rvpm test` runs them. `examples/site_test.rv` runs its assertions from
`main()` only so it is self-contained and can be built and run directly:

```text
raven build examples/site_test.rv -o t.exe && ./t.exe
```

## Beyond the string: driving the real page

Golden-HTML and `warnings()` cover the markup and the safety net. To test
that a behaviour actually *runs* (a counter increments, a tab switches), load
the emitted `index.html` and `app.js` into a DOM and drive them. raven-web's
own suite does this with jsdom in `testing/raven-web-verify/`; that harness is
the model to copy if you need behaviour-level tests. Most sites do not: the
runtime is fixed and audited, so testing your own markup and bindings is
usually enough.
