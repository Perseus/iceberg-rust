# `dfd/runtime-predicate-5b82ab9`: fork change ledger

> Fork-only bookkeeping. This file is never to be upstreamed. Drop it with the
> branch.

## Purpose

[datafusion-distributed](https://github.com/datafusion-contrib/datafusion-distributed)
(DFD) prunes Iceberg data files and row groups with DataFusion dynamic filters.
That needs iceberg-rust APIs that are still open upstream PRs. This branch
carries exactly those PRs, each as its own commit, on top of the iceberg-rust
rev DFD already depends on. DFD points at it with a root `Cargo.toml`
`[patch."https://github.com/apache/iceberg-rust"]` for `iceberg`,
`iceberg-property-macro` and `iceberg-storage-opendal`.

## Base rev

`5b82ab99` ([apache/iceberg-rust@5b82ab99](https://github.com/apache/iceberg-rust/commit/5b82ab99aeaf50bf4f5b3b776ac9cd6380a5f3e3),
"perf(manifest): upgrade to apache-avro 0.22 ... (#3354)").

This is exactly the rev that `datafusion-iceberg` pins. Both must resolve to the
same rev, so that the dependency graph has a single `iceberg` crate and types
such as `FileScanTask` are shared between DFD and `datafusion-iceberg`.

## Rules

- History is never rewritten. Downstream `Cargo.lock`s pin specific commits of
  this branch, so every earlier pin must stay resolvable. Changes are only added
  as new commits on top, pushed as fast-forwards.
- One commit per upstream PR, built from the PR's net diff against its base,
  with the PR head SHA recorded. Adaptations needed only because this base is
  older are kept in that commit and described below, or in a separate, clearly
  labelled commit.
- Each commit carries trailers: `Upstream-PR`, `Upstream-commit` (merged) or
  `Upstream-head` (unmerged), `Backport-notes`, `Needed-by`, `Remove-when`.
- Backported code keeps the upstream author as the git author. The fork
  maintainer is the committer. Adaptation-only commits are authored by the fork
  maintainer.

## Rebuilding when the pin moves

When `datafusion-iceberg` moves its iceberg-rust pin to rev `X`:

1. Create a new branch `dfd/runtime-predicate-<short X>` from `X`. Never move
   or rewrite this branch.
2. For each entry below, in order: drop it if `X` already contains the upstream
   merge commit (its removal condition holds). Otherwise re-apply the PR's
   latest net diff on `X`, redo the adaptations that are still needed, and drop
   the ones that are not.
3. Copy this file, update the base rev, the SHAs and the backport notes, and
   mark entries that were dropped.
4. Point DFD's `[patch]` at the new branch head. If every entry is gone, remove
   the `[patch]` entirely.

## Changes

### 1. Execution-time predicates in `ArrowReader` (apache/iceberg-rust#3351)

- **Fork commit:** `3f9c68b7` (`git cherry-pick -x`; author unikdahal, kept).
- **Upstream:** PR [apache/iceberg-rust#3351](https://github.com/apache/iceberg-rust/pull/3351),
  issue [apache/iceberg-rust#3350](https://github.com/apache/iceberg-rust/issues/3350).
- **Upstream SHA used:** PR head `821fc65e3417fcba1fbe772cf476bf3a10516fea`
  (single-commit PR; PR base was `ecba0b53`, an ancestor of `5b82ab99`).
- **Upstream status:** open, not merged (as of 2026-10-24).
- **Backport notes:** applied cleanly. One test needed an adaptation to this
  base; see entry 2.
- **Why DFD needs it:** adds `RuntimePredicateProvider` and
  `ArrowReaderBuilder::with_runtime_predicate_provider`, which DFD uses to apply
  live dynamic filters when a data-file task opens.
- **Used by:** `datafusion-distributed-iceberg` file-open pruning, behind
  `iceberg.dynamic_filter_pruning_enabled`;
  [Perseus/datafusion-distributed#5](https://github.com/Perseus/datafusion-distributed/pull/5)
  (adds the `[patch]`) and
  [Perseus/datafusion-distributed#6](https://github.com/Perseus/datafusion-distributed/pull/6)
  (uses the API).
- **Remove when:** the pinned iceberg-rust rev contains #3351's merge commit.

### 2. Test-only `.unwrap()` for `FileScanTaskDeleteFile::builder().build()`

- **Fork commit:** `1935cc80` (authored by the fork maintainer; adaptation only,
  no upstream code).
- **Upstream cause:** [apache/iceberg-rust#3145](https://github.com/apache/iceberg-rust/pull/3145),
  merged as [`b0d8baaf`](https://github.com/apache/iceberg-rust/commit/b0d8baaf966813f82363279db46d6688bf455d24)
  ("fix(scan): validate deletion vector delete tasks"). It made
  `FileScanTaskDeleteFile`'s builder validate, so `build()` returns
  `Result<FileScanTaskDeleteFile>`. #3145 is in `5b82ab99` but not in #3351's
  base (`ecba0b53`), so #3351's test
  (`crates/iceberg/src/arrow/reader/runtime_predicate_tests.rs`) did not compile
  here.
- **Upstream status:** #3145 merged; #3351 open (as of 2026-10-24).
- **Backport notes:** adds `.unwrap()` after one `build()` call in that test.
  No library code changes.
- **Why DFD needs it:** keeps the #3351 tests compiling and running on this
  base.
- **Used by:** only the fork's tests.
- **Remove when:** #3351 is rebased onto a base containing #3145 (then the
  upstream test already handles the `Result`), or with entry 1.

### 3. Optional column statistics on `FileScanTask` (apache/iceberg-rust#3424)

- **Fork commit:** `9832f9c7` (manual backport; upstream author Anurag
  Mantripragada kept as git author, fork maintainer is committer).
- **Upstream:** PR [apache/iceberg-rust#3424](https://github.com/apache/iceberg-rust/pull/3424),
  issue [apache/iceberg-rust#3422](https://github.com/apache/iceberg-rust/issues/3422).
- **Upstream SHA used:** PR head `86e35d3d6365fa3dc1a17be26096f45b2b3e6985`,
  net diff against merge base `beba39c8` (all 10 PR commits squashed).
- **Upstream status:** open, not merged (as of 2026-10-24).
- **Backport notes:** `context.rs`, `task.rs`, `test_utils/scan.rs` and
  `public-api.txt` applied cleanly with `git apply --3way`; `scan/mod.rs` had 7
  conflict hunks, all from upstream changes made after `5b82ab99`. Resolutions:
  1. `collect_scan_field_ids`: use the PR's new `require_field_id` helper, as
     upstream does. At this base `resolve_field_id` returns `Option<i32>`, not
     `Result<Option<i32>>` (that came with
     [#3070](https://github.com/apache/iceberg-rust/pull/3070), "reject ambiguous
     case-insensitive fields"), so `require_field_id` drops the `?`. Effect:
     `with_column_stats` resolves names exactly like `select` does on this base;
     an ambiguous case-insensitive name picks a field instead of failing, the same
     as `select` here.
  2. and 3. `TableScanBuilder` struct and `new`: add `column_stats_names` next to
     this base's `snapshot_id` (upstream has `snapshot_selection` from
     [#3260](https://github.com/apache/iceberg-rust/pull/3260), time travel).
  4. Builder methods: add the PR's `with_column_stats` unchanged; keep this
     base's `snapshot_id` method and docs (upstream's docs mention `as_of_time`,
     which does not exist here).
  5. Test imports: import only `resolve_column_stats_columns` (this base's tests
     do not import `resolve_field_id`).
  6. Test `spec` imports: this base's list plus the PR's additions, `ListType`
     and `StructType`.
  7. Tests: add the PR's `nested_schema`, `column_names` and the two
     `resolve_column_stats_columns` tests. Not carried: the two
     `case_insensitive ... ambiguous` tests, which are #3070's, appear only as
     conflict context.

  Non-conflict adaptations, needed for the code to compile on this base:
  - `task.rs` `bound_as_column_type` calls `Datum::promote_to`, added by
    [#3483](https://github.com/apache/iceberg-rust/pull/3483). It is inlined as the
    same widening (`int` -> `long`, `float` -> `double`); no `spec` changes.
  - Tests build fields with `Type::Primitive(..)`: before
    [#3527](https://github.com/apache/iceberg-rust/pull/3527) the `NestedField`
    constructors take `Type`, not `impl Into<Type>`.
  - `test_utils/scan.rs` imports `SchemaRef`, which upstream already imported.
- **Why DFD needs it:** DFD will skip whole data files by comparing each file's
  column min/max with the live dynamic filters, which needs per-file column
  statistics on `FileScanTask`.
- **Used by:** the root `Cargo.toml` `[patch]` on
  [`iceberg-dyn-filter/f2-fork-column-stats`](https://github.com/Perseus/datafusion-distributed/tree/iceberg-dyn-filter/f2-fork-column-stats).
  No DFD code calls `with_column_stats` yet; later changes on top of that
  branch will.
- **Remove when:** the pinned iceberg-rust rev contains #3424's merge commit.
