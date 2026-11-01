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
  as new commits on top, pushed as fast-forwards. There is one exception,
  described under "History" below.
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

- **Fork commit:** `6ed9531b` (`git cherry-pick -x`; author unikdahal, kept).
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
  [Perseus/datafusion-distributed#18](https://github.com/Perseus/datafusion-distributed/pull/18) (branch `iceberg-dyn-filter-v2/f2-runtime-predicate-pruning`), which
  adds the `[patch]` and uses the API. It replaces Perseus/datafusion-distributed#5
  and #6 from an earlier series.
- **Remove when:** the pinned iceberg-rust rev contains #3351's merge commit.

### 2. Test-only `.unwrap()` for `FileScanTaskDeleteFile::builder().build()`

- **Fork commit:** `cbfab28d` (authored by the fork maintainer; adaptation only,
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

- **Fork commit:** `cfd8069b` (manual backport; upstream author Anurag
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
  - Docs: upstream's `with_column_stats` doc says `build` fails on an ambiguous
    case-insensitive name. That check is #3070's and is not carried, so the doc
    instead says names resolve as `select` does and an ambiguous name picks a
    field (fixed in a follow-up commit, see "Commit trailer notes").
- **Why DFD needs it:** DFD will skip whole data files by comparing each file's
  column min/max with the live dynamic filters, which needs per-file column
  statistics on `FileScanTask`.
- **Used by:** the root `Cargo.toml` `[patch]` in
  [Perseus/datafusion-distributed#7](https://github.com/Perseus/datafusion-distributed/pull/7).
  That PR belongs to an earlier series, since replaced. In the current series
  the `[patch]` is added once, by [Perseus/datafusion-distributed#18](https://github.com/Perseus/datafusion-distributed/pull/18),
  and `with_column_stats` is called by the follow-up that skips data files by
  column bounds. This pin carries the backport already so that follow-up needs
  no repin.
- **Remove when:** the pinned iceberg-rust rev contains #3424's merge commit.

### 4. Pruning-only runtime predicates (fork-original)

- **Fork commits:** `fca3878a` (code and tests) and the commit that adds this
  entry. Authored by the fork maintainer.
- **Fork-original:** this is not a backport. No upstream PR or commit exists.
  It builds on entry 1 (apache/iceberg-rust#3351).
- **What it adds:** `ArrowReaderBuilder::with_runtime_predicate_row_filter(bool)`.
  The default, `true`, keeps #3351's behaviour unchanged. With `false`, the
  runtime predicate still prunes row groups (statistics and bloom filters) and
  pages, but is not ANDed into the Arrow row filter. The task's own predicate,
  equality deletes and positional deletes are applied exactly as before. With
  no task predicate and no deletes, no row filter is installed, so the file is
  decoded directly with no late materialization.
- **Why it exists:** on TPC-H SF10 Iceberg, DFD's runtime predicates (compacted
  per-column ranges from DataFusion dynamic filters) made some queries slower
  than with them off: q8 454 ms vs 392 ms and q9 702 ms vs 589 ms (median,
  single node). Profiles showed the time going to the row-filter evaluation,
  which costs rows times predicate size. A local experiment that kept pruning
  but dropped the runtime part of the row filter removed every regression
  measured. Per-row evaluation is not needed for a correct result: DataFusion
  puts no `FilterExec` above the scan for a dynamic filter; the operator that
  produced it makes the result exact (the hash join probes its hash table, TopK
  and aggregates re-check their threshold). DataFusion's own Parquet scan
  applies dynamic filters per row only when
  `datafusion.execution.parquet.pushdown_filters` is on.
- **Related upstream:** [apache/iceberg-rust#3351](https://github.com/apache/iceberg-rust/pull/3351);
  epic [apache/iceberg-rust#3343](https://github.com/apache/iceberg-rust/issues/3343),
  item "Preserve late materialization with runtime predicates".
- **Long-term:** to be proposed upstream as part of that epic item; DFD then
  moves to whatever control upstream ships. When proposing it, an enum such as
  `RuntimePredicateUse::{PruneOnly, PruneAndFilter}` may be preferable to a
  bool: it names both modes and leaves room for more.
- **Used by:** `datafusion-distributed-iceberg`'s file open, which uses
  pruning-only mode unless `datafusion.execution.parquet.pushdown_filters` is
  true; branch
  [`iceberg-dyn-filter-v2/f2-runtime-predicate-pruning`](https://github.com/Perseus/datafusion-distributed/tree/iceberg-dyn-filter-v2/f2-runtime-predicate-pruning)
  ([Perseus/datafusion-distributed#18](https://github.com/Perseus/datafusion-distributed/pull/18)). The earlier branch
  `iceberg-dyn-filter/g2-pruning-only-runtime-predicate` is superseded.
- **Remove when:** upstream ships an equivalent control for runtime predicates
  (epic #3343, item "Preserve late materialization with runtime predicates"),
  or with entry 1.

## Commit trailer notes

- `acf73714`, `cfd8069b` and `dacdc376` carry
  `Needed-by: https://github.com/Perseus/datafusion-distributed/tree/iceberg-dyn-filter/f2-fork-column-stats`
  (a branch link), because the DFD PR did not exist when they were committed.
  The PR is [Perseus/datafusion-distributed#7](https://github.com/Perseus/datafusion-distributed/pull/7).
  Their messages were not changed (not even by the identity rewrite in
  "History"), so the trailers still read as committed.
- The follow-up commit that corrects the `with_column_stats` doc and this
  ledger links #7 directly.
- Entry 4's first two commits carry
  `Needed-by: https://github.com/Perseus/datafusion-distributed/tree/iceberg-dyn-filter/g2-pruning-only-runtime-predicate`
  (a branch link), for the same reason. That branch is superseded by
  [`iceberg-dyn-filter-v2/f2-runtime-predicate-pruning`](https://github.com/Perseus/datafusion-distributed/tree/iceberg-dyn-filter-v2/f2-runtime-predicate-pruning),
  which is what DFD's `[patch]` now comes from.
- The ledger-correction commit after entry 4 carries `Needed-by:` that v2
  branch.

## History

- On 2026-10-30 this branch was rewritten once, only to replace the author and
  committer identity of the fork maintainer's commits with
  `Perseus <6716458+Perseus@users.noreply.github.com>`. Upstream authors were
  kept. Every commit's tree and message are unchanged. This is the one exception
  to the no-rewrite rule above; the DFD branches were repinned to the new SHAs
  at the same time.
- Commit messages still mention the old SHAs where they did before (for
  example in `Needed-by` trailers); use this table to map them.

| Old SHA | New SHA | Commit |
|---|---|---|
| `3f9c68b786978da8a989f328adc76d7cb870c435` | `6ed9531b7162eaf9797c8a67bb65215b368e5482` | feat(reader): support execution-time predicates in ArrowReader |
| `1935cc80e95ab0210183db20f601ad2129f8d7f5` | `cbfab28d97d8621cd6e427152d75fe93e49f572d` | test(reader): unwrap FileScanTaskDeleteFile builder result at this base rev |
| `b822c0521ddaf0f3777e86b75e76328bffda5816` | `acf737147a4d9d1b2ef4c42585c1765d4c4050ef` | docs: add DFD_FORK.md change ledger for this fork branch |
| `9832f9c7af9738ae550f33211a4145a2438b1bb6` | `cfd8069b163913c51506abbd44e6a7ac0d8f521d` | feat(scan): optionally retain column statistics on FileScanTask |
| `d7f56eb4b8b04d5612ca42d53a31946ffcced7b5` | `dacdc37692ac91edacafdb445bd49020be24fc1b` | docs: record apache/iceberg-rust#3424 backport in DFD_FORK.md |
| `12d45da7b834b0e3e6fa6e71f0f32d5db0f0c615` | `bd8a39b5982de241fe62d392d586f7d54ea3e2d3` | docs(scan): match with_column_stats docs to this base; link DFD PR in ledger |
