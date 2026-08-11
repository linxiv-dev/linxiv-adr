# ADR 0023: MathJax for TeX rendering, with HTML-emitting packages excluded

## Status

Accepted

## Context

Paper titles, abstracts, and note content contain LaTeX math, and raw markup is unacceptable for the target audience. The renderer choice (KaTeX vs MathJax) was left open for a long time (CONTEXT.md § TeX Rendering said "not yet committed"); it has in fact been decided and shipped, and the deliberate security carve-out in the shipped configuration is exactly the kind of thing a future contributor might "fix" without this record.

## Decision

**MathJax** (`mathjax-full`), running the TeX → SVG pipeline in-process via `liteAdaptor` with `fontCache: "none"`, set up once in `src/lib/tex.tsx` and consumed everywhere through the `MathText` primitive. MathJax's TeX coverage is materially broader than KaTeX's, which matters when the input is arbitrary arXiv-authored TeX rather than site-authored markup.

**Five TeX input packages are deliberately excluded** from `AllPackages` (see `excludedPackages` in `src/lib/tex.tsx`):

- `html` — emits raw HTML nodes (`\href`, `\style`, …): an XSS path when the input is untrusted paper metadata.
- `require` — `\require{html}` would re-enable the excluded `html` package at runtime, reopening the same hole.
- `setoptions` — can likewise re-configure the input jax from within the input.
- `newcommand`, `configmacros` — persist macro definitions across renders on the shared MathJax document, letting one paper's TeX redefine macros for every subsequent render.

Do not re-enable any of these because "some paper's math doesn't render" — the first three are a security boundary (paper metadata is untrusted input fetched from the network), and the last two are a cross-render correctness boundary on the shared document.

Rendering is user-toggleable (`tex_rendering_enabled` setting); math is recognized by `$…$` / `$$…$$` delimiters with guards against currency-style stray `$`.

## Consequences

- Swapping renderers later is a real chore — `MathText` is woven through search results, library cards, paper detail, and note rendering — so this choice carries lock-in; that is accepted.
- TeX constructs that require the excluded packages render as plain text (the `mathHtml` failure path returns the raw string). This is by design, not a bug.

## References

- `src/lib/tex.tsx` — pipeline setup, `excludedPackages`, `MathText`
- CONTEXT.md § TeX Rendering
