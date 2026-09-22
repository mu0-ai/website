# mu0-website

Hugo site (PaperMod theme). Posts live in `content/posts/`, built with `hugo`.

## Math in posts

Math is rendered by KaTeX, loaded only when the front matter has `math: true`.
Delimiters are configured as Goldmark passthrough in `hugo.toml`: `\[ ... \]`
for display and `\( ... \)` for inline.

**Never start a line inside a math block with a bare `=` (or a bare `-`).**
Markdown reads a line containing only `=` as a setext heading underline. That
breaks the block before the passthrough extension can protect it: the preceding
line becomes an `<h1>`, backslashes are stripped, and `_` is reinterpreted as
emphasis (`\mathrm{Foo}_\phi` renders as `\mathrm{Foo}\phi`). Every formula from
that point on in the file loses its formatting.

Wrong:

```
\[
\mathrm{Attention}(Q,K,V)
=
\mathrm{softmax}\!\left(\frac{QK^\top}{\sqrt{d_k}}\right)V.
\]
```

Right — keep the operator at the end of the previous line:

```
\[
\mathrm{Attention}(Q,K,V) =
\mathrm{softmax}\!\left(\frac{QK^\top}{\sqrt{d_k}}\right)V.
\]
```

Check before committing a post with math:

```sh
grep -rn '^=\+$' content/posts/   # must return nothing
```

Then build and confirm the rendered post has no unexpected `<h1>` in
`<div class="post-content">` and that the `\[` / `\]` counts in the body match
the source.
