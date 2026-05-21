# Transmute

Declarative optimization specs grounded in authoritative performance literature, executed by AI agents.

## What this is

Performance knowledge for any given language lives in books, blog posts, RFCs, and the heads of compiler engineers. None of it is directly executable. If you point an LLM at a codebase and say "make it faster," you get plausible-sounding optimizations with no grounding in real expertise and no verification that the behavior was preserved.

Transmute closes that gap by separating two concerns:

1. **What to optimize** — a YAML spec, grounded section-by-section in an authoritative source for one language (e.g. Nethercote's *Rust Performance Book* for `rust.yaml`).
2. **How to detect, apply, and verify it** — an AI agent (Claude) acting as the runtime, working iteratively against the spec.

The spec is portable. The agent is generic. The grounding is what makes the optimizations trustworthy.

## The goal

A performance change that gets applied to a codebase should satisfy three properties:

- **Grounded** — it comes from someone who actually knows what they're talking about, not from an LLM pattern-matching on similar-looking code.
- **Verified** — its effect is measured, not assumed. Across multiple dimensions (latency, allocations, peak memory, binary size, LOC, compile time), with explicit weights and per-dimension regression tolerance.
- **Safe** — it preserves behavior. Tests pass. Benchmarks don't regress past tolerance. Invariants hold. Things the spec lists as forbidden (e.g. introducing `unsafe`) don't appear in diffs.

Transmute encodes all three into the schema itself, so the agent can't accidentally violate them.

## How it works

The spec defines:

- **target** — toolchain and build commands
- **contracts** — tests, benchmarks, invariants, behavior assertions, each with an `on_failure` policy
- **dimensions** — what to optimize for, with weights summing to 1.0, plus composite scoring rules
- **techniques** — the catalog of detectable + applicable + verifiable patterns, each with confidence, tradeoffs, and (where relevant) a `requires_human_consent` gate
- **workflow** — pre-flight checks, iteration loop, convergence criteria, termination
- **safety** — budgets (wall-clock, tokens, files-per-iteration, lines-per-iteration), forbidden changes
- **provenance** — source URL, scope, what's deliberately excluded and why

The agent runs the workflow:

```
pre-flight → detect technique instances → propose diff
          → run contracts → measure dimensions
          → keep or revert → repeat
          → terminate on convergence or budget exhaustion
          → emit PR
```

One technique-instance per iteration. Changes that don't improve the composite score past threshold get reverted. Changes that trip a contract get reverted. The spec, not the agent's intuition, decides what's a win.

## Status

POC. The `rust.yaml` spec (55 techniques across 13 chapters of Nethercote) is complete and passes integrity checks; validation against a real Rust substrate is the next milestone. This is a personal-use tool, not a commercial product. Use accordingly.

## Languages

| Language | Spec        | Source                                                                                        | Techniques |
| -------- | ----------- | --------------------------------------------------------------------------------------------- | ---------- |
| Rust     | `rust.yaml` | [The Rust Performance Book](https://nnethercote.github.io/perf-book/) (Nicholas Nethercote)   | 55         |
| Python   | *planned*   | TBD (candidate: *High Performance Python*)                                                    | —          |
| Go       | *planned*   | TBD (candidate: Go performance docs + Dave Cheney)                                            | —          |
| C++      | *planned*   | TBD                                                                                           | —          |

## Repo layout

```
README.md                        # you are here
SCHEMA_REFERENCE.md              # schema documentation, reusable across languages
rust.yaml                        # executable optimization spec for Rust
OPERATIONALIZATION_NOTES.md      # what made it in, what was deferred, why (Rust)
```

When other languages get added, expect parallel files: `python.yaml` + `OPERATIONALIZATION_NOTES_python.md`, etc.

## How to use

1. Drop the relevant `<language>.yaml` and your target codebase into an environment where Claude has filesystem and shell access.
2. Tell Claude: *"Execute the workflow in `rust.yaml` against `<path-to-codebase>`."*
3. Claude reads the spec, runs pre-flight, iterates through techniques, runs contracts after each change, scores against dimensions, and emits a PR-shaped summary at the end.
4. Review the diff. Merge what holds up.

The spec is self-describing — the `workflow` section tells the agent exactly what to do, the `safety` section bounds what it can do, the `contracts` section catches what it shouldn't have done.

## Authoring a new language

See `SCHEMA_REFERENCE.md` for the full schema and the authoring recipe. Short version:

1. Pick an authoritative source for the language (book, official docs, well-established practitioner).
2. Walk it section by section, applying four criteria per candidate technique:
   - **Mechanical detection** — can a script find instances?
   - **Concrete action** — is the rewrite unambiguous?
   - **Measurable verification** — does a benchmark catch the effect?
   - **Automated validation** — do tests catch behavior changes?
3. Techniques that pass all four → `high` confidence. Partial passes → `medium`. Architectural or judgment-heavy → `low`, gated by `requires_human_consent`.
4. Swap detection and profiling tools for the language's ecosystem. The dimension set and workflow structure stay the same.

The point of the schema being reusable is that adding a new language is a grounding exercise, not a redesign.

## Why this exists

Open question: is the right way to use LLMs for code optimization to ask them to reason from scratch, or to ground them in domain authorities and let them act as a careful runtime over an explicit spec?

This repo is the second answer, made concrete enough to test. If it works, it works. If it doesn't, the failure mode will be informative.
