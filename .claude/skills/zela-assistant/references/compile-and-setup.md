# compile-and-setup

*Snapshot taken 2026-05-05 from `zela-demo/Cargo.toml`, `zela-demo/{hello_world,priority_fees,block_time}/Cargo.toml`. Verify against live files if anything contradicts the workspace.*

## 1. Toolchain prerequisites

- Target: `wasm32-wasip2`.
- WASI SDK: optional Nix shell at `zela-demo/shell.nix` pins it.

## 2. Workspace integration

Root `Cargo.toml` shape (zela-demo):

```toml
[workspace]
members = ["block_time", "hello_world", "priority_fees"]
resolver = "3"

[workspace.dependencies]
zela-std = { git = "https://github.com/Zela-io/zela-std.git", rev = "<pinned-sha>" }
serde   = { version = "1.0.228", features = ["derive"] }
log     = { version = "0.4" }
```

**Always read the live root `Cargo.toml` for the current `rev`** — do not paste a rev from this snapshot. Adding a new procedure means appending its folder name to `members`.

## 3. Per-crate `Cargo.toml` shape

Minimal (no Solana RPC), modeled on `hello_world`:

```toml
[package]
name    = "<name>"
version = "0.1.0"
edition = "2024"

[lib]
crate-type = ["cdylib"]

[dependencies]
zela-std.workspace = true
serde.workspace    = true
log.workspace      = true
```

Solana-aware variant (modeled on `priority_fees`) — only when the procedure needs Solana types or wants native unit tests against a real RPC:

```toml
[dependencies]
zela-std.workspace = true
serde.workspace    = true
log.workspace      = true
solana-sdk         = { version = "2.2" }   # or .workspace = true if the workspace pins it
# Crates that touch EncodedTransaction / UiMessage also need:
# solana-transaction-status-client-types = { version = "2" }

[target.'cfg(not(target_arch = "wasm32"))'.dependencies]
solana-client = { version = "2.2" }

[dev-dependencies]
tokio       = { version = "1", features = ["full"] }
env_logger  = { version = "0.11" }
```

Note: `block_time` diverges from the example above — it pins `solana-sdk = "3.0.0"`, declares `log = "0.4.28"` directly (not via the workspace), and adds `chrono = { version = "0.4.42", features = ["serde"] }`. There is no enforced uniformity across demo crates; read the live `Cargo.toml` of the crate you're editing.

Key rules:

- `crate-type = ["cdylib"]` is mandatory — Zela loads the resulting `.wasm` as a dylib.
- Native-only deps (sockets, `solana-client`) must live under `[target.'cfg(not(target_arch = "wasm32"))'.dependencies]` or the wasm build breaks with linker errors.

## 4. cfg-gating snippet (use both directions)

```rust
#[cfg(target_arch = "wasm32")]
use zela_std::rpc_client::RpcClient;

#[cfg(not(target_arch = "wasm32"))]
use solana_client::nonblocking::rpc_client::RpcClient;
```

When the procedure uses an RPC client *and* has a native `#[cfg(test)]` block, also wrap the `CustomProcedure` impl itself in `#[cfg(target_arch = "wasm32")] mod zela { ... zela_custom_procedure!(X); }` and provide an inherent `impl X { pub async fn run(p, rpc) -> Result<_, String> { ... } }` for the native side. See `references/procedure-anatomy.md` for the full pattern.

## 5. Build the WASM artifact

```
cargo build --release --target wasm32-wasip2 --package <name>
```

Output: `target/wasm32-wasip2/release/<name>.wasm`. Native `cargo check` / `cargo test` work as usual.

## 6. `run-procedure.sh` pointer

`zela-demo/run-procedure.sh` is the official one-line CLI invoker. It uses different env vars from the Python helpers (`ZELA_PROJECT_KEY_ID`, `ZELA_PROJECT_KEY_SECRET`) and hardcodes the auth + executor URLs. Read it before recommending a custom invocation script.
