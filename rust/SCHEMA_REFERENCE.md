# SCHEMA_REFERENCE — Transmute optimization-workflow YAML schema

This document defines the schema used by `rust.yaml` so that future YAMLs grounded in other authoritative sources (e.g., Python, Go, C++ performance books) can adopt the same shape and remain interoperable with the same agent runtime.

The schema has seven top-level sections:

1. **Metadata** — `schema_version`, `spec_kind`, `language`
2. **`target`** — what the workflow operates on (toolchain, build commands)
3. **`contracts`** — what must be preserved (tests, benchmarks, invariants)
4. **`dimensions`** — what counts as improvement (loc, latency, allocations, ...)
5. **`techniques`** — the operationalizable patterns from the source
6. **`workflow`** — pre-flight, iteration loop, termination
7. **`safety`** — bounds and escape hatches
8. **`provenance`** — what authoritative source grounds the spec

Each section below documents fields, types, and acceptable values.

---

## Metadata

```yaml
schema_version: "0.1.0"          # SemVer of this schema
spec_kind: "transmute_optimization_workflow"
language: "rust"                 # target programming language
```

Increment minor version when adding fields; major version when breaking field semantics.

---

## §1 `target`

```yaml
target:
  language: string                # must match top-level `language`
  toolchain_version: string       # SemVer range, e.g. ">=1.70"
  toolchain_channel: string       # "stable" | "beta" | "nightly"
  build_command: string           # release-mode build invocation
  test_command: string            # test invocation
  bench_command: string           # optional, but recommended
  cargo_manifest: string          # path to root manifest (rust-specific; "package.json", "go.mod", etc. in other YAMLs)
  workspace_root: string          # filesystem path; agent does not modify outside this
```

For non-Rust YAMLs: `cargo_manifest` becomes the language-appropriate root manifest. Keep the field name `manifest` if you prefer language-agnostic naming.

---

## §2 `contracts`

Contracts are what optimization MUST preserve. There are four kinds:

```yaml
contracts:

  tests:
    - name: string
      command: string
      timeout: duration           # "15m"
      on_failure: enum            # "discard_rewrite" | "abort_run" | "require_human_review"
      required: bool

  benchmarks:
    - name: string
      command: string
      parser: string              # see §parsers below
      baseline_file: path
      regression_tolerance_pct: float
      timeout: duration
      optional_if_missing: bool

  invariants:
    - name: string
      verification_command: string
      on_failure: enum
      optional: bool

  behavior:
    - name: string
      verification_command: string
      on_failure: enum
      optional: bool
```

**`on_failure` values:**

- `discard_rewrite` — revert the iteration's change, continue to next iteration
- `discard_rewrite_if_regression` — same, but only if the result is measurably worse
- `discard_rewrite_if_regression_exceeds_tolerance` — same, with tolerance from `regression_tolerance_pct`
- `abort_run` — stop the entire run; produce a partial PR or no PR
- `require_human_review` — pause and surface to operator

`invariants` carry codebase-specific properties pulled from the methodology substrate (Bible, ADRs). The agent reads them at pre-flight and includes them in the contracts loop. Operators should populate them.

---

## §3 `dimensions`

Dimensions are what counts as improvement. Each is a measurable quantity that the agent can compute before and after each rewrite.

```yaml
dimensions:
  - id: string                    # snake_case, unique
    description: string
    measure_command: string       # shell command that produces measurement
    parser: string                # name of parser to apply to command output
    metric_path: string           # JSONPath or similar within parser output (optional)
    improvement: enum             # "decrease" | "increase"
    weight: float                 # 0..1, sums across dims should equal 1.0
    requires_workload: bool       # whether dimension needs a representative workload
    requires_benchmarks: bool     # whether dimension needs benchmark suite
```

**Composite scoring** combines per-dimension deltas:

```yaml
composite_scoring:
  method: "weighted_sum_of_normalized_deltas"
  improvement_threshold: float    # composite must exceed this to commit
  regression_tolerance_per_dim: float
```

A dimension regression beyond `regression_tolerance_per_dim` blocks the commit regardless of composite. This prevents techniques from trading away too much on one axis.

The six dimensions in `rust.yaml` (loc, allocations, peak_memory, latency_p50, binary_size, compile_time) are language-agnostic enough that other YAMLs can reuse the same set. Adjust `measure_command` and `parser` per language.

---

## §4 `techniques`

The heart of the schema. Each technique is one operationalizable pattern from the source.

```yaml
techniques:
  - id: string                              # snake_case, globally unique
    source_chapter: string                  # verbatim chapter title from source
    source_url: string                      # fragment URL to specific section
    detect:
      method: enum                          # see §detection-methods below
      pattern: string | array               # method-specific
      # ... method-specific subfields
    action:
      description: string                   # human-readable
      change_type: enum                     # see §action-types below
      # ... type-specific subfields
    verify:
      method: enum                          # see §verify-methods below
      dimensions: array<string>             # which dimensions to recheck
      expected_change: enum                 # "decrease" | "increase"
    targets: array<string>                  # dimensions this technique improves
    confidence: enum                        # "high" | "medium" | "low"
    tradeoffs: array<string>                # dimension/property names worsened
    requires_human_consent: bool            # surface before applying
    requires_workload: bool                 # technique needs profiling data
    requires_nightly: bool                  # technique requires nightly toolchain
    once_only: bool                         # apply at most once per run
    fallback_technique: string              # id of alternative if this fails
    notes: string                           # free-form explanation
```

### Detection methods (`detect.method`)

| Method | Description |
|---|---|
| `grep_with_context` | regex match, optionally filtered by hot path or other context |
| `ast_pattern` | structural pattern over AST (rust-analyzer, tree-sitter, or rustc query) |
| `type_signature_search` | match on type usages, parameterized |
| `cargo_toml_scan` | inspect Cargo.toml profile/dependency state |
| `config_toml_scan` | inspect .cargo/config.toml |
| `clippy_run` | invoke clippy, parse diagnostics |
| `clippy_lint` | single clippy lint as detector |
| `rust_source_scan` | repository-wide source-text scan |
| `rustc_print_type_sizes` | invoke nightly rustc with `-Zprint-type-sizes` |
| `cargo_llvm_lines` | invoke cargo-llvm-lines |
| `rustc_macro_stats` | invoke rustc with `-Zmacro-stats` |
| `ad_hoc_profile` | detection depends on prior profiling data |
| `command_substring_check` | inspect another field of the YAML for a substring |
| `compound` | conjunction/disjunction of sub-checks |
| `platform_check` | gate on OS / architecture |

Each method has its own subfields. The `compound` method is the escape hatch:

```yaml
detect:
  method: "compound"
  checks:
    - method: "..."
      ...
    - method: "..."
      ...
  combinator: "all" | "any"   # defaults to "all"
```

### Action types (`action.change_type`)

| Type | Description |
|---|---|
| `ast_modification` | source-level code rewrite |
| `cargo_toml_edit` | edit Cargo.toml |
| `cargo_config_edit` | edit .cargo/config.toml |
| `clippy_toml_edit` | edit clippy.toml |
| `import_and_type_substitution` | add `use` + rename type |
| `lint_directed_rewrite` | apply clippy's `MachineApplicable` suggestion |
| `compound` | sequence of sub-actions |
| `configuration_check` | verify a config; do not modify |
| `report_only` | surface info to operator; no auto-rewrite |

For `ast_modification` and `compound` types, a `diff_template` field with sample before/after is recommended. The agent uses it as a model for the rewrite shape.

### Verify methods (`verify.method`)

| Method | Description |
|---|---|
| `dhat_allocation_count` | re-run DHAT, compare totals |
| `criterion_benchmark_for_hot_path` | rerun criterion bench tied to changed code |
| `rebench_after_change` | rerun benchmarks |
| `remeasure_after_change` | rerun dimension measurements |
| `tests_pass_plus_no_regression` | tests green + no dim regression |
| `tautological_after_apply` | no measurement needed (e.g. preventive guards) |
| `asm_diff_check` | inspect generated asm for change |
| `cargo_llvm_lines` | rerun cargo-llvm-lines |
| `compile_time_measurement` | time a clean build |
| `compound` | conjunction of sub-checks |

### Confidence ratings

The `confidence` field is the single most important calibration knob. Definitions:

- **`high`** — detection is unambiguous, action is mechanical, the change is verifiable as an improvement on at least one targeted dimension with very low probability of correctness regression. **Example:** `heap_clone_from_over_assign_clone`. Detection is a single AST pattern; the action is a one-line rewrite; behavior is preserved; allocations measurably decrease.

- **`medium`** — detection has noise (false positives possible), or the action requires light judgment, or the improvement is conditional on workload shape. The technique is still safe to attempt, but a meaningful fraction of attempts will fail validation and be discarded. **Example:** `heap_smallvec_for_consistently_short_vecs`. Detection requires profiling data; the action changes type signature widely; SmallVec is slightly slower for normal ops so improvement isn't guaranteed.

- **`low`** — detection requires significant judgment, or the action has architectural implications, or the improvement is measurable only at the assembly level. Should typically be gated by `requires_human_consent`. **Example:** `wrap_combine_co_accessed_mutexes`. Changes lock granularity; affects API.

### Tradeoffs and "non-measured" dimensions

Some tradeoffs are not directly measured dimensions but matter to operators:

- `portability` (target-cpu=native)
- `semantics` (panic=abort, swap_remove)
- `debuggability` (strip=symbols, debug=false)
- `hashdos_resistance` (FxHash for any non-trusted key source)
- `readability` (workhorse collection, filter_map fusion)
- `correctness_risk` (smaller integer indices, debug_assert! conversion)
- `api_surface` (lock combining, return-impl-Iterator on pub fn)

These appear in `tradeoffs` arrays even though they're not in `dimensions`. The agent must surface them to the operator when `requires_human_consent: true`.

---

## §5 `workflow`

The workflow describes how techniques are applied in time.

### Pre-flight

```yaml
workflow:
  pre_flight:
    - step: string                # snake_case identifier
      # ... step-specific fields
```

Each step is sequential. Standard steps:

- `read_methodology_substrate` — absorb invariants from project docs
- `verify_baseline_green` — fail fast if tests don't pass
- `capture_baseline` — measure every dimension
- `identify_hot_paths` — flamegraph / samply
- `identify_alloc_hotspots` — DHAT
- `snapshot_type_sizes` — `-Zprint-type-sizes` (nightly)

### Iteration

```yaml
workflow:
  iteration:
    max_iterations: int
    bounded_by: ref               # reference into safety
    convergence:
      rolling_window: int
      composite_threshold: float
      action_on_convergence: string
    per_iteration:
      - step: ...
```

The iteration loop runs at most `max_iterations` times. Each iteration is one (technique, instance) pair. Convergence stops the loop early if the rolling window of composite deltas drops below threshold.

Standard `per_iteration` steps:

1. `select_technique_instance` — prioritize, exclude exhausted
2. `conflict_check` — refuse overlapping changes
3. `consent_check` — surface to operator if required
4. `apply_change` — write the diff
5. `validate_contracts` — tests, behavior, invariants, benchmarks
6. `measure_dimensions` — quantify the change
7. `compute_delta_and_composite` — composite score
8. `commit_or_discard` — final decision

### Termination

```yaml
workflow:
  termination:
    triggers: array<string>
    on_terminate: array<string>
    pull_request:
      title_template: string
      body_template: string
```

---

## §6 `safety`

```yaml
safety:
  max_wall_clock: duration
  max_token_budget: string
  max_files_modified_per_iteration: int
  max_lines_changed_per_iteration: int
  branch_protection_required: bool
  human_review_required: bool
  abort_if_consecutive_failures: int
  preserve_original_branch: bool
  halt_on: array<condition>
  forbidden_changes: array<rule>
```

The `forbidden_changes` list is hard-coded constraints the agent never violates regardless of technique. For Rust, this includes "no new unsafe blocks." For other languages, adjust accordingly (e.g., "no `eval`" in Python, "no `unsafe.Pointer` introductions" in Go).

---

## §7 `provenance`

```yaml
provenance:
  source_book: string
  source_author: string
  source_url: string
  rule: string                    # the no-novel-claims discipline
  scope_excluded:
    - chapter: string
      reason: string
```

This block is what makes the spec defensible. Every technique traces to a section of the named source. The `scope_excluded` list documents which parts of the source were deliberately not operationalized.

---

## How to author a new YAML for a different language

The schema is language-agnostic. To produce e.g. `python.yaml`:

1. **Pick the authoritative source.** Examples: *High Performance Python* by Gorelick & Ozsvald; CPython's `Doc/howto/perf_profiling.rst`; the `cProfile` documentation.

2. **Walk the source section by section.** For each technique, apply the four operationalizability criteria (mechanical detection, measurable verification, concrete action, automated validation).

3. **Map each accepted technique to a `techniques` entry.** Use the same fields and the same confidence definitions.

4. **Translate the detection and action methods** to language-appropriate tools:
   - AST tooling: `ast` / `libcst` for Python; `gopls` for Go; `clang-tidy` for C++
   - Profilers: `py-spy` / `cProfile` for Python; `pprof` for Go; `perf` for C++
   - Allocation profilers: `memray` / `tracemalloc` for Python; `pprof -alloc_space` for Go; Valgrind massif for C++

5. **Reuse dimensions where possible.** loc, latency, peak_memory, binary/wheel size all translate. allocations is rust-specific; in Python the analog is `tracemalloc` peak.

6. **Match the workflow.** The pre-flight → iteration → termination shape is universal. Substitute `cargo` with the language's build tool.

7. **Keep the no-novel-claims rule.** If a technique seems valuable but isn't in your chosen source, leave it out or move it to a separate YAML grounded in a different source.

---

## Versioning policy

- **PATCH** (`0.1.x`) — add techniques, refine confidence ratings, add dimension parsers
- **MINOR** (`0.x.0`) — add new fields, add new detection/action/verify methods, change pre-flight or iteration steps non-breakingly
- **MAJOR** (`x.0.0`) — break field semantics, remove fields, change workflow structure incompatibly

Agents reading this spec should fail loudly on a major-version mismatch between the spec and what they understand.
