# Code quality tools

This repository coordinates two related Python projects:

- [`repos/code_quality_tools`](repos/code_quality_tools) — `archmap`, an
  architecture discovery and enforcement CLI with an AST-based symbol graph.
- [`repos/code_analysis`](repos/code_analysis) — reporting and evidence
  generation built on the `archmap` graph engine.

The projects are maintained as separate repositories and included here as
Git submodules. For normal use, clone the individual repository you need.

## Projects

### archmap

`archmap` scans Python repositories, creates versioned architecture manifests,
checks layer and dependency rules, and reports structural drift.

See [`repos/code_quality_tools/README.md`](repos/code_quality_tools/README.md)
for installation and usage.

### code-analysis

`code-analysis` turns the same symbol graph into JSON inventories, coupling
reports, and Markdown architecture evidence.

See [`repos/code_analysis/README.md`](repos/code_analysis/README.md) for
installation and usage.

## Status

These are Python 3.11+ experimental developer tools. They are useful for
architecture exploration and review, but are not presented as a complete
replacement for a type checker, linter, or runtime dependency tracer.

Both packages are licensed under the MIT License.
