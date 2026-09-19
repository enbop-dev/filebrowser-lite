# WASI backend

Minimal Rust/WASI backend for File Browser Lite. See the
[main README](../README.md) for the project overview and supported features.

## Build From Source

Build the frontend before compiling the WASI component:

```bash
cd frontend
corepack pnpm install --frozen-lockfile
FILEBROWSER_LITE_WASI=1 corepack pnpm exec vite build

cd ../filebrowser-lite-wasi
cargo build --target=wasm32-wasip2 --release
```

To force a fresh WASI rebuild, run `cargo clean` before `cargo build`.

## Run Locally

Run it from the repository root:

```bash
mkdir -p data

wasmtime run \
  -Scli -Stcp -Sinherit-network \
  --dir data \
  ./filebrowser-lite-wasi/target/wasm32-wasip2/release/filebrowser-lite-wasi.wasm \
  --listen 127.0.0.1:8082
```

## Implementation Notes

- Lite-mode uploads send file bytes as the raw HTTP request body; multipart
  parsing and TUS uploads are not included.
- Routes validate uploads before reading the body. Accepted uploads stream to a
  temporary file beside the destination, then copy to the destination with a
  bounded buffer after the complete body arrives. A broken request leaves the
  existing file untouched and cleans up its temporary file. This needs staging
  disk space proportional to the upload; final filesystem write failures are not
  transactional, and abruptly killing the process can leave a temporary file.
- Upload responses return resource metadata without echoing text content. GET
  resource responses retain the existing text-content behavior.
- Responses that leave a request body unread close the HTTP/1 connection instead
  of draining that body. Completely read uploads support connection reuse.
- Request paths are normalized and reject `..` traversal.
- One long-running component process owns the Tokio runtime and HTTP listener,
  so all requests share the same process-level state.
- The guest can only access directories mounted with `wasmtime run --dir`.
- Rebuild the frontend before recompiling the component when frontend assets
  change.
- The repository's `.cargo/config.toml` enables Tokio's unstable WASIp2 network
  support during native and component builds.

## Upload Regression Tests

The HTTP tests require Python 3.11+ and use only its standard library. They start
one server with temporary data, exercise rejected/slow bodies, chunked uploads,
connection reuse, interrupted uploads, concurrent creates, and large text
uploads, then stop their server and remove the temporary data.

From `filebrowser-lite-wasi/`:

```bash
cargo test --locked
cargo build --locked
python3 tests/http_uploads.py -- target/debug/filebrowser-lite-wasi --listen '127.0.0.1:{port}'
```

To test the component through Fungi with a 64 MiB guest memory ceiling (including
an accepted 48 MiB text upload):

```bash
cargo build --locked --release --target wasm32-wasip2
python3 tests/http_uploads.py -- fungi run -Scli -Stcp -Sinherit-network \
  -W max-memory-size=67108864 --dir '{data}::data' \
  target/wasm32-wasip2/release/filebrowser-lite-wasi.wasm --listen '127.0.0.1:{port}'
```

`{data}` and `{port}` are substituted by the test runner. These checks do not
address synchronous filesystem/image work blocking the current-thread runtime.
