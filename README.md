# Thin Edge Specifications

View the specification with rendered mermaid diagram from [here](https://thin-edge.github.io/thin-edge.io-specs/) powered by GitHub Pages.

View the specifications online from [here](./src/SUMMARY.md)

# How to generate the documentation locally
This book is generated using [`mdbook`](https://lib.rs/crates/mdbook).

To generate the documentation from source, you will have to run:
1. `cargo install mdbook`
2. `cargo install mdbook-toc`       # inline Table of Contents
3. `cargo install mdbook-mermaid`   # add [mermaid.js](https://mermaid-js.github.io/mermaid/#/) support
4. `mdbook serve`

The documentation is then published on `http://localhost:3000/`.
