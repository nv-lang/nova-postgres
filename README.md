# nova-postgres

The PostgreSQL frontend/backend wire protocol for [Nova](https://github.com/nv-lang/nova),
written as pure functions: bytes become protocol messages, messages become bytes,
and authentication (SCRAM-SHA-256, MD5) runs as pure state steps. The package never
opens a socket — the caller moves the bytes, the same shape as `nova-tls`.

**Status: work in progress.** This is slice D8a of the Nova plan 286 (database
driver). Type codecs and the `Db` effect facade come in later slices.

License: MIT OR Apache-2.0.
