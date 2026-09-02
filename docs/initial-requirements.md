# Architecture Mapper (`archmap`) — Initial Requirements

## 1. Purpose

`archmap` is a Python CLI tool that discovers and enforces the architectural
structure of a codebase. It produces a versioned, extensible YAML manifest
that declares what the codebase *is* — its layers, boundaries, and structural
invariants — and verifies that the codebase continues to conform to that
manifest over time.

## 2. Core Concept

The tool has two fundamental capabilities:

1. **Discovery** — scan an unfamiliar or changed codebase, invoke an LLM
   (Codewhale) to interpret its structure, and produce a proposed manifest.
2. **Enforcement** — read a committed manifest, check the current codebase
   against it, and report drift as structured violations.

The manifest format is *extensible*: the initial release defines a set of
invariant dimensions (layers, dependencies, required artifacts), but the
schema is designed so new dimensions can be added in future versions without
breaking existing files.

## 3. Architecture

```
archmap (Python CLI)
  ├── scanner/         # walks the filesystem, parses imports, gathers facts
  ├── schema/          # YAML manifest schema, validation, versioning
  ├── llm/             # pluggable LLM backends (Codewhale first)
  ├── commands/        # init, check, diff (and future modes)
  └── output/          # formatters: human-readable, JSON, SARIF (future)
```

### 3.1 Pluggable LLM Backend

The discovery path delegates to an LLM through a backend interface with a
single method: `generate(prompt: str) -> str`. Adding a new provider
(DeepSeek, OpenAI, local model) means implementing that interface — no
changes to the scanner, schema, or command layer.

The initial implementation uses a **file-based adapter for Codewhale**:

1. `archmap init` scans the repo and writes a structured prompt to
   `.archmap/prompt.yaml`.
2. The user (or a CI wrapper script) feeds that prompt to Codewhale and
   saves the response to `.archmap/response.yaml`.
3. `archmap init --from-response .archmap/response.yaml` ingests the LLM
   response, validates it against the schema, and writes
   `architecture.yaml`.

This keeps `archmap` decoupled from any specific Codewhale invocation
mechanism (CLI, API, SDK). As Codewhale's interface matures, a direct
adapter (subprocess or HTTP) can be added behind the same `generate()`
interface without changing any tool logic.

The structured prompt sent to the LLM contains:
- The directory tree (with depth and size limits)
- Key file summaries (imports, top-level definitions, docstrings)
- The target schema version

The LLM returns YAML conforming to the schema.

### 3.2 Manifest Format

- **80-column YAML** — human-readable in a terminal, clean diffs in PRs.
- **Versioned** — a `schema_version` field at the top so the tool can
  migrate or reject manifests from incompatible versions.
- **Extensible** — each major section (layers, dependencies, etc.) is a
  top-level key. Adding a new dimension means adding a new top-level key
  and a corresponding checker; existing keys are unmodified.

### 3.3 Invariant Dimensions (Initial Set)

These are the dimensions the first release of `archmap` discovers and checks.
The schema is designed so that adding a new dimension (e.g., naming
conventions, complexity thresholds, license requirements) is a non-breaking
change.

| Dimension | What it captures |
|---|---|
| `layers` | Named architectural layers, their filesystem paths, their roles |
| `dependencies` | Which layers may import from which other layers |
| `required` | Artifacts that must exist within each layer (e.g., `test/`, `__init__.py`) |
| `entry_points` | Build, test, and tooling commands or script paths |

## 4. Command Modes

### 4.1 `archmap init`

**Purpose**: Produce a proposed architecture manifest for a codebase that has
none (or needs a complete refresh).

**Behavior**:
1. Scan the repo root (directories, import graphs, key file signatures).
2. If `--from-response` is given: read the LLM response file, validate it
   against the schema, and write `architecture.yaml`. Skip to step 5.
3. Otherwise: build a structured prompt from the scan results and write it
   to `.archmap/prompt.yaml`.
4. Print instructions for feeding the prompt to Codewhale and re-running
   with `--from-response`.
5. Report any directories the LLM could not classify.
6. Exit 0 on success; non-zero if the LLM response is missing or invalid.

**Flags**:
- `--root PATH` — repo root (default: `.`)
- `--output PATH` — where to write the manifest (default: `architecture.yaml`)
- `--llm-backend NAME` — which backend to use (default: `codewhale`)
- `--from-response PATH` — ingest a pre-generated LLM response instead of
  producing a new prompt

### 4.2 `archmap check`

**Purpose**: Verify that the current codebase conforms to a committed
manifest.

**Behavior**:
1. Load and validate the manifest against its schema.
2. Scan the current codebase for the invariant dimensions present in the
   manifest.
3. Report every violation — structured output with file, line, layer,
   dimension, and message.
4. Exit 0 if no violations found; non-zero if any violation exists.

**Flags**:
- `--manifest PATH` — path to the manifest (default: `architecture.yaml`)
- `--format [text|json]` — output format (default: `text`)
- `--severity [error|warn]` — minimum severity to report (default: `error`)

### 4.3 `archmap diff`

**Purpose**: Show what changed between the manifest and the current codebase
— useful before deciding whether to re-init.

**Behavior**:
1. Load the committed manifest.
2. Scan the codebase.
3. Produce a structured diff:
   - **Added** — directories/modules on disk but not in the manifest.
   - **Removed** — manifest entries whose paths no longer exist on disk.
   - **Moved** — manifest entries whose content appears to have relocated
     (same role, new path).
   - **Changed** — manifest entries whose role or dependencies no longer
     match the code (LLM re-evaluation vs. original manifest).
4. Exit 0 if no differences; non-zero if differences exist.

**Flags**: Same as `check`, plus:
- `--include-llm-reeval` — run LLM re-evaluation for `Changed` detection
  (slower, off by default).

### 4.4 Future Modes (Out of Scope for v1)

| Mode | Description |
|---|---|
| `archmap validate` | Validate a manifest against its schema without scanning the repo |
| `archmap render` | Produce a Markdown architecture document from a manifest |
| `archmap watch` | File watcher that re-runs `check` on changes |
| `archmap migrate` | Upgrade an older manifest to the current schema version |

## 5. Extensibility Design

The schema and checker architecture must support adding new invariant
dimensions without breaking changes. The design principles:

1. **Schema versioning**: `schema_version` is an integer. The tool rejects
   manifests with a version higher than it supports. It may auto-migrate
   known older versions.
2. **Dimension isolation**: Each invariant dimension is an independent
   top-level YAML key with its own checker module. Adding `naming_rules`
   means adding a `naming_rules:` key to the schema and a
   `checkers/naming_rules.py` module — nothing else changes.
3. **Discovery is per-dimension**: The LLM prompt for `init` is built from
   the active dimensions. Adding a dimension means adding its prompt
   template; existing dimension prompts are unmodified.

## 6. Non-Functional Requirements

- **Language**: Python 3.11+ (matches the primary target ecosystem).
- **CLI framework**: `click` or `argparse` (decision deferred to
  implementation).
- **Dependencies**: Minimal. No heavy frameworks. Import graph analysis
  should use `ast` from the standard library where possible.
- **Performance**: `check` on a ~500-file Python repo should complete in
  under 5 seconds (excluding LLM calls; only `init` and `diff --include-llm-reeval`
  invoke the LLM).
- **Error handling**: The tool must never crash on malformed source code,
  missing files, or LLM failures. Degrade gracefully with clear error
  messages.
- **Exit codes**: 0 = clean, 1 = violations found, 2 = tool error (bad
  config, LLM failure, etc.).
- **Testing**: The scanner, schema validation, and check logic must be
  unit-testable without an LLM dependency. LLM integration tests may use a
  mock backend.

## 7. Resolved Decisions

1. **Codewhale invocation interface**: File-based adapter. `init` writes a
   prompt to `.archmap/prompt.yaml`; the user (or CI script) invokes
   Codewhale and saves output to `.archmap/response.yaml`; `init
   --from-response` ingests it. A direct adapter (subprocess/HTTP) can be
   added later behind the same `generate()` interface.

## 8. Resolved Design Decisions

1. **Manifest file location convention**: Single `architecture.yaml` at the
   repo root for v1. If `architecture.d/` exists in the future, files within
   it will be merged on top of the root manifest (opt-in overlay pattern for
   monorepos and large projects).
2. **Discovery depth**: Full depth by default — the scanner traverses all
   directories (subject to exclusion rules) and lets the LLM decide what is
   architecturally significant vs. internal organization. An optional
   `--max-depth N` flag on `init` provides a safety valve for repos large
   enough to exceed the LLM's context window.
3. **Exclusion patterns**: Reuse `.gitignore` as the primary exclusion list.
   The scanner also hardcodes a blacklist of obvious noise (`.git`,
   `__pycache__`, `.venv`, `dist`, `build`, `.egg-info`, `node_modules`,
   `.mypy_cache`, `.pytest_cache`, `.tox`). A separate `.archignore` file
   is a future escape hatch for repos that need to skip directories beyond
   what `.gitignore` covers.
4. **Multi-language support initial scope**: Python-first. The v1 scanner
   understands Python imports (via `ast`) and directory structure. TypeScript
   import parsing is a v2 feature. The schema itself is language-agnostic —
   layers, dependencies, and required artifacts apply regardless of language.

## 9. Imported Concepts from code-quality-coordinator

After comparative analysis of `code-quality-coordinator` (v0.6.0), five
concepts are slated for phased import into archmap. Phase A (evidence-rich
violation model) is complete as of v0.1.1. The remaining phases are:

2. **Symbol-level reference graph** (Phase B) — stable symbol IDs, typed
   reference edges (import/call/inherit/decorate), reachability analysis.
   Adapted from `code_coordinator/graph.py`.
3. **Architecture inventory and comparison** (Phase C) — symbol-level
   before/after snapshots enabling contract diffs and change categorization.
4. **Decision governance** (Phase D) — immutable acceptance proposals for
   known violations, evidence-fingerprint drift checks, append-only lifecycle
   audit.
5. **Confidence-calibrated severity** (Phase E) — reduced confidence for
   LLM-generated findings and dynamically-visible symbols.

Two CQC concepts are deferred: declarative tactic/workflow YAML
(over-engineering until checkers number 6+) and external tool plugins
(different product direction; revisit after phases A–E stabilize).