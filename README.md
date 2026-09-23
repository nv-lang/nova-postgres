# nova-postgres

The PostgreSQL frontend/backend wire protocol for [Nova](https://github.com/nv-lang/nova),
written as pure functions: bytes become protocol messages, messages become bytes,
and authentication (SCRAM-SHA-256, MD5) runs as pure state steps. The package never
opens a socket — the caller moves the bytes, the same shape as `nova-tls`.

**Status: work in progress.** This is slice D8a of the Nova plan 286 (database
driver). Type codecs and the `Db` effect facade come in later slices.

## What is here

- `src/` — module `postgres`:
  - `ClientMessage` — pure encoders for every client message of the slice:
    `StartupMessage`, `SSLRequest`, `PasswordMessage`, `SASLInitialResponse`,
    `SASLResponse`, `Query`, `Parse`, `Bind`, `Describe`, `Execute`, `Sync`,
    `Terminate`. Each returns the exact wire bytes.
  - `ServerMessage.decode` / `ServerMessage.decode_all` — incremental decoding
    of every server message of the slice (`Authentication*` of all kinds,
    `ParameterStatus`, `BackendKeyData`, `ReadyForQuery`, `RowDescription`,
    `DataRow`, `CommandComplete`, `ErrorResponse`, `NoticeResponse`,
    `ParseComplete`, `BindComplete`, `NoData`, `EmptyQueryResponse`):
    a short buffer answers "need more bytes", a bad frame is a structural
    `PostgresError` — never a panic.
  - `ScramClient` — SCRAM-SHA-256 (RFC 5802 / RFC 7677) as three pure state
    steps with a caller-injected client nonce.
  - `md5_response` — the legacy MD5 password response.
  - `PostgresError` — one structural error domain for the whole package.
- `examples/smoke/` — a live smoke against a throwaway PostgreSQL in Docker:
  startup → SCRAM-SHA-256 → `SELECT 1` → `RowDescription`, `DataRow`,
  `ReadyForQuery`. The only place a socket moves.
- `src/*_test.nv` — peer tests: every encoder against reference bytes from
  the protocol documentation, every decoder against reference frames, every
  proper prefix answering "need more bytes", streams cut at every byte, and
  the RFC 7677 / MD5 authentication vectors.

## Usage sketch

```nova
import postgres.{ClientMessage, ServerMessage, ScramClient}

// encode: message -> bytes
ro frame = ClientMessage.query("SELECT 1")

// decode: bytes -> message, incrementally
match ServerMessage.decode(buffer) {
    Ok(Message { msg, consumed }) => { /* buffer[consumed..] is the tail */ }
    Ok(NeedMoreBytes) => { /* read more bytes, call again */ }
    Err(e) => { /* structural error, e.kind classifies it */ }
}
```

## Running

```sh
./nova.sh test src        # the package test suite
./nova.sh check src       # type-check
```

Live smoke (needs Docker):

```sh
docker run --rm -d --name nova-pg-smoke -e POSTGRES_PASSWORD=smoke -p 55432:5432 postgres:15
./nova.sh build examples/smoke/src/main.nv -o target/smoke.exe
./target/smoke.exe
docker stop nova-pg-smoke
```

License: MIT OR Apache-2.0.
