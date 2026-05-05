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

Standard `client_credentials` flow against `ZELA_TOKEN_URL` with HTTP Basic `(ZELA_CLIENT_ID, ZELA_PRIVATE_KEY)`. Per docs the IDP URL is `https://auth.zela.io/realms/zela/protocol/openid-connect/token`.

Per docs *Scope Types*:

- `zela-executor:call` — execute a procedure / call Zela JSON-RPC.
- `zela-builder:read` + `zela-builder:write` — both required to upload a manually-built procedure.

Pick scopes based on what you'll do: invoke only → `zela-executor:call`; upload + invoke → all three.

## 4. Upload `.wasm` (manual upload only)

Used when the procedure is not GitHub-linked. From the docs §*Manual Upload*:

- Method: `POST {ZELA_CORE_URL}/procedures/<procedure>/wasm`
- Headers: `Authorization: Bearer <token>`, `Content-Type: application/wasm`
- Query: `project=<project_uuid>`, `file_name=<filename>`
- Body: raw WASM bytes
- Required scopes on the JWT: `zela-builder:read` *and* `zela-builder:write`
- Project UUID lives in the API key (`rpk-<PROJECT_UUID>-<INTERNAL>`) or the Dashboard

Identifier hash for manually uploaded procedures = **SHA-1 of the WASM bytes**. For GitHub-linked procedures, the identifier hash is the **Git commit hash** instead — the Builder produces it.

## 5. Execute (JSON-RPC 2.0)

- Method: `POST {ZELA_EXECUTOR_URL}` (`https://executor.zela.io`)
- Headers (required): `Authorization: Bearer <token>`, `Content-Type: application/json`
- Body shape:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "zela.<procedure>#<identifier_hash>",
  "params": { /* matches the procedure's Params */ }
}
```

- The `#<identifier_hash>` suffix is **always required**. It is the Git commit hash (Builder-built) or the SHA-1 of the WASM (manual upload).
- Optional header `zela-route-by`: `auto` (default — follows the Solana Leader) or `static <instance_label>` (e.g. `static fr2`). Default is `auto` if the header is omitted.

## 6. End-to-end pipeline (manual-upload variant)

```
cargo build --release --target wasm32-wasip2 --package <name>
        ↓
target/wasm32-wasip2/release/<name>.wasm
        ↓
hash = sha1_hex(bytes)
        ↓
POST {token}  → access_token            (scopes: zela-builder:read/write to upload, zela-executor:call to invoke)
        ↓
POST {core}/procedures/<name>/wasm      (Bearer; application/wasm; ?project=...&file_name=...)
        ↓
POST {executor}  body method = "zela.<name>#<hash>"  → result
```

For GitHub-linked procedures: skip the upload row; `<hash>` is the commit hash that produced the Success build.

## 7. Calling a procedure with `solana_client::RpcClient::send`

`RpcClient::send` requires `params` to be an array. To make your procedure callable from a Rust client that uses it, accept `[YourParams; 1]` and destructure:

```rust
type Params = [YourParams; 1];

async fn run(params: Self::Params) -> Result<Self::SuccessData, RpcError<Self::ErrorData>> {
    let [params] = params;
    // ...
}
```

(From the docs *Common Debugging Steps*.)

## 8. Pointers

- `zela-demo/run-procedure.sh` — minimal shell-only invoker.
- `https://docs.zela.io/` — authoritative behavior spec; refetch when in doubt.
