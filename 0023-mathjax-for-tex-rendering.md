# ADR 0023: MathJax for TeX rendering, with HTML-emitting packages excluded

## Status

Accepted

## Context

Paper titles, abstracts, and note content contain LaTeX math, and raw markup is unacceptable for the target audience. The renderer choice (KaTeX vs MathJax) was left open for a long time (CONTEXT.md § TeX Rendering said "not yet committed"); it has in fact been decided and shipped, and the deliberate security carve-out in the shipped configuration is exactly the kind of thing a future contributor might "fix" without this record.

## Decision

**MathJax** (v4, `@mathjax/src`; originally `mathjax-full` v3), running the TeX → SVG pipeline in-process via `liteAdaptor` with `fontCache: "none"`, set up once in `src/lib/tex.tsx` and consumed everywhere through the `MathText` primitive. MathJax's TeX coverage is materially broader than KaTeX's, which matters when the input is arbitrary arXiv-authored TeX rather than site-authored markup.

**Several TeX input packages are deliberately excluded.** v3 filtered them out of `AllPackages`; v4 removed `AllPackages`, so `src/lib/tex.tsx` now imports each allowed extension explicitly and the exclusion is everything absent from that list:

- `html`, `texhtml` — emit raw HTML nodes (`\href`, `\style`, …): an XSS path when the input is untrusted paper metadata.
- `require` — `\require{html}` would re-enable the excluded `html` package at runtime, reopening the same hole. `autoload` is excluded with it (it exists to lazily `\require` extensions).
- `setoptions` — can likewise re-configure the input jax from within the input.
- `newcommand`, `configmacros`, `begingroup` — persist macro definitions across renders on the shared MathJax document, letting one paper's TeX redefine macros for every subsequent render.

v4 also splits rarely-used glyph ranges of the default font (`\mathcal`, `\mathfrak`, …) into files loaded on demand; `tex.tsx` maps that loading onto lazy Vite chunks and re-renders `MathText` when a chunk arrives, so the main bundle doesn't carry ~10MB of glyph data.

Do not re-enable any of these because "some paper's math doesn't render" — the first three are a security boundary (paper metadata is untrusted input fetched from the network), and the last two are a cross-render correctness boundary on the shared document.

Rendering is user-toggleable (`tex_rendering_enabled` setting); math is recognized by `$…$` / `$$…$$` delimiters with guards against currency-style stray `$`.

## Consequences

- Swapping renderers later is a real chore — `MathText` is woven through search results, library cards, paper detail, and note rendering — so this choice carries lock-in; that is accepted.
- TeX constructs that require the excluded packages render as plain text (the `mathHtml` failure path returns the raw string). This is by design, not a bug.

## References

- `src/lib/tex.tsx` — pipeline setup, extension allowlist, `MathText`
- CONTEXT.md § TeX Rendering
