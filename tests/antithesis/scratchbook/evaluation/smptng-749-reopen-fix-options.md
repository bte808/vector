# SMPTNG-749 reopen crash-loop: fixes that work

Two approaches satisfy all three constraints (no data loss, no liveness loss, no
on-disk format change). Both avoid a v3. Ship the first; the second is a heavier
alternative kept only for completeness.

## The bug

`delete_completed_data_file` (reader.rs:504-584) runs three non-atomic steps in the
wrong order: `delete_file(data-N.dat)` (572) -> `increment_acked_reader_file_id()`
(574) -> `flush()` (575, the only durable point). A crash between 572 and 575 leaves
the `.dat` gone but the durable ledger still naming file N as the reader resume point.
On restart `seek_to_next_record` re-mmaps that missing path (reader.rs:867, 869-874),
ENOENT -> `ReaderSeekFailed` -> exit 78 (EX_CONFIG) -> supervised restart -> identical
failure -> permanent crash loop. Two principles broken: write-ahead ordering (destroy
the referent before the dereference is durable) and crash-only/recovery==startup
(reopen IS recovery; mapping a recoverable inconsistency to EX_CONFIG guarantees the
loop). A buffer that exists to drain must fail open, not closed.

The skip is safe because a data file is unlinked ONLY after every record in it is
acked (`handle_pending_acknowledgements`, reader.rs:587-716). A missing resume file ==
fully delivered. Skipping it drops nothing.

## How real stores solve it

The universal rule: make the reference-removal durable BEFORE unlinking the referent,
and on load treat a resume pointer that points past/below available data as a
forward-skip to the nearest valid position, never as corruption.

- Kafka: rename-to-`.deleted` then async unlink; recovery-point checkpoint advances
  only after covered data is fsynced; a committed offset below `logStartOffset` is
  `OffsetOutOfRange` -> deterministic forward reset, logged, never fatal;
  `LogLoader.recoverLog` reconciles `logStartOffset` forward to the lowest base offset
  present.
- RocksDB: a compaction fsyncs the MANIFEST edit dropping old SSTs before unlinking
  them, plus an idempotent orphan-GC sweep on open.
- PostgreSQL: WAL segments removed only after a checkpoint durably records a redo point
  past them. etcd: snapshot persisted before truncating the log it covers; reads below
  the compacted index return `ErrCompacted` served from the snapshot.

disk_v2 already holds Kafka's key precondition for free (unlink only after fully
acked). It just fails to apply the forward-skip on load.

## Option A (ship this): reopen reconciliation pass

Add `reconcile_reader_position` in reader.rs, called from `from_config_inner`
(mod.rs) after `validate_last_write` and before `seek_to_next_record`. It walks
`reader_current_data_file` forward to the lowest file id that actually exists, then
flushes once. After it runs, the ledger points at a real file, so `seek_to_next_record`
is left untouched and reopen completes.

```
loop {
    let reader_id = ledger.get_current_reader_file_id();
    let writer_id = ledger.get_current_writer_file_id();
    if reader_id == writer_id || reader_id == ledger.get_next_writer_file_id() { break; }
    let path = ledger.get_data_file_path(reader_id);
    match ledger.filesystem().open_file_readable(&path).await {
        Ok(_) => break,                                   // resume file present
        Err(e) if e.kind() == NotFound => {
            warn!(skipped_file_id = reader_id, writer_file_id = writer_id,
                  data_file_path = %path.display(),
                  "Reader resume data file missing on reopen; fully acknowledged \
                   before deletion. Advancing past it.");
            // increment a recovery-skip counter
            ledger.increment_acked_reader_file_id();      // advance only
            continue;
        }
        Err(e) => return Err(e.into()),                   // genuine I/O error: fail closed
    }
}
if advanced { ledger.flush()?; }                          // single durable commit
```

Mandatory companion edit: the dominant skip path at runtime is
`ensure_ready_for_read` line 803, which today logs nothing. Add the same `warn!` +
counter on that branch so every forward-skip is observable, not just the rare seek
fast-path one.

Do NOT touch `total_buffer_size` on a skip. `update_buffer_size` (ledger.rs:680-737)
seeds it from a sum over files that EXIST; an absent file contributed zero, so a
decrement underflows and wedges the writer (the #21683 class). This is also why
`delete_completed_data_file` cannot be reused here (its `open_file_readable` at
reader.rs:532 would ENOENT).

Constraints:
- No data loss: only advances past a file that is absent AND strictly below the
  writer (the equality guards break before the live file); the invariant proves any
  such file was fully delivered; non-NotFound errors propagate; every skip logged +
  counted.
- No liveness loss: ledger names an existing file before seek runs; loop is monotone,
  bounded by `MAX_FILE_ID`, terminates at the writer; forward-only + single flush =
  crash-only idempotent.
- No format change: probes via existing per-id `open_file_readable`; writes only the
  existing `reader_current_data_file` via existing `increment_acked_reader_file_id` +
  `flush`. Cross-version compatible both directions.

Optional hardening (gated): also reorder `delete_completed_data_file` to advance +
flush BEFORE unlink, shrinking the crash window to a harmless orphan for newly-written
buffers. But a bare reorder regresses liveness: an orphan below the reader collides
with the writer on ring-wrap (`AlreadyExists` -> `wait_for_reader` forever). If you
take the reorder you MUST also add orphan reclamation (a bounded sweep in
`update_buffer_size` for ids outside the live ring span, or `AlreadyExists`-reuse in
`ensure_ready_for_write`). Since Option A already restores liveness fully, this is
optional, not required.

## Option B (alternative, heavier): two-phase rename-then-delete + startup GC

Kafka-style. `delete_completed_data_file` renames `data-N.dat` -> `data-N.dat.deleted`,
advances + flushes the ledger, then unlinks the `.deleted` file; reopen runs a GC sweep
that purges any leftover `.deleted`. A crash never leaves the reader naming a vanished
file. Satisfies all three constraints (`.deleted` is an ephemeral sidecar, not a format
change; an older binary ignores it). But it adds a `rename_file` trait method, the GC
sweep, and on-disk surface for zero guarantee beyond Option A. Use only if you want the
delete itself crash-atomic for reasons beyond this bug. Otherwise skip it.

## What the repro test must assert

The repro `reopen_recovers_when_reader_resume_data_file_is_missing` lives on branch
`blt/disk_buffers_v2_reopen_buffer_after_restart_crash_loop`. Against the in-memory
test filesystem (`MAX_FILE_ID` = 6, common.rs:43) the fix must hold:

1. Crash window: reader@N, N unlinked, ledger durably @N -> reopen succeeds, build does
   not return `ReaderSeekFailed`.
2. Lossless drain: every record in files N+1..writer delivered as written; no surviving
   file skipped.
3. Skip observed: WARN fires with `skipped_file_id`/`writer_file_id` and the
   recovery-skip counter increments -- asserted against `ensure_ready_for_read:803`,
   the path that actually fires.
4. Multiple contiguous missing files: reader@N, N..N+k unlinked, writer@N+k+1 ->
   single-steps to N+k+1, terminates, drains.
5. Genuine I/O error (not NotFound) propagates (fail-closed), not skipped.
6. `total_buffer_size` does not underflow after a skip (equals the sum of surviving
   files).
7. Ring wraparound: reader near `MAX_FILE_ID-1`, writer wrapped lower, missing file
   between -> equality guard (not numeric `<`) terminates; no infinite loop, live
   writer file not skipped.

Open items for review: confirm no reader/writer task starts between `update_buffer_size`
and reconcile in `from_config_inner` (the zero-contribution buffer-size argument depends
on it); pick the skip metric level (expected fully-acked skip is INFO/WARN + dedicated
counter, not the dropped-events path); if the reorder is taken, orphan reclamation is
mandatory.
