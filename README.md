# capa-http

A capability-typed HTTP client for Capa, backed by Python's
`urllib.request` via the Unsafe boundary. The headline shape:

- The `Unsafe` is acquired once at the program's wiring point.
- A factory (`make_urllib_client`) returns an `Http`-typed client.
- Application code declares `http: Http` and exercises HTTP
  through the capability surface. **The `Unsafe` is invisible
  from there.**
- An auditor reading the manifest of an `Http`-using function
  sees `[Http, ...]` and nothing more.

## Status

v0.1 (seed library). The surface is the smallest API that lets a
real program talk to an HTTP API with custom methods, headers,
and bodies. Out of scope for v1: single-host attenuation,
streaming bodies, redirects (status 3xx returns to the caller
unchanged), proxies, TLS verification toggles.

## Quick start

```capa
import capa_http.http
import capa_http.urllib_client

fun fetch_users(http: Http) -> Result<Response, HttpError>
    let req = get("https://api.example.com/users")
    return http.send(req)

fun main(stdio: Stdio, u: Unsafe)
    let http = make_urllib_client(u)
    match fetch_users(http)
        Ok(resp)                  -> stdio.println("status: ${resp.status}")
        Err(NetworkFailure(msg))  -> stdio.eprintln("network: ${msg}")
        Err(InvalidRequest(msg))  -> stdio.eprintln("invalid: ${msg}")
```

Consumed via Capa's package manager. Declare the dependency in
your project's `capa.toml` (audit-grade form with the publisher's
GPG fingerprint):

```toml
[package]
name = "my-project"
version = "0.1.0"

[dependencies.capa_http]
git = "https://github.com/nelsonduarte/capa_http"
tag = "v0.1.1"
verify_key = "6C1D222D491FB88031E041A536CFB426101AA24B"
```

Then `capa install` fetches the library into `./vendor/`, runs
`git verify-tag` against your GPG keyring (import the publisher's
key first; see [`SECURITY.md`](SECURITY.md)), and the loader
picks it up automatically. The `urllib_helper.py` Python
side-module is materialised next to `urllib_client.capa` so the
factory's `py_import` path resolves without `sys.path` games.

The full runnable example is [`example.capa`](./example.capa);
it makes real HTTP calls to `httpbin.org` so an Internet
connection is required.

## API surface

### Types (from `capa_http.http`)

- `Request { method, url, headers, body, timeout_secs }`
- `Response { status, headers, body }`
- `HttpError = NetworkFailure(String) | InvalidRequest(String)`

A 4xx or 5xx status from the server is NOT an error: it is a
successful `Response` with that status. The caller checks
`response.status` and decides. `HttpError` is reserved for
transport-layer failures (DNS, connection refused, timeout,
TLS handshake failure) and for locally-rejected requests
(malformed URL, etc.).

### Capability (from `capa_http.http`)

```capa
pub capability Http
    fun send(self, req: Request) -> Result<Response, HttpError>
```

### Convenience constructors (from `capa_http.http`)

- `get(url) -> Request` : GET with 30 s timeout
- `post_json(url, body) -> Request` : POST with
  `Content-Type: application/json`

### Implementor (from `capa_http.urllib_client`)

- `pub type UrllibHttpClient { u: Unsafe }`
- `pub fun make_urllib_client(u: Unsafe) -> UrllibHttpClient`

## Audit claim

A function that declares `http: Http` exercises HTTP and
nothing else. `capa --manifest example.capa` shows the bound:

```
fetch_demo: ['Http', 'Stdio']
post_demo:  ['Http', 'Stdio']
main:       ['Stdio', 'Unsafe']
```

The `Unsafe` is contained in `main` (where the client is
constructed). The application code never sees it.

## Implementation notes

- The Capa side serialises Request fields as JSON (headers
  as a list of `[k, v]` pairs, body as a JSON string or
  `null`) and routes through a small Python helper
  ([`urllib_helper.py`](./urllib_helper.py)) that handles the
  actual `urllib.request` call. The helper catches every
  Python exception and surfaces failures as a JSON envelope,
  so nothing raises across the `Unsafe` boundary.
- Headers on the response are returned as a list of pairs to
  preserve duplicate-key semantics (`Set-Cookie`-style).
- The body is decoded as UTF-8 with `errors="replace"`; binary
  bodies are out of scope for v1.

## Known limitations (v1)

- **No user-defined-capability method type inference yet**: the
  Capa analyser does not propagate the return type of
  `http.send(req)` automatically (only built-in capability
  methods get full inference). The example uses an explicit
  `let resp: Response = http.send(req)?` annotation to recover
  the type information; downstream method dispatch
  (`resp.body.length()`) then resolves correctly. A future
  Capa release that extends method dispatch to user-defined
  capabilities will remove this need.
- No automatic redirect following. Status 3xx is returned to
  the caller.
- No streaming. Bodies are buffered in memory.
- No TLS verification toggles. Whatever Python's `urllib`
  defaults are.
- Headers on the request are passed to `urllib.request.Request`
  via a dict, which collapses duplicate-key request headers.
  Server-side response headers preserve duplicates.
