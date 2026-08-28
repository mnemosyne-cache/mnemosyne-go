# DESIGN.md

## Goal

Build the Go implementation of Mnemosyne — the same LRU and TTL
caching strategies already built correctly in
[`mnemosyne-python`](https://github.com/mnemosyne-cache/mnemosyne-python),
now in Go, written idiomatically for Go rather than ported
line-by-line. Part of the multi-language fluency goal behind the whole
`mnemosyne-cache` org: build real, comparable understanding of the
same data structures and algorithms across different language
paradigms, not just "have it in more than one language."

## Scope

Mirrors `mnemosyne-python`'s MVP scope, adapted to Go:

- **LRU cache**: `Get()` / `Put()` / delete, fixed capacity, O(1)
  operations, capacity-based eviction only (no TTL)
- **TTL / expiring cache**: `Get()` / `Put()` / delete, time-based
  expiration as the primary mechanism, with capacity-based eviction as
  a backstop against unbounded growth — inherited directly from
  `mnemosyne-python`'s DESIGN.md (Decided, 2026-08-24: a burst of
  `Put()` calls could otherwise grow the cache unbounded). `Put()`,
  `Get()`, and `Has()` all refresh an entry's expiration on access
  (sliding TTL), same inherited decision.
- Explicit method API (not a map-mimicking interface)
- Single-threaded / non-concurrent for the first pass; no locking —
  revisit concurrent access (goroutines, `sync.Mutex`/`sync.RWMutex`)
  once both cache types are solid
- Fixed, concrete key/value types for the first pass; revisit Go
  generics (type parameters) once the concrete-type versions are
  correct — sequencing already anticipated in `mnemosyne-python`'s
  DESIGN.md
- Unit tests covering: basic get/set, eviction/expiration behavior,
  capacity/duration boundary conditions, updating an existing key
  (should not evict), and edge cases around zero/negative capacity or
  zero/negative TTL

## Status

Not started. This repo currently contains only scaffolding docs
(`CLAUDE.md`, `README.md`, this file) — no Go code yet. Sequenced
after `mnemosyne-python`'s MVP, which shipped complete on 2026-08-25.

## Design Decisions

### Why Go, alongside Python (and Rust)
See `mnemosyne-python`'s DESIGN.md — the goal isn't just "have a
caching library," it's building real, comparable fluency in
implementing the same data structure and algorithm correctly across
different language paradigms (dynamic vs. statically typed, GC vs.
manual/ownership-based memory management, idiomatic interfaces in each
ecosystem).

### Implementation approach — LRU cache
Same core structure as the Python implementation: hash map + doubly
linked list for O(1) get/put/evict. Written idiomatically for Go —
explicit struct definitions, no unnecessary abstraction. Avoid
reaching for Go's `container/list` package as a first-pass shortcut —
build the linked list manually first to actually understand the
mechanics, the same "understand it before shortcutting it" reasoning
already applied on the Python side.

### Implementation approach — TTL / expiring cache
Same core structure as the Python implementation: a map of key to
(value, expiration time), with a linear scan for the eviction
candidate under capacity pressure (an accepted tradeoff on the Python
side — revisit if it matters in practice here too). No linked list —
expiry is a per-entry age check, not a relative ordering between
entries.

### Error handling
Go's explicit `(value, error)` return convention should be used
consistently rather than reaching for panics for anything that isn't
a genuine programmer error. Invalid constructor arguments may still
warrant a panic (matching how Python's version raises `ValueError`
from `__init__`), but a normal cache miss on `Get()` should never be
an error — same as Python's version returning `nil`/`None`. Exact
convention TBD once implementation starts (see Open Questions).

## Non-Goals (for now)

- Hand-rolled distributed/multi-node caching protocol — matches
  `mnemosyne-python`'s Non-Goals; Redis is the intended backend,
  post-MVP, project-wide
- Persistence to disk
- Production-grade performance tuning — correctness first
- Generics / type parameters — deferred until concrete-type versions
  are correct (see Scope)
- Concurrency (goroutines, locking) — deferred until both cache types
  are solid single-threaded (see Scope)

## Open Questions

- Exact ordering relative to `mnemosyne-rust` and `mnemosyne-hw` —
  not yet decided (see the org's overall roadmap)
- Error handling convention for invalid constructor args (panic vs.
  returning an error from a constructor function) — Go idiom favors
  avoiding panics where possible; worth deciding explicitly once
  implementation starts rather than defaulting silently
