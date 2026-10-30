# Solana Protocol Specification

**[Read the current specification](https://bw-solana.github.io/solana-protocol-spec/)**

The specification is maintained as an
[mdBook](https://rust-lang.github.io/mdBook/). Its authoritative chapter order is
[src/SUMMARY.md](src/SUMMARY.md).

## Making Changes

Edit the chapter files under `src/`.

## Local Preview

Install mdBook, then serve the specification with live reload:

```text
cargo install mdbook --version 0.5.4 --locked
mdbook serve --open
```

For a static build, run `mdbook build`. The navigable book is written to `book/`, and
`book/print.html` contains the complete specification as one printable page.

## Publication

GitHub Actions checks that the book builds for every pull request targeting `master`.
After a change is merged, the same workflow publishes the updated book to GitHub Pages
automatically. The generated `book/` directory is not committed.
