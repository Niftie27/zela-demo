# call-and-test

*Snapshot taken 2026-05-05 from `zela-demo/run-procedure.sh` and the `#[tokio::test]` block in `priority_fees`. Source of truth for behavior is `https://docs.zela.io/`.*

## 1. Native unit test shapes

- **With RPC** (e.g. `priority_fees`): the procedure exposes an inherent `pub async fn run(input, rpc) -> Result<Output, String>`, and the `CustomProcedure` impl is gated `#[cfg(target_arch = "wasm32")]`. Tests build a native `solana_client::nonblocking::rpc_client::RpcClient` (e.g. against `https://api.mainnet-beta.solana.com`) and call the inherent `run`. See `procedure-anatomy.md` for the full pattern.
- **Without RPC** (e.g. `hello_world`): no cfg gymnastics — `#[tokio::test]` calling `MyProc::run(input).await` directly.

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

Standard `client_credentials` flow against `ZELA_TOKEN_URL` with HTTP Basic `(ZELA_CLIENT_ID, ZELA_PRIVATE_KEY)`. Zela-specific bits:

- Full scope: `zela-builder:read zela-builder:write zela-executor:call`.
- Read-only invoke (`run-procedure.sh`): `zela-executor:call`.

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
