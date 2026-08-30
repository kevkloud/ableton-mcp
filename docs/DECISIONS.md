# Design decisions on `kloud` — alternatives considered

Kept alongside the commits so that if a change misbehaves later, the fallback
option is already written down rather than re-derived.

## delete_track (4eea833)

**Single-track primitive, not a list.** `delete_track(track_index, recursive=False)`
mirrors `delete_clip`'s shape; batch deletion is the caller working in descending
index order (and now `batch` does this automatically).
*Alternative:* accept a list of indices and sort internally. Fewer round trips, and
it removes the index-shift footgun at the source — but it makes the primitive
inconsistent with every other command. Superseded in practice by `batch`.

**Refuse a non-empty group unless `recursive=True`,** returning a description of
every track that would be removed.
*Alternative:* delete silently and rely on Live's ⌘Z. Simpler and less chatty, but
a group deletion can remove a dozen tracks and the caller may not realise what the
index pointed at. If the confirmation step becomes annoying in practice, the middle
ground is to refuse only when a child contains clips.

**Return a manifest of what was destroyed.** There is no undo tool on the MCP side,
so the transcript is the record.
*Alternative:* return just a count. Smaller payload, but nothing to reconstruct from.

## include_clip_slots (3c1c3bc)

**Opt-in parameter, default keeps the existing fat shape.**
*Alternative considered and rejected for now:* make lean the default and return
`occupied_slots` always. Better for every caller, but a breaking response-shape
change. A third option — lean by default on `kloud`, opt-in for upstream — was
rejected because the branches would diverge and need managing forever.
If upstream ever accepts a shape change, flipping the default is a one-line edit.

## TCP_NODELAY / status message (23f1424)

**Kept despite measuring no improvement.** Correct socket hygiene, zero risk.
Baseline before: min 99ms, median 200ms. After: min 97ms, median 201ms —
distribution unchanged. The Nagle hypothesis was wrong; Live's ~100ms scheduler
tick dominates entirely. Do not describe this as a performance win.
*Implication for future work:* socket- and payload-level tuning cannot help
latency. Only removing round trips can. That is why `batch` exists.

## Dropped: per-command logging behind an env var

Was going to gate `log_message` on the hot path behind `ABLETON_MCP_VERBOSE`.
Dropped once the TCP_NODELAY measurement showed the tick dominates — the change
was justified entirely by a latency theory that did not survive contact with the
data. Revisit only if profiling inside Live shows logging on the critical path.
