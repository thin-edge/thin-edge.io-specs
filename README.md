# Thin Edge Specifications

You can [read the specification directly from the source](./src/SUMMARY.md).

## How to generate the documentation
This book is generated using [`mdbook`](https://lib.rs/crates/mdbook).

To generate the documentation from source, you will have to run:
1) `cargo install mdbook`
2) `cargo install mdbook-toc`       # inline Table of Contents
2) `cargo install mdbook-mermaid`   # add [mermaid.js](https://mermaid-js.github.io/mermaid/#/) support
2) `mdbook serve`

The documentation is then published on `http://localhost:3000/`.
