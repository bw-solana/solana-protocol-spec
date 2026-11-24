# Solana Protocol Specification

**[Read the current specification](https://bw-solana.github.io/solana-protocol-spec/)**

The specification is maintained as an
[mdBook](https://rust-lang.github.io/mdBook/). Its authoritative chapter order is
[src/SUMMARY.md](src/SUMMARY.md).

## Making Changes

Edit the chapter files under `src/`.

## Local Preview

Install PlantUML and Graphviz, then install the pinned mdBook tools. On
Debian/Ubuntu:

```sh
sudo apt-get update
sudo apt-get install --no-install-recommends -y plantuml graphviz
```

On macOS, use `brew install plantuml graphviz`. Then:

```text
cargo install mdbook --version 0.5.4 --locked
cargo install mdbook-plantuml --version 2.0.0 --locked --no-default-features
mdbook serve --open
```

For a static build, run `mdbook build`. The navigable book is written to `book/`, and
`book/print.html` contains the complete specification as one printable page.

Keep diagrams in fenced `plantuml` blocks in the chapter Markdown. The build renders
them locally as embedded SVG images, including in the printable specification; readers
do not need JavaScript or access to an external diagram service. Invalid diagrams fail
the build. Generated images are cached in `.mdbook-plantuml-cache/`, which is not
committed.

## Publication

GitHub Actions checks that the book builds for every pull request targeting `master`.
After a change is merged, the same workflow publishes the updated book to GitHub Pages
automatically. The generated `book/` directory is not committed.
