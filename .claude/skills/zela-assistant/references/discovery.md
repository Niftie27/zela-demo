# discovery

Routing guide — read when a snapshot may be stale, when the user asks about something not covered, or when the workspace's actual files contradict a snapshot.

## When to refetch

- The user's question concerns runtime *behavior* (limits, lifecycle, scheduling, routing) — these aren't snapshotted here, only docs.zela.io is authoritative.
- The user mentions an `zela_std` type, method, or feature flag not present in `procedure-anatomy.md`.
- A snapshot fact contradicts what's actually in the workspace's `Cargo.toml`, `Cargo.lock`, or source files.
- CI/local build emits an error string not in `compile-and-setup.md`'s failure table.
- The snapshot date in `metadata.snapshot_date` is older than the user's question's scope (e.g. "the new feature added last week").

When any of these hits, fetch live and prefer the live source over the snapshot.

## Source map

| Source | URL | Authoritative for |
| --- | --- | --- |
| Zela docs | `https://docs.zela.io/` | Build lifecycle states, request routing / instance groups, runtime *Limitations* (concurrency, no procedure-to-procedure calls, no shred streams, no scheduled execution), JSON-RPC execution semantics, JWT auth |
| Demo procedures | `https://github.com/Zela-io/zela-demo` | Example crates, workspace shape, the Cargo `rev` pin |
| SDK source | `https://github.com/Zela-io/zela-std` | `CustomProcedure` trait, `RpcError`, `RpcClient`, `JsonValue`, the `zela_custom_procedure!` macro. **Always look at the rev pinned in the workspace `Cargo.toml`** |

## How to fetch

- Prose / behavior: `WebFetch https://docs.zela.io/`. The site is a single-page Jekyll doc; fetch once and search by heading.
- SDK signatures at the pinned rev:
  - First `grep` the workspace `Cargo.toml` for the `zela-std` `rev = "<sha>"`.
  - Then `WebFetch https://raw.githubusercontent.com/Zela-io/zela-std/<sha>/src/lib.rs` (or whatever path within the repo applies).
- Alternative SDK source: `WebFetch https://github.com/Zela-io/zela-std/tree/<sha>` to discover the file layout first.

## Reconciling conflicts

If the live source contradicts a snapshot, **trust the live source**. If you have permission to edit the skill, update the relevant `references/*.md`:

1. Replace the stale fact.
2. Bump the `Snapshot taken: <date>` line at the top of the file.
3. Bump `metadata.snapshot_date` in `SKILL.md` if every reference has been refreshed.

Don't silently leave a contradiction in place.
