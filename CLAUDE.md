# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

Workflow rules (worktrees, branch naming, commits, merging, quality gates, test requirements) are in AGENTS.md and apply in full:

@AGENTS.md

## Commands

```sh
cargo test                                   # all tests (unit tests live in each src/*.rs)
cargo test aggregates_full_tree              # single test by name substring
cargo test scan::tests                       # one module's tests
cargo test -- --nocapture                    # show println! output

cargo fmt --check && cargo clippy --all-targets -- -D warnings && cargo test && cargo build --release
                                             # exactly what CI runs (ubuntu + macos)

cargo run -- <PATH>                          # interactive TUI
cargo run -- <PATH> -o -                     # headless scan → ncdu JSON on stdout (no TTY needed)
cargo run -- -f dump.json -o -               # import + re-export a dump
./build-portable.sh                          # static musl Linux + native macOS binaries into dist/
```

The TUI needs a real terminal, so check behavior headlessly with `-o -` or through unit tests on `App`/`Tree`.

## Platform

Unix only (`std::os::unix::fs::MetadataExt` for dev/ino/blocks/nlink). Platform-specific code uses `#[cfg(target_os = ...)]`, e.g. the three `opener_command` variants in `app.rs`. Anything you change has to build and pass on both Linux and macOS.

Dependencies are deliberately few (`ratatui`, `crossbeam-channel`, `serde_json`, `compact_str`). The core never spawns external processes. The only exceptions are `o` (the platform opener), `b` (`$SHELL`) and `rcdu update` (`curl`). SHA-256 is implemented in-tree (`sha256.rs`) to avoid adding a crate.

## Architecture

### Data flow: scanner threads → channel → UI-thread-owned tree

- **`scan.rs`**: N worker threads (default `cores × 4`) pull `DirJob`s from a shared `Frontier`. Each worker reads one directory, `lstat`s its entries, pushes subdirectories back onto the frontier, and sends exactly **one `Batch` per directory** over an unbounded crossbeam channel. A `Batch` carries the `parent_id` (a scanner-assigned `u64`, not a `NodeIdx`) and the new nodes. Subdirectories carry a `dir_id` that later batches use as their `parent_id`.
- **Scan completion**: a `pending` count (queued + in-flight jobs) reaches 0 → workers exit → every `Sender` drops → the channel disconnects. A disconnect is the UI's only "scan done" signal. `ScanControl` (the UI's steering handle) intentionally holds no `Sender`, so it can't keep the channel alive.
- **Priority**: the frontier has `hot` and `cold` queues. `ScanControl::set_focus(path)` re-buckets the queued jobs so anything under the viewed directory goes first. `App::refocus()` calls this on every navigation.
- **`model.rs` (`Tree`)**: a flat `Vec<Node>` indexed by `NodeIdx`, owned only by the UI thread. `Tree::apply` maps `parent_id → NodeIdx` through `dir_index`, grafts the nodes, and propagates sizes up every ancestor, so totals are correct at every moment. Batches whose parent doesn't exist yet are buffered in `orphans`. `finish_scan()` frees `dir_index` and `orphans` once all streams end.
- **Counted vs. displayed size**: hard-link duplicates (`shared`) and excluded entries (`Excluded::{Pattern, OtherFs, Age}`) keep their real size for display but contribute 0 to aggregates (`Node::counts()`). A node's own size is derived (aggregate minus counted children), not stored. Keep `Node` lean: there is one per filesystem entry.
- **Tombstones**: deleting or rescanning detaches nodes but never removes them from `tree.nodes`. Indices stay stable, and unreachable nodes are simply left in the vector.

### Safety gates (do not weaken)

- `Node::unscanned` counts the directories not yet read in a subtree. `Tree::subtree_complete(idx)` (`unscanned == 0`) gates both **delete** (`d`) and **rescan** (`r`). Deleting a partially scanned subtree could remove things the user hasn't seen. Rescanning one could double-count batches still arriving from the original scan.
- Deletion runs on a background thread (`perform_delete` → `poll_delete`). Only one runs at a time, the subtree being deleted is locked, and quitting during a delete needs two presses (`quit_armed`).

### In-place rescan (`r`)

`Tree::begin_refresh` detaches the subtree and resets its aggregates. `scan::start_at(path, opts, id_base)` then starts a new scanner **generation**. Each generation gets a disjoint id range (`id_base = n × SCAN_ID_STRIDE`, where `SCAN_ID_STRIDE` is 2⁴⁰) so it can stream into the same tree alongside the original scan. `App` keeps a `Vec<ScanHandle>` plus a `Vec<ScanControl>`, and `poll_scans` drains all of them. Each generation has its own hard-link `seen` set, so hard links crossing the rescan boundary can be overcounted (a known limitation, documented in the README).

### UI / event loop

- `main.rs::run` loops: `poll_scans(20_000)` (a cap, so input never starves) → `poll_delete` / `poll_open` → handle any `take_shell_request()` (tear down the TUI, run `$SHELL`, re-init) → draw → `event::poll` with an 80 ms timeout while busy, 250 ms otherwise.
- Long-running side effects (delete, open) go to background threads and are polled once per frame through channels. Keep the UI thread non-blocking.
- `App.selected` tracks node **identity** (`NodeIdx`), not row position, so the highlight stays on the same entry as live updates re-sort the list.
- `App.status` is a one-line footer message that clears on the next key press. `App.modal` holds popups (help, confirm, top files, info).

### Adding a key binding

Touch all of these in one change:
1. `KeyAction` variant (`app.rs`)
2. `decode_key` mapping (`main.rs`; watch the match-arm order, e.g. `Ctrl-d` must come before plain `d`)
3. `App::on_key` handling (`app.rs`)
4. The help popup `render_help` (`ui.rs`)
5. The Keys table and a feature section in README.md

### ncdu compatibility

`dump.rs` reads and writes ncdu JSON format version 1.2 (`[1, 2, {meta}, <root>]`). On import, hard-link dedup is re-derived from `(dev, ino)` so totals match a live scan. CLI flags mirror ncdu's (`-x`, `-r`, `-o`, `-f`, `-X`, `--exclude`), so keep new flags consistent with ncdu where an equivalent exists. Arguments are parsed by hand in `main.rs::parse_args`; there is no clap.

## Tests

Tests live in `#[cfg(test)]` modules at the bottom of each file. Patterns to reuse:
- **`scan.rs`**: builds a real temporary directory tree, runs the real scanner to completion, and asserts on aggregates. Covers concurrency, hard links, excludes, focus priority and rescan.
- **`app.rs`**: builds a `Tree` in memory with `Tree::new_imported` + `import_child` (see `app_with_entries()`), wraps it in `App::new(tree, disk_usage, read_only, scanning)`, and drives it through `on_key(KeyAction::…)`. No terminal is needed.
- Rendering in `ui.rs` isn't unit-tested. Per AGENTS.md, say so in the PR and verify it manually.
