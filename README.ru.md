# nova-postgres

Протокол PostgreSQL (клиент/сервер) для [Nova](https://github.com/nv-lang/nova),
написанный чистыми функциями: байты превращаются в сообщения протокола, сообщения —
в байты, а аутентификация (SCRAM-SHA-256, MD5) идёт чистыми шагами состояния. Пакет
сам сокет не открывает — байты носит вызывающий, так же устроен `nova-tls`.

**Статус: в работе.** Это срез Д8а плана Nova 286 (драйвер БД). Кодеки типов и
фасад эффекта `Db` — в следующих срезах.

## Что здесь

- `src/` — модуль `postgres`:
  - `ClientMessage` — чистые кодировщики каждого клиентского сообщения среза:
    `StartupMessage`, `SSLRequest`, `PasswordMessage`, `SASLInitialResponse`,
    `SASLResponse`, `Query`, `Parse`, `Bind`, `Describe`, `Execute`, `Sync`,
    `Terminate`. Каждый возвращает точные байты протокола.
  - `ServerMessage.decode` / `ServerMessage.decode_all` — инкрементный разбор
    каждого серверного сообщения среза (`Authentication*` всех видов,
    `ParameterStatus`, `BackendKeyData`, `ReadyForQuery`, `RowDescription`,
    `DataRow`, `CommandComplete`, `ErrorResponse`, `NoticeResponse`,
    `ParseComplete`, `BindComplete`, `NoData`, `EmptyQueryResponse`):
    короткий буфер отвечает «нужно ещё байтов», плохой кадр — структурная
    ошибка `PostgresError`, паники нет.
  - `ScramClient` — SCRAM-SHA-256 (RFC 5802 / RFC 7677) тремя чистыми шагами
    состояния с переданным извне клиентским nonce.
  - `md5_response` — легаси-ответ MD5-аутентификации.
  - `PostgresError` — один структурный домен ошибок на весь пакет.
- `examples/smoke/` — живой дым против одноразового PostgreSQL в Docker:
  startup → SCRAM-SHA-256 → `SELECT 1` → `RowDescription`, `DataRow`,
  `ReadyForQuery`. Единственное место, где двигается сокет.
- `src/*_test.nv` — peer-тесты: каждый кодировщик — против эталонных байт из
  документации протокола, каждый декодер — против эталонных кадров, каждый
  собственный префикс — ответ «нужно ещё байтов», потоки, разрезанные в каждой
  точке, и векторы аутентификации RFC 7677 / MD5.

## Пример использования

```nova
import postgres.{ClientMessage, ServerMessage, ScramClient}

// кодирование: сообщение -> байты
ro frame = ClientMessage.query("SELECT 1")

// разбор: байты -> сообщение, инкрементно
match ServerMessage.decode(buffer) {
    Ok(Message { msg, consumed }) => { /* buffer[consumed..] — остаток */ }
    Ok(NeedMoreBytes) => { /* дочитать байты и вызвать снова */ }
    Err(e) => { /* структурная ошибка, класс — в e.kind */ }
}
```

## Запуск

```sh
./nova.sh test src        # тесты пакета
./nova.sh check src       # тип-проверка
```

Живой дым (нужен Docker):

```sh
docker run --rm -d --name nova-pg-smoke -e POSTGRES_PASSWORD=smoke -p 55432:5432 postgres:15
./nova.sh build examples/smoke/src/main.nv -o target/smoke.exe
./target/smoke.exe
docker stop nova-pg-smoke
```

Лицензия: MIT OR Apache-2.0.
