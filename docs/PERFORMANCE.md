# Performance, measured

Numbers for raven-web v0.10.0. Everything here was measured, not estimated. The
method is at the bottom, including what these numbers do **not** show.

## Payload

The runtime is a fixed constant: the same ~10.8 KB (3.6 KB gzipped) whatever the
page does. Only the config, which is the page's own bindings and initial state,
grows.

| page | app.js | gzipped | of which config |
|---|---|---|---|
| minimal (`examples/landing.rv`) | 10.9 KB | **3.6 KB** | 51 B |
| client router, 4 routes + guard | 11.1 KB | **3.7 KB** | 274 B |
| components, 9 instances | 11.9 KB | **3.9 KB** | 1.1 KB |
| validated form | 12.1 KB | **3.9 KB** | 1.2 KB |
| 200 components + 500 row list | 57.8 KB | 11.2 KB | 47 KB |

For scale, `react` + `react-dom` are about 45 KB gzipped before you add a router,
a form library, or your own components. That comparison is real but it is not
like-for-like: React runs your code in the browser, and raven-web does not (see
`FRONTEND-GAP-ANALYSIS.md`). raven-web is small **because** it does less.

Content does not wait on any of it. The HTML is complete at build time, and with
`signal_of` even a data-driven list is in the document on first paint, so the
page renders before app.js is parsed.

## Cost of a state write

The dependency graph is resolved at build time, so a write wakes only the
bindings that read that key. Measured on a deliberately hostile page: 200
component instances (401 bindings) plus a 500 row list.

| action | bindings re-applied | document scans |
|---|---|---|
| click one counter (2 bindings read it) | **2 of 401** | **0** |
| write a key nothing binds to | **0** | **0** |
| unrelated write, with a 500 row list on the page | **0 rows touched** | **0** |

Two properties matter here and both hold:

- Work is proportional to what reads the key, not to page size. A 500 row list
  costs nothing when you type somewhere else.
- Lists reconcile by key. Across a refetch of the same ids, rows keep their DOM
  identity: an `<input>` inside a row keeps the text you typed, and removing one
  row leaves the survivors' nodes untouched.

## The caching fix this measurement found

Until v0.10.0 every binding ran `document.querySelectorAll` on every write it
took part in, so a targeted update still paid for a document-wide scan per
binding. raven-web's own marks are unique and live as long as the page, so they
are now resolved once and cached. An author's selector is still resolved every
time, because a list may rebuild the rows it matches.

Measured on the 200-component + 500-row page (jsdom, so read these as ratios,
not as browser timings):

| | before | after |
|---|---|---|
| boot | 2255 ms | **721 ms** |
| 100 clicks | 937 ms | **18 ms** |
| document scans per click | 2 | **0** |

## Method, and what this does not show

Measured by loading the real emitted `index.html` and `app.js` into jsdom and
counting real DOM operations through instrumented `querySelectorAll`,
`querySelector`, `insertBefore`, `removeChild`, and `cloneNode`. The page is
`examples/bench.rv`.

- **The operation counts are the real result.** They are a property of the
  algorithm and hold in any DOM.
- **The millisecond figures are jsdom, not a browser.** jsdom's
  `querySelectorAll` costs about 0.69 ms on this page, roughly a thousand times
  a browser's. Treat the timings as before/after ratios on identical code, which
  is what they are, and not as what a user would feel.
- **Not measured:** real browser paint and layout, memory, or behaviour on a
  slow device. No comparison against React was run; the size figure above is
  from React's published bundle, not from a benchmark of the two doing the same
  job.
