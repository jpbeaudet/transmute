# OPERATIONALIZATION_NOTES — Rust Performance Book → rust.yaml

**Source:** *The Rust Performance Book* by Nicholas Nethercote
**URL:** https://nnethercote.github.io/perf-book/

Companion to `rust.yaml`. This document records, chapter by chapter, which techniques operationalized cleanly, which operationalized partially, which were deferred to human judgment, and why. The criteria are those defined in `TRANSMUTE_RUST_YAML_OPERATIONALIZATION_BIBLE.md` §3:

1. **Detection is mechanical** (grep / AST / tool output)
2. **Verification is measurable** (profiler / benchmark / test)
3. **Action is concrete** (specific code change, not architectural)
4. **Validation is automated** (cargo test / bench)

When all four hold, a technique enters `rust.yaml`. When one or more fails, the technique is excluded with a note here.

---

## Summary table

| Chapter | Operationalized | Partial | Deferred | Notes |
|---|---|---|---|---|
| Introduction | 0 | 0 | 0 | Meta — no techniques. |
| Benchmarking | 0 | 0 | 0 | Methodology — encoded in `contracts.benchmarks` and `dimensions`. |
| Build Configuration | 11 | 0 | 1 | PGO deferred (toolchain-heavy setup). Cranelift backend excluded (nightly-only, narrow audience). |
| Linting | 2 | 0 | 0 | clippy::perf application + disallowed_types preventive guard. |
| Profiling | 0 | 0 | 0 | Methodology — encoded in `workflow.pre_flight.identify_hot_paths`. |
| Inlining | 2 | 1 | 0 | "Harder cases" (split fn into inlined/uninlined twins) partially deferred — fragile detection. |
| Hashing | 2 | 1 | 1 | FxHash + nohash mapped. Byte-wise hashing (zerocopy/bytemuck) deferred — type-property dependent. ahash/fnv covered by FxHash branch. |
| Heap Allocations | 11 | 1 | 2 | Box/Rc/Arc are passive types (no action). `to_owned` partially deferred (overlap with clone). Profiling sub-section is methodology. |
| Type Sizes | 5 | 0 | 1 | Field ordering excluded (compiler already does it). |
| Standard Library Types | 5 | 0 | 0 | All five concrete techniques operationalized; Rc::make_mut surfaced but rare enough to defer. |
| Iterators | 6 | 0 | 0 | All operationalizable techniques captured. |
| Bounds Checks | 2 | 1 | 1 | get_unchecked excluded (introduces unsafe). |
| I/O | 4 | 0 | 0 | All four operationalizable. |
| Logging and Debugging | 2 | 0 | 0 | Both. |
| Wrapper Types | 1 | 0 | 0 | Lock-combining; low confidence due to architectural risk. |
| Machine Code | 0 | 0 | All | Exploratory only — human judgment essential. |
| Parallelism | 0 | 0 | All | Architectural; outside scope. |
| General Tips | 0 | 0 | All | Heuristics. |
| Compile Times | 2 | 0 | Several | factor-non-generic-inner + macro-stats report. cargo-expand/cargo-show-asm are inspection tools, not techniques. |

**Totals: 55 techniques mapped, 4 partial, ~8 explicitly deferred. 4 chapters fully excluded by design.**

---

## Chapter-by-chapter detail

### Introduction
Meta only. No techniques.

### Benchmarking
The chapter is methodology: choose tools, choose workloads, choose metrics. This information is encoded in:

- `contracts.benchmarks` — which benchmark suite gates rewrites
- `dimensions` — what metrics count as improvement
- `workflow.pre_flight.capture_baseline` — when measurement happens

The chapter's tool list (Criterion, Divan, Hyperfine, Bencher, Gungraun) is reflected in the parser options inside `dimensions[].parser`. Criterion is the default because it has wide adoption and JSON output suitable for parsing.

### Build Configuration

**Mapped (11):**

| Technique | id | confidence |
|---|---|---|
| Use release builds | `bc_use_release_build` | high |
| codegen-units = 1 | `bc_set_codegen_units_one` | high |
| Fat LTO | `bc_enable_fat_lto` | high |
| Thin LTO (fallback) | `bc_enable_thin_lto` | high |
| mimalloc allocator | `bc_alternative_allocator_mimalloc` | medium |
| jemalloc allocator (Linux/macOS) | `bc_alternative_allocator_jemalloc` | medium |
| target-cpu=native | `bc_target_cpu_native` | high (gated by consent) |
| panic = abort | `bc_panic_abort` | medium (semantics) |
| strip = symbols | `bc_strip_symbols` | high |
| Faster linker (lld/mold) | `bc_faster_linker_lld` | high |
| Disable dev debuginfo | `bc_disable_dev_debuginfo` | high |

The book also mentions `opt-level = "z" | "s"` (size). Captured implicitly in `bc_strip_symbols` cluster; not given a separate ID because the user's choice between speed and size is a configuration-policy decision and the YAML targets runtime speed by default. Add as a custom invocation by setting a dimension weight that prioritizes `binary_size`.

**Deferred:**

- **PGO (Profile-guided optimization):** Book describes it as "an advanced technique that takes some effort to set up." Requires a two-phase build with workload between. The workflow doesn't currently model two-phase builds, and the cargo-pgo tooling adds external dependency. Defer to a v0.2 extension that introduces a `pgo` workflow phase.
- **Cranelift backend:** Nightly-only, recommended only for dev builds, narrow audience. Excluded.
- **Custom profiles:** Project-specific; not a technique.
- **Parallel front-end (`-Zthreads=N`):** Nightly-only and effect is highly variable. Excluded.

### Linting

**Mapped (2):**

- `lint_apply_clippy_perf_group` — run clippy with `-W clippy::perf` and apply each diagnostic as its own sub-iteration. This is high-confidence because clippy produces structured JSON diagnostics with suggested rewrites.
- `lint_disallow_std_hashmap_if_fxhash_adopted` — preventive: once FxHash is adopted somewhere, add a `clippy.toml` `disallowed-types` entry so the codebase doesn't accidentally reintroduce the default hasher.

The book is explicit that clippy detection is preferred over manual detection: "Given that automated detection of problems is preferable to manual detection, the rest of this book will not mention performance problems that Clippy detects by default." This justifies clippy as a first-pass technique in the workflow.

### Profiling

Methodology, not techniques. Encoded in:

- `workflow.pre_flight.identify_hot_paths` (flamegraph primary, samply fallback)
- `workflow.pre_flight.identify_alloc_hotspots` (DHAT)
- `workflow.pre_flight.snapshot_type_sizes` (`-Zprint-type-sizes`)
- `dimensions[].measure_command` — picks per-dimension profiler

The chapter's discussion of frame pointers and symbol demangling is configuration that profilers need to function; that surfaces through `workflow.pre_flight.identify_hot_paths.fallback`.

### Inlining

**Mapped (2):**

- `inline_small_hot_function` — add `#[inline]` to small functions on hot paths
- `inline_cold_attribute` — add `#[cold]` to cold-path functions

**Partial:**

- **Harder cases (split fn into always-inlined + never-inlined variants).** The book gives an example PR (rustc#53513). The pattern is operationalizable in principle — detect a function with multiple call sites where Cachegrind shows one call site contributes >80% of execution — but the detection is fragile (needs per-call-site profiling, not per-function) and the rewrite reshapes the public API. Excluded from initial YAML; the operator can run this manually when warranted.

### Hashing

**Mapped (2):**

- `hash_fxhash_for_integer_keys` — high-confidence drop-in for integer-keyed maps
- `hash_nohash_for_random_integer_newtype` — low-confidence (distribution dependent), gated by consent

**Partial:**

- **General FxHash replacement (any HashMap, not just integer-keyed).** The book is supportive but explicit that HashDoS resistance is the tradeoff. Captured as the `requires_human_consent` gate on the technique. The agent surfaces the use site (e.g. "HashMap keyed on String received from network") for operator judgment; not auto-applied.

**Deferred:**

- **Byte-wise hashing via `zerocopy` / `bytemuck` `#[derive(ByteHash)]`.** Book describes it as "an advanced technique, and the performance effects are highly dependent on the hash function and the exact structure of the types being hashed. Measure carefully." Type-property dependent: requires no padding bytes, which is a layout property. Excluded from YAML.
- **`fnv` vs `ahash` choice.** Book documents that an attempt to switch rustc from fxhash to ahash *regressed* by 1-4%, and switching back to default hasher regressed 4-84%. So FxHash is the clear book-endorsed default. Other hashers are alternatives the operator can swap in by editing the technique's `action.steps` if they want to A/B them; not auto-applied.

### Heap Allocations (the largest chapter — most operationalizable surface)

**Mapped (11):**

| Technique | id | confidence |
|---|---|---|
| Vec::with_capacity for known loop bound | `heap_vec_with_capacity_for_known_loop_bound` | high |
| String::with_capacity for known size | `heap_string_with_capacity_for_known_size` | high |
| HashMap::with_capacity for known size | `heap_hashmap_with_capacity_for_known_size` | high |
| SmallVec for consistently short Vecs | `heap_smallvec_for_consistently_short_vecs` | medium |
| ArrayVec when max length known | `heap_arrayvec_when_max_length_known` | medium |
| smartstring for short Strings | `heap_smartstring_for_short_strings` | medium |
| Redundant .clone() removal on hot path | `heap_redundant_clone_on_hot_path` | medium |
| clone_from over `a = b.clone()` | `heap_clone_from_over_assign_clone` | high |
| Cow for mixed static/dynamic strings | `heap_cow_for_mixed_static_dynamic_strings` | medium |
| Workhorse collection hoisted out of loop | `heap_workhorse_collection_outside_loop` | medium |
| BufRead::read_line over .lines() | `heap_bufread_read_line_over_lines` | high |

**Partial:**

- **`to_owned` reduction.** Largely subsumed by the `.clone()` technique. Where `to_owned` produces an allocation that an explicit borrow could avoid, the same detection and action as `heap_redundant_clone_on_hot_path` apply. Did not add a separate technique to avoid double-counting.

**Deferred:**

- **Box / Rc / Arc** descriptions in the chapter are passive — they explain when these types help. There's no concrete "do X" action attached to them in the chapter (the action is "use them when appropriate," which is human judgment). The relevant *action* for Rc/Arc — boxing an enum variant — appears in the Type Sizes chapter and is captured there as `type_box_large_enum_variant`.
- **"Profiling (heap)" sub-section.** Methodology, not a technique.
- **"Using an Alternative Allocator" sub-section.** Cross-referenced to Build Configuration; captured there as `bc_alternative_allocator_*`.
- **"Avoiding Regressions" (dhat heap testing).** This is a CI infrastructure recommendation, not a code rewrite. Surface to operator separately; not in YAML.

### Type Sizes

**Mapped (5):**

| Technique | id | confidence |
|---|---|---|
| Box large enum variant | `type_box_large_enum_variant` | medium |
| Boxed slice for immutable Vec | `type_boxed_slice_for_immutable_vec` | medium |
| ThinVec for oft-empty Vec | `type_thinvec_for_oft_empty_vec` | low |
| Smaller integer for bounded indices | `type_smaller_integer_for_bounded_indices` | low (consent) |
| static_assertions for hot type | `type_static_assertion_for_hot_type` | high (preventive) |

**Deferred:**

- **Field ordering.** Book explicitly says: "The Rust compiler automatically sorts the fields in struct and enums to minimize their sizes (unless the `#[repr(C)]` attribute is specified), so you do not have to worry about field ordering." So no action needed unless `#[repr(C)]` is in play, which is a layout-correctness commitment outside this YAML's scope.

### Standard Library Types

**Mapped (5):**

| Technique | id | confidence |
|---|---|---|
| vec![0; n] for zero-init | `stdlib_vec_zero_init_macro` | high |
| swap_remove when order doesn't matter | `stdlib_swap_remove_when_order_irrelevant` | medium (consent) |
| Vec::retain over filter+rebuild | `stdlib_vec_retain_over_filter_rebuild` | medium |
| Lazy fallback for Option/Result | `stdlib_lazy_fallback_for_option_result` | high |
| parking_lot Mutex | `stdlib_parking_lot_mutex` | medium |

Note on `Rc::make_mut` / `Arc::make_mut`: book says "they are not needed often." The detection pattern would be "code that clones an Rc and then mutates the clone," which is rare in practice. Excluded as not-cost-effective to operationalize.

### Iterators

**Mapped (6):**

| Technique | id | confidence |
|---|---|---|
| Avoid collect-then-iterate | `iter_avoid_collect_then_iterate` | high |
| Return impl Iterator over Vec | `iter_return_impl_iterator_over_vec` | medium |
| filter+map → filter_map | `iter_filter_map_fusion` | medium |
| chunks → chunks_exact when divisible | `iter_chunks_exact_when_divisible` | medium |
| `.iter().copied()` for Copy types | `iter_copied_for_small_copy_types` | low |
| Implement size_hint on custom iterator | `iter_implement_size_hint` | medium |

All six chapters' techniques captured. The book's `extend` discussion is covered as an alternative form of the collect technique (the action is the same: don't build an intermediate Vec).

### Bounds Checks

**Mapped (2):**

- `bounds_indexed_loop_to_iterator` — `for i in 0..v.len() { v[i] }` → `for x in &v`
- `bounds_slice_lift_out_of_loop` — bind slice before loop, index into slice

**Partial:**

- **Add assertions on the ranges of index variables.** Operationalizable in principle, but the book's example (asserting `i < v.len()` at the top of a function) requires identifying which assertion would actually lift bounds checks — this is LLVM-IR-dependent and often produces no measurable change. Detection would need to look at generated assembly. Deferred to manual application after profiling identifies a specific bounds-check hotspot.

**Deferred:**

- **`get_unchecked` / `get_unchecked_mut`.** Introduces `unsafe`. The YAML's safety section explicitly forbids introducing new `unsafe` blocks (`safety.forbidden_changes`). The book itself frames these as "As a last resort."

### I/O

**Mapped (4):**

| Technique | id | confidence |
|---|---|---|
| Lock stdout in hot loop | `io_lock_stdout_in_hot_loop` | high |
| Wrap File writer in BufWriter | `io_wrap_file_writer_in_bufwriter` | high |
| Wrap File reader in BufReader | `io_wrap_file_reader_in_bufreader` | high |
| Raw bytes when no UTF-8 needed | `io_raw_bytes_when_no_utf8_needed` | low (consent) |

The chapter's "Reading Lines from a File" section is captured under Heap Allocations as `heap_bufread_read_line_over_lines` (cross-referenced both places in the book).

### Logging and Debugging

**Mapped (2):**

- `log_guard_expensive_log_args` — wrap expensive log argument computation in `log_enabled!` guard
- `log_debug_assert_for_hot_non_safety_assert` — promote hot `assert!` to `debug_assert!` when not safety-critical

The second is `requires_human_consent: true` because the safety-criticality determination cannot be made mechanically. The agent surfaces the assertion and waits for operator judgment.

### Wrapper Types

**Mapped (1):**

- `wrap_combine_co_accessed_mutexes` — merge co-accessed `Mutex`/`RefCell` fields into a single wrapper

Confidence is low because lock granularity is an architectural concern. Always gated by consent.

### Machine Code

**Deferred entirely.** The chapter explicitly frames machine-code inspection as exploratory: "When you have a small piece of very hot code it may be worth inspecting the generated machine code to see if it has any inefficiencies." The decisions taken from inspection — manual SIMD with `core::arch`, restructuring loops for vectorization — are all human-judgment. The agent can offer `cargo-show-asm` output as a diagnostic but does not act on it.

### Parallelism

**Deferred entirely.** The chapter itself says "an in-depth treatment of parallelism is beyond the scope of this book." Introducing rayon or crossbeam is architectural redesign, not local rewrite.

### General Tips

**Deferred entirely.** Heuristics: "make hot functions faster or call them less often," "lazy/on-demand computations are often a win," "specially handle 0/1/2 element cases." These are programmer wisdom, not detection-and-rewrite patterns. The agent can apply them when invited but doesn't operationalize them as YAML techniques.

### Compile Times

**Mapped (2):**

- `compile_factor_non_generic_inner_fn` — extract non-generic body from generic function
- `compile_macro_stats_report` — report-only: surface top macros by expansion size

**Deferred:**

- **`cargo --timings` Gantt chart interpretation.** Visualization tool; output requires human interpretation.
- **Replacing `Option::map`/`Result::map_err` chains with `match`.** Book says "the effects of these sorts of changes on compile times will usually be small, though occasionally they can be large." Low-yield rewrite; defer.

---

## Open questions, resolved

The bible's §8 listed six design questions. The YAML resolves them as follows:

**Q1 — Granularity of technique selection.**
Resolved: per `(technique, instance)` pair per iteration (`workflow.iteration.per_iteration.select_technique_instance.instance_granularity = "one_technique_one_site_per_iteration"`). Keeps each iteration's diff small, reviewable, and individually measurable.

**Q2 — Technique conflict resolution.**
Resolved: `workflow.iteration.per_iteration.conflict_check` refuses a candidate diff whose lines overlap any previously-applied technique in this run. Conflict is recorded and next candidate selected.

**Q3 — Hot path detection accuracy.**
Resolved: `workflow.pre_flight.identify_hot_paths` requires a representative workload via flamegraph (primary) or samply (fallback). If neither is available, the workflow downgrades all hot-path-conditioned techniques to medium confidence and warns the operator. This is conservative: rather than optimize the wrong code, we widen the candidate set and rely on per-iteration benchmarks to filter.

**Q4 — Multi-file refactors.**
Resolved by exclusion: techniques requiring multi-file refactors are excluded from this YAML (they fail the "action is concrete" criterion). The Cargo.toml-edit techniques (`bc_*`) are single-file by definition. The few cross-file techniques retained (e.g. `iter_return_impl_iterator_over_vec` which touches callers) are gated on call-site analysis and bounded by `safety.max_files_modified_per_iteration: 5`.

**Q5 — Compile time vs. runtime tradeoffs.**
Resolved: tradeoffs are explicit on each technique (`tradeoffs: [compile_time, binary_size]`). The composite scoring weighs all six dimensions; if the operator wants to prioritize compile time, they raise its weight in `dimensions.compile_time.weight`.

**Q6 — When to stop applying a technique.**
Resolved: techniques are not applied wholesale. Each iteration applies one technique at one site, measures, and decides commit/discard. A given technique can apply to many sites across many iterations, interleaved with other techniques per the priority function. This avoids monolithic rewrites and keeps the PR coherent.

---

## What success looks like for the first validation run

Per the bible §7 success criteria:

- ✅ YAML executes without requiring agent-infrastructure code changes
- ✅ Contracts catch behavior-breaking rewrites
- ✅ Final PR shows measurable improvement on at least one dimension
- ✅ Improvements are reviewable as a coherent diff
- ✅ Agent can defend each rewrite by reference to a book technique

The fifth criterion is the strongest test of authority transfer: a reviewer should be able to read the PR, click through to the book section that justifies each change, and confirm the rewrite matches the book's intent.

---

## Provenance & no-novel-claims discipline

The bible's §6 is explicit: "Every technique in the YAML traces to the Rust Performance Book. Novel optimizations beyond the book's scope stay out — they undermine the credibility transfer." This is enforced by the `provenance` block in `rust.yaml` and by the requirement that every technique carries a `source_chapter` and `source_url` that resolves to a section of the book.

The bible's §3 lists example techniques that don't appear in this YAML:
- "Per-call regex compilation → `once_cell` or `lazy_static`"
- "Sequential awaits on independent futures → `try_join!`"

These were illustrative examples in the bible, not items from the book. They are real and useful, but **the book does not cover them**, so they do not belong in this YAML. The discipline holds.

If you want regex and async-Rust optimization techniques, those belong in a separate YAML grounded in a different authoritative source (e.g., the async book, or regex-crate documentation).
