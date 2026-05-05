# call-and-test

*Snapshot taken 2026-05-05 from `zela-demo/run-procedure.sh` and the `#[tokio::test]` block in `priority_fees`. Source of truth for behavior is `https://docs.zela.io/`.*

## 1. Native unit test pattern

Two shapes are seen in the wild.

**Shape A — simple inherent run** (procedures with cfg-gated RPC, e.g. `priority_fees`):

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use solana_client::nonblocking::rpc_client::RpcClient;
    use solana_sdk::commitment_config::CommitmentConfig;

    #[tokio::test]
    async fn test() {
        let _ = env_logger::builder()
            .is_test(true)
            .parse_env(env_logger::Env::new().default_filter_or("info,priority_fees=debug")) // replace with your crate name
            .try_init();

        let rpc = RpcClient::new_with_commitment(
            "https://api.mainnet-beta.solana.com".to_owned(),
            CommitmentConfig::confirmed(),
        );
        let input = Input { /* ... */ };
        let out = MyProc::run(&input, &rpc).await.unwrap();
        println!("{out:#?}");
    }
}
```

This requires the procedure to expose an inherent `pub async fn run(input, rpc) -> Result<Output, String>` on the native side, and gate the `CustomProcedure` impl behind `#[cfg(target_arch = "wasm32")]`.

**Shape B — direct trait test** (procedures without external RPC, e.g. `hello_world`): just call `MyProc::run(input).await` from a `#[tokio::test]` — no cfg gymnastics needed.

## 2. Required env vars (Python helper flow)

| Name | Purpose |
| --- | --- |
| `ZELA_CLIENT_ID` | OAuth2 client ID |
| `ZELA_PRIVATE_KEY` | OAuth2 client secret (HTTP Basic password) |
| `ZELA_TOKEN_URL` | Token endpoint (full URL, no path suffix) |
| `ZELA_EXECUTOR_URL` | Executor endpoint (full URL) |
| `ZELA_CORE_URL` | Core API endpoint for WASM uploads |

The shell script `run-procedure.sh` instead reads `ZELA_PROJECT_KEY_ID` and `ZELA_PROJECT_KEY_SECRET` and hardcodes the auth + executor URLs.

## 3. OAuth2 (client_credentials)

- Method: `POST {ZELA_TOKEN_URL}`
- Auth: HTTP Basic `(ZELA_CLIENT_ID, ZELA_PRIVATE_KEY)`
- Body (form-encoded): `grant_type=client_credentials`, `scope=zela-builder:read zela-builder:write zela-executor:call`
- Response: JSON; extract `access_token`
- Shell-script scope is narrower: just `zela-executor:call` (read-only invoke).

## 4. Upload `.wasm`

- Method: `POST {ZELA_CORE_URL}/procedures/{procedure}/wasm`
- Headers: `Authorization: Bearer <token>`, `Content-Type: application/wasm`
- Query: `project={project_id}`, `file_name={filename}`
- Body: raw WASM bytes
- Versioning: SHA-1 hex of the bytes is the de-facto procedure version.
- `422` = duplicate hash (treated as no-op success). The local Python helper caches uploaded SHA-1s in `.cache/wasm_uploads.json` to skip re-uploads.

## 5. Execute (JSON-RPC 2.0)

- Method: `POST {ZELA_EXECUTOR_URL}`
- Headers: `Authorization: Bearer <token>`, `Content-Type: application/json`, optional `zela-route-by: <route>`, optional `zela-route-dbg: true`
- Body shape:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "zela.<procedure>#<sha1_hex>",
  "params": { /* matches the procedure's Params */ }
}
```

- The `#<sha1_hex>` suffix is **optional** — without it, the executor picks the routed default. With it, you pin to a specific upload.
- Debug response header (when `zela-route-dbg: true` was sent): `zela-routed-dbg` (JSON).

## 6. End-to-end pipeline

```
cargo build --release --target wasm32-wasip2 --package <name>
        ↓
target/wasm32-wasip2/release/<name>.wasm
        ↓ sha1
hash = sha1_hex(bytes)
        ↓
POST {token}  → access_token        (needed for both upload and execute)
        ↓ if hash not cached
POST {core}/procedures/<name>/wasm  (Authorization: Bearer; Content-Type: application/wasm)
        ↓
POST {executor}  body method = "zela.<name>#<hash>"  → result
```

## 7. Pointers

- `zela-demo/run-procedure.sh` — minimal shell-only invoker.
- `https://docs.zela.io/` — authoritative behavior spec; refetch when in doubt.
