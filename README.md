# nova-postgres

The PostgreSQL frontend/backend wire protocol for [Nova](https://github.com/nv-lang/nova),
written as pure functions: bytes become protocol messages, messages become bytes,
connections are pure state machines, and authentication (SCRAM-SHA-256, MD5) runs
as pure state steps. The protocol core never opens a socket — the caller moves the
bytes, the same shape as `nova-tls`. A thin `pgnet` adapter over `std.net` is the
only place a socket moves.

**Status: work in progress.** Slices D8a (protocol as pure functions), D8b
(session state machine + network adapter) and D8c (TLS negotiation step,
structured adapter errors) of the Nova plan 286 (database driver) are done.
Type codecs and the `Db` effect facade come in later slices. TLS is composed by
the consumer through `nova-tls` (see `examples/live`).

## What is here

- `src/` — module `postgres`:
  - `ClientMessage` — pure encoders for every client message of the slices:
    `StartupMessage`, `SSLRequest`, `CancelRequest`, `PasswordMessage`,
    `SASLInitialResponse`, `SASLResponse`, `Query`, `Parse`, `Bind`,
    `Describe`, `Execute`, `Sync`, `Terminate`. Each returns the exact wire
    bytes.
  - `ServerMessage.decode` / `ServerMessage.decode_all` — incremental decoding
    of every server message of the slices (`Authentication*` of all kinds,
    `ParameterStatus`, `BackendKeyData`, `ReadyForQuery`, `RowDescription`,
    `DataRow`, `CommandComplete`, `ErrorResponse`, `NoticeResponse`,
    `ParseComplete`, `BindComplete`, `NoData`, `EmptyQueryResponse`):
    a short buffer answers "need more bytes", a bad frame is a structural
    `PostgresError` — never a panic.
  - `Session` — the connection as a pure state machine (a consume type):
    feed server bytes in, drain client bytes out, read `PgEvent`s
    (`Authenticated`, `Parameter`, `BackendKey`, `Ready` with the transaction
    status, `RowDesc`, `Row` with raw `[]ParamValue` values, `Complete`,
    `EmptyQuery`, `Notice`, `QueryError` with the SQLSTATE). Phases:
    `Authenticating → Startup → Ready ⇄ QuerySimple/QueryExtended`, plus
    `Closed` and `Failed`. Trust, cleartext, MD5 and SCRAM-SHA-256
    authentication; after a mid-query error the machine skips to the
    `ReadyForQuery` that closes the cycle; illegal transitions are structural
    `Protocol` errors, never panics. `Session consume @close()` returns the
    final flush (Terminate last).
  - `ScramClient` — SCRAM-SHA-256 (RFC 5802 / RFC 7677) as three pure state
    steps with a caller-injected client nonce.
  - `md5_response` — the legacy MD5 password response.
  - `PostgresError` — one structural error domain for the whole package.
- `src/pgnet/` — module `postgres.pgnet`: the thin `std.net` adapter
  (plaintext). `open_tcp` / `request_ssl` (sends SSLRequest, classifies the
  answer byte with the pure `SslAnswer.classify`: `S` / `N` / any other byte
  is a structured protocol error) / `connect_plain` (start-up +
  authentication) build a Ready `Connection`; `simple_query` /
  `extended_query` run one query round; `transaction_status` reads `I`/`T`/`E`;
  `close` sends the final flush. Failures are structured `ConnError`
  variants (`Net`, `RefusedTls`, `Protocol`, `Auth`, `Server` with SQLSTATE
  and fields) — matchable, not strings. The SCRAM nonce is drawn from OS
  randomness here, at the edge. TLS itself is composed by the consumer
  through `nova-tls` (see `examples/live`; pgnet imports no tls types —
  third-party package types do not survive this compiler's cross-module
  boundary, see the module banner in `src/pgnet/conn.nv`).
- `examples/smoke/` — the D8a live smoke (raw protocol over `std.net`).
- `examples/session-check/` — the whole D8b dialog suite (every scripted
  exchange against reference bytes, each exchange also fed cut at every
  point, the illegal transitions, the transaction status) as a runnable
  program: prints `SESSION CHECKS OK`.
- `examples/live/` — the D8c live run against two throwaway Docker
  PostgreSQL containers on Docker-chosen free ports: TLS with your own CA
  (SCRAM over TLS 1.3, `pg_stat_ssl`, the whole D8b flow, transaction
  statuses T/E/I), structured TLS errors (hostname mismatch, unknown CA),
  and the plaintext modes (Require → structured RefusedTls, Prefer/Disable →
  plaintext). Declares the `tls` dependency on nova-tls.
- `src/*_test.nv` — peer tests: every encoder against reference bytes from
  the protocol documentation, every decoder against reference frames, every
  proper prefix answering "need more bytes", streams cut at every byte, the
  RFC 7677 / MD5 authentication vectors, every session dialog against
  reference events and client bytes with cut-at-every-point, the illegal
  transitions, and the transaction status.

## Usage sketch

```nova
import postgres.pgnet.{connect, Connection}

// the only place a socket moves:
consume conn = match connect("127.0.0.1", 5432, "postgres", "pw", "postgres") {
    Ok(consume c) => c
    Err(e) => return
}
ro evs = conn.simple_query("SELECT 1")?   // []PgEvent up to ReadyForQuery
ro status = conn.transaction_status()     // u8: 'I' | 'T' | 'E'
ro fin = conn.close()                     // final flush with Terminate
```

Pure core without the adapter:

```nova
import postgres.{Session, PgEvent}

consume s = Session.new("postgres", "pw", "postgres", nonce)
ro out = s.drain()                    // StartupMessage, send first
ro evs = s.feed(bytes_from_server)?   // survives any chunking
s.query("SELECT 1")?
```

## Running

```sh
./nova.sh test src        # the package test suite
./nova.sh check src       # type-check
```

Live smoke (needs Docker; uses a fixed port — see the D8a acceptance note):

```sh
docker run --rm -d --name nova-pg-smoke -e POSTGRES_PASSWORD=smoke -p 55440:5432 postgres:15
./nova.sh build examples/smoke/src/main.nv -o target/smoke.exe
./target/smoke.exe
docker stop nova-pg-smoke
```

Live session checks and the TLS live run (Docker picks free ports; the TLS
container recipe with your own CA is in `BRIEF-D8c.md`):

```sh
./nova.sh build examples/session-check/src/main.nv -o target/session-check.exe
./target/session-check.exe
# TLS container with your own CA (recipe in BRIEF-D8c.md) + a plain one:
docker run --rm -d --name nova-pg-d8c-tls  -e POSTGRES_PASSWORD=smoke -p 127.0.0.1::5432 postgres:15 bash -c '...'
docker run --rm -d --name nova-pg-d8c-plain -e POSTGRES_PASSWORD=smoke -p 127.0.0.1::5432 postgres:15
docker port nova-pg-d8c-tls 5432                 # -> 127.0.0.1:<tls-port>
docker port nova-pg-d8c-plain 5432               # -> 127.0.0.1:<plain-port>
docker cp nova-pg-d8c-tls:/var/lib/postgresql/ca.crt target/pg-ca.crt
./nova.sh build examples/live/src/main.nv -o target/live.exe
./target/live.exe 127.0.0.1 <tls-port>  ca target/pg-ca.crt localhost
./target/live.exe 127.0.0.1 <tls-port>  mismatch target/pg-ca.crt
./target/live.exe 127.0.0.1 <tls-port>  system target/pg-ca.crt localhost
./target/live.exe 127.0.0.1 <plain-port> require
./target/live.exe 127.0.0.1 <plain-port> prefer target/pg-ca.crt
./target/live.exe 127.0.0.1 <plain-port> disable
docker stop nova-pg-d8c-tls nova-pg-d8c-plain
```

License: MIT OR Apache-2.0.
