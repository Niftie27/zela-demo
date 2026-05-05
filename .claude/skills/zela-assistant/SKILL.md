---
name: zela-assistant
description: Entry point for authoring Zela procedures (Rust crates compiled to wasm32-wasip2 implementing zela-std's CustomProcedure trait) and integrating with the Zela executor/core API. Use when creating a new procedure crate, editing existing ones (hello_world, priority_fees, block_time in zela-demo), debugging build/execution errors, asking about CustomProcedure, RpcError, zela_custom_procedure!, wasm32-wasip2, OAuth/upload/JSON-RPC integration, or anything Zela-specific. Bundles concrete-fact references extracted as snapshots from the demo + test repos, plus a procedure scaffold under assets/templates/, and routes Claude to canonical sources (docs.zela.io, github.com/Zela-io) when a snapshot may be stale. Trigger whenever the user edits a Rust file that imports zela_std, calls the Zela executor/core API, or asks "how do I X with Zela". Do NOT use for general Solana program development unrelated to Zela or Zela platform operations.
license: MIT
metadata:
  version: 0.1.0
  snapshot_date: 2026-05-05
---

# Zela Assistant

Help an author go from a clean checkout to a deployed-and-callable Zela procedure: scaffold the crate, compile it to `wasm32-wasip2`, run native unit tests, then upload + invoke via JSON-RPC.

The skill is a **snapshot + discovery hybrid**. References under `references/` cache concrete facts extracted from the live repos on the date in the frontmatter. When in doubt, follow `references/discovery.md` to refetch from `https://docs.zela.io/` and the GitHub repos.

## When to use

Trigger when the user:

- Creates or edits a procedure crate inside a workspace that depends on `zela-std`.
- Sees a build failure with `wasm32-wasip2`, `cdylib`, `zela_std`, or cfg-gating around `RpcClient`.
- Asks about `CustomProcedure`, `RpcError`, the `zela_custom_procedure!` macro, `LOG_MAX_LEVEL`, or workspace dependency pinning.
- Wants to call a deployed procedure (OAuth → upload `.wasm` → JSON-RPC execute).
- Asks "how do I X with Zela?" or references docs.zela.io.

Do NOT use for:

- General Solana program development (Anchor, native programs) unrelated to Zela.
- Operating Zela platform infrastructure itself.
- Test-orchestration scripting that lives outside this repo.

## Quick start: zero to a working procedure

1. **Toolchain** — `rustup target add wasm32-wasip2`. If unavailable, `rustup update stable`. (See `references/compile-and-setup.md` for Nix/WASI SDK pinning.)
2. **Scaffold** — copy `assets/templates/new-procedure/` to `<repo-root>/<name>/`, replace `{{NAME}}` → struct name, set `[package].name`, register the crate in the workspace `members` array.
3. **Compile** — `cargo build --release --target wasm32-wasip2 --package <name>` → `target/wasm32-wasip2/release/<name>.wasm`.
4. **Native test** — `cargo test --package <name>`. (Requires the cfg-gated `RpcClient` pattern when the procedure uses `zela_std::rpc_client::RpcClient`. See `references/procedure-anatomy.md`.)
5. **Call deployed** — OAuth → upload `.wasm` → JSON-RPC execute. Full payload shapes in `references/call-and-test.md`.

Step-1 failure hint: `error: target wasm32-wasip2 not found` → install via `rustup`. Step-3 failure hint: `unresolved import zela_std::rpc_client` natively → see the cfg-gating pattern in `references/compile-and-setup.md`.

## Project structure

```
<workspace-root>/
├── Cargo.toml                    # workspace: members + workspace.dependencies (zela-std pinned by rev)
├── shell.nix                     # optional Nix env (pinned WASI SDK)
├── run-procedure.sh              # CLI invoker — read it before recommending custom scripts
├── hello_world/                  # simplest live example (no Solana deps)
├── priority_fees/                # Solana RPC client + cfg-gated trait impl + native test
├── block_time/                   # async + JsonValue error data
└── .claude/skills/zela-assistant/  # this skill
```

Per-crate layout: `<crate>/Cargo.toml` (cdylib, edition 2024) + `<crate>/src/lib.rs` (one struct + impl + `zela_custom_procedure!` + optional `#[cfg(test)] mod tests`).

## Map of references

| If the user is… | Open… |
| --- | --- |
| Setting up the toolchain, compiling, debugging build failures, structuring `Cargo.toml` | `references/compile-and-setup.md` |
| Running native tests, calling the deployed executor, uploading WASM, OAuth | `references/call-and-test.md` |
| Writing the trait impl, error mapping, logging conventions | `references/procedure-anatomy.md` |
| Asking anything not covered, or about behavior that may have changed | `references/discovery.md` |

## Stable invariants

Each ends with *"verify via `discovery.md` if uncertain"*:

- Procedure crate: `crate-type = ["cdylib"]`, `edition = "2024"`, target `wasm32-wasip2`.
- Build: `cargo build --release --target wasm32-wasip2 --package <name>`.
- Trait surface: `impl CustomProcedure for X { type Params; type SuccessData; type ErrorData; const LOG_MAX_LEVEL; async fn run; }` then `zela_custom_procedure!(X);`.
- Imports always include `use zela_std::{CustomProcedure, RpcError, zela_custom_procedure};`.
- cfg-gating for RPC: `#[cfg(target_arch = "wasm32")]` → `zela_std::rpc_client::RpcClient`; `#[cfg(not(target_arch = "wasm32"))]` → `solana_client::nonblocking::rpc_client::RpcClient` for native tests.

## Edit / debug playbooks

**Edit an existing procedure**

1. Read the crate's `Cargo.toml` and `src/lib.rs` first — patterns vary (hello_world has no cfg-gating; priority_fees wraps the trait impl in `#[cfg(target_arch = "wasm32")] mod zela { ... }`).
2. Preserve the `zela_custom_procedure!(...)` registration and the `LOG_MAX_LEVEL` constant.
3. Mirror the existing logging convention (e.g. phase-tagged `log::info!("[PHASE] ...")`).
4. Run `cargo test --package <name>` (if a `#[cfg(test)]` block exists), then `cargo build --release --target wasm32-wasip2 --package <name>`.

**Debug a Zela-specific build failure**

1. cfg-gating: `zela_std::rpc_client::RpcClient` must be gated `#[cfg(target_arch = "wasm32")]`; `solana-client` must live under `[target.'cfg(not(target_arch = "wasm32"))'.dependencies]`.
2. If symbols from `zela_std` don't match: re-read the workspace `Cargo.toml` `rev` and refetch `zela-std` source at that rev (see `references/discovery.md`).

## When to refetch

Snapshots in `references/` are dated in their headers. Refetch via `references/discovery.md` when:

- The user's question is about behavior that could have changed since the snapshot date.
- A snapshot's claim contradicts what's actually in the workspace (`Cargo.toml`, source files).
- The user asks about runtime limits, build lifecycle states, or SDK types not in `procedure-anatomy.md`.

## Anti-patterns

- Don't paraphrase the Zela docs at length; point at `https://docs.zela.io/`.
- Don't hardcode `zela-std` SDK signatures from memory; verify against the rev pinned in the workspace `Cargo.toml`.
- Don't conflate this skill with test-orchestration tooling that lives outside this repo.
- Don't add Solana-only deps without cfg-gating — they'll break the wasm build.
