# procedure-anatomy

*Snapshot taken 2026-05-05 from `hello_world`, `priority_fees`, `block_time` in zela-demo. Re-verify against the linked files if details look off.*

Compact reference for trait/struct/error/log details. Use this once a procedure compiles and is callable; for the build/call pipeline see `compile-and-setup.md` and `call-and-test.md`.

## Three patterns observed

| Pattern | Example | When to use |
| --- | --- | --- |
| **Direct trait impl** | `hello_world` | No external RPC, no native test needed |
| **Direct trait impl with `zela_std::rpc_client::RpcClient`** | `block_time` | Uses Zela's RPC proxy but no native unit test |
| **cfg-gated trait + inherent native run** | `priority_fees` | Uses RPC *and* wants `cargo test` against a real Solana endpoint |

## Pattern A: direct (hello_world)

```rust
use serde::{Deserialize, Serialize};
use zela_std::{CustomProcedure, RpcError, zela_custom_procedure};

pub struct HelloWorld;

#[derive(Deserialize, Debug)]
pub struct Input { first_number: i32, second_number: i32 }

#[derive(Serialize)]
pub struct Output { pub sum: i32 }

impl CustomProcedure for HelloWorld {
    type Params      = Input;
    type ErrorData   = ();
    type SuccessData = Output;

    async fn run(params: Self::Params) -> Result<Self::SuccessData, RpcError<Self::ErrorData>> {
        if params.first_number == 0 {
            return Err(RpcError {
                code: 400,
                message: String::from("number cannot be 0"),
                data: None,
            });
        }
        Ok(Output { sum: params.first_number + params.second_number })
    }

    const LOG_MAX_LEVEL: log::LevelFilter = log::LevelFilter::Debug;
}

zela_custom_procedure!(HelloWorld);
```

## Pattern B: cfg-gated (priority_fees)

Two `impl` blocks: an inherent one with the real logic (testable natively), plus a `CustomProcedure` impl gated to wasm that maps native errors to `RpcError`.

```rust
#[cfg(target_arch = "wasm32")]
use zela_std::rpc_client::RpcClient;
#[cfg(not(target_arch = "wasm32"))]
use solana_client::nonblocking::rpc_client::RpcClient;

pub struct MyProc;

impl MyProc {
    pub async fn run(p: &Input, rpc: &RpcClient) -> Result<Output, String> {
        // real logic; uses `?` and returns String for errors
    }
}

#[cfg(target_arch = "wasm32")]
mod zela {
    use super::*;
    use zela_std::{CustomProcedure, RpcError, zela_custom_procedure};

    impl CustomProcedure for MyProc {
        type Params      = Input;
        type ErrorData   = ();
        type SuccessData = Output;
        const LOG_MAX_LEVEL: log::LevelFilter = log::LevelFilter::Info;

        async fn run(params: Self::Params) -> Result<Self::SuccessData, RpcError<Self::ErrorData>> {
            let rpc = RpcClient::new();
            MyProc::run(&params, &rpc).await.map_err(|message| RpcError {
                code: 1,
                message,
                data: None,
            })
        }
    }
    zela_custom_procedure!(MyProc);
}
```

Crucially: the `zela_custom_procedure!(...)` macro lives **inside** the `mod zela` so it only emits exports on the wasm target.

## Trait surface

```rust
type Params;       // Deserialize — request body shape
type SuccessData;  // Serialize — happy-path response
type ErrorData;    // Serialize — typed error payload (or `()` / `JsonValue`)
const LOG_MAX_LEVEL: log::LevelFilter;
async fn run(params: Self::Params) -> Result<Self::SuccessData, RpcError<Self::ErrorData>>;
```

## `RpcError` shape

```rust
RpcError { code: i64, message: String, data: Option<Self::ErrorData> }
```

- `code: 1` is the convention for "generic native error mapped to RpcError" in `priority_fees`.
- `code: 400` (or any HTTP-like code) is a free choice when the procedure has its own error taxonomy — see `hello_world`.
- For richer payloads, set `type ErrorData = JsonValue;` (see `block_time`) or a custom `Serialize` struct, and put detail in `data`.

## Logging

- `LOG_MAX_LEVEL` is a hard ceiling; messages above this level are suppressed at runtime. Use `log::LevelFilter::Debug` while developing, `Info` for production-like procedures.
- Phase-tagged style: `log::info!("[PHASE] field={}, ms={}", x, y)` — makes log scanning easier across phases of a multi-step procedure.

## Serde conventions

- Inputs derive `Deserialize` (often plus `Debug`).
- Outputs derive `Serialize`.
- Enum responses with case-style: `#[serde(rename_all = "snake_case")]`.
- Untagged input enums for "either-or" params: `#[derive(Deserialize)] #[serde(untagged)] pub enum Input { Latest { ... }, Specific { ... } }` — pattern used in `priority_fees`.

## What's NOT in this snapshot

Runtime limits (concurrency, procedure-to-procedure calls, shred streams, scheduling), build lifecycle states, and request-routing semantics live in the Zela docs. Fetch via `discovery.md`.
