# Workspace discovery benchmarks

`workloads.rs` measures the `ignore` operations used by Python development tools:

| Consumer | Operation | Benchmarks |
| --- | --- | --- |
| [Ruff file discovery](https://github.com/astral-sh/ruff/blob/ae8ec3753192debd310e7b3d93255a19a5a2af85/crates/ruff_workspace/src/resolver.rs#L530) and [ty project indexing](https://github.com/astral-sh/ruff/blob/ae8ec3753192debd310e7b3d93255a19a5a2af85/crates/ruff_db/src/system/os.rs#L312) | Parallel walks, hidden files included, up to 12 workers | `walk_*_parallel_*`, `walk_explicit_files` |
| [uv script discovery](https://github.com/astral-sh/uv/blob/83d556d712c7e316932b239c49e86a5de95d124a/crates/uv/src/commands/workspace/list.rs#L137) | Serial traversal with ignore rules and hidden files included | `walk_*_serial`, `uv_scripts_source_archive` |
| [uv workspace discovery](https://github.com/astral-sh/uv/blob/83d556d712c7e316932b239c49e86a5de95d124a/crates/uv-workspace/src/workspace.rs#L1931) | Stop at the first non-directory entry, or exhaust an ignored member | `uv_*_workspace_member` |
| [ty incremental indexing](https://github.com/astral-sh/ruff/blob/ae8ec3753192debd310e7b3d93255a19a5a2af85/crates/ruff_db/src/system/os/ignore.rs#L10) | Repeated relative-path checks with cached ancestor matchers | `incremental_*` |

The synthetic trees contain Python modules, package metadata, non-Python files,
ignored bytecode directories, a virtual environment, and build outputs. The
layouts vary independently: a small project, many sibling packages, deeply
nested packages, and packages with their own ignore rules and negations. The
parallel visitor counts files locally and merges once per worker, avoiding an
artificial shared counter on every entry. One- and four-worker cases supplement
the 12-worker configuration used by Ruff and ty. The unfiltered walk models
disabling ignore rules and provides a control for traversal overhead.
The source-archive case honors `.gitignore` without a `.git` directory, matching
uv's `require_git(false)` configuration.

Fixtures are created before timing and removed afterward. Every walk checks its
Python file count against the fixture's expected count. The benchmarks retain
parent ignore discovery, but disable global Git ignore configuration so that
results do not depend on a user's Git configuration. Runs use a warm filesystem
cache. They measure `ignore`, excluding parsing, configuration resolution, and
other application work; they are not end-to-end application speedups.

`incremental_cold` includes building matchers and loading directory rules on
first use. `incremental_warm` checks 256 paths after their directories have been
cached. `incremental_ignored` checks a path beneath a pruned virtual environment.
The `build_*` and `match_*` microbenchmarks help distinguish compilation and
matching costs from filesystem traversal.

Like the `globset` benchmarks, these use Rust's nightly `test` harness:

```console
cargo +nightly bench -p ignore --bench workloads
cargo +nightly bench -p ignore --bench workloads incremental
```

For comparisons, use the same compiler, profile, fixture parameters, and CPU
affinity for both revisions. Alternate baseline and candidate runs, and retain
the per-run results instead of relying on a single measurement. Parallel walks
also include worker startup and shutdown, which can dominate small projects.
