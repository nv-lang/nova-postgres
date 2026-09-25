# nova-postgres

Протокол PostgreSQL (клиент/сервер) для [Nova](https://github.com/nv-lang/nova),
написанный чистыми функциями: байты превращаются в сообщения протокола, сообщения —
в байты, соединение — чистый автомат состояний, а аутентификация (SCRAM-SHA-256,
MD5) идёт чистыми шагами состояния. Ядро протокола само сокет не открывает — байты
носит вызывающий, так же устроен `nova-tls`. Тонкий адаптер `pgnet` над `std.net` —
единственное место, где движется сокет.

**Статус: в работе.** Готовы срезы Д8а (протокол как чистые функции) и Д8б (автомат
сессии + адаптер сети) плана Nova 286 (драйвер БД). Кодеки типов и фасад эффекта
`Db` — в следующих срезах; TLS — вместе с `nova-tls`.

## Что здесь

- `src/` — модуль `postgres`:
  - `ClientMessage` — чистые кодировщики каждого клиентского сообщения срезов:
    `StartupMessage`, `SSLRequest`, `CancelRequest`, `PasswordMessage`,
    `SASLInitialResponse`, `SASLResponse`, `Query`, `Parse`, `Bind`,
    `Describe`, `Execute`, `Sync`, `Terminate`. Каждый возвращает точные
    байты протокола.
  - `ServerMessage.decode` / `ServerMessage.decode_all` — инкрементный разбор
    каждого серверного сообщения срезов (`Authentication*` всех видов,
    `ParameterStatus`, `BackendKeyData`, `ReadyForQuery`, `RowDescription`,
    `DataRow`, `CommandComplete`, `ErrorResponse`, `NoticeResponse`,
    `ParseComplete`, `BindComplete`, `NoData`, `EmptyQueryResponse`):
    короткий буфер отвечает «нужно ещё байтов», плохой кадр — структурная
    ошибка `PostgresError`, паники нет.
  - `Session` — соединение как чистый автомат (consume-тип): серверные байты
    на вход, клиентские на выход, события `PgEvent` (`Authenticated`,
    `Parameter`, `BackendKey`, `Ready` со статусом транзакции, `RowDesc`,
    `Row` с сырыми значениями `[]ParamValue`, `Complete`, `EmptyQuery`,
    `Notice`, `QueryError` с SQLSTATE). Фазы: `Authenticating → Startup →
    Ready ⇄ QuerySimple/QueryExtended`, плюс `Closed` и `Failed`.
    Аутентификация trust, cleartext, MD5 и SCRAM-SHA-256; после ошибки
    посреди запроса автомат пропускает всё до `ReadyForQuery`, закрывающего
    цикл; недопустимые переходы — структурная ошибка `Protocol`, не паника.
    `Session consume @close()` возвращает финальную порцию байт (Terminate —
    последним).
  - `ScramClient` — SCRAM-SHA-256 (RFC 5802 / RFC 7677) тремя чистыми шагами
    состояния с переданным извне клиентским nonce.
  - `md5_response` — легаси-ответ MD5-аутентификации.
  - `PostgresError` — один структурный домен ошибок на весь пакет.
- `src/pgnet/` — модуль `postgres.pgnet`: тонкий адаптер над `std.net`.
  `connect(host, port, user, password, database)` проходит фазу старта и
  возвращает готовое `Connection`; `simple_query` / `extended_query` гонят
  один круг запроса и возвращают события; `transaction_status` читает
  `I`/`T`/`E`; `close` отправляет финальный сброс. Клиентский nonce SCRAM
  берётся из системного генератора здесь, на краю.
- `examples/smoke/` — живой дым Д8а (сырой протокол поверх `std.net`).
- `examples/session-check/` — весь диалоговый набор Д8б (каждый записанный
  обмен против эталонных байт, каждый обмен ещё и разрезанный в каждой
  точке, недопустимые переходы, статус транзакции) как исполняемая
  программа: печатает `SESSION CHECKS OK`.
- `examples/live/` — живой прогон критерия 5 Д8б против одноразового
  PostgreSQL в Docker на свободном порту, который выбирает Docker.
- `src/*_test.nv` — peer-тесты: каждый кодировщик против эталонных байт из
  документации протокола, каждый декодер против эталонных кадров, каждый
  собственный префикс с ответом «нужно ещё байтов», потоки, разрезанные в
  каждом месте, векторы аутентификации RFC 7677 / MD5, каждый диалог сессии
  против эталонных событий и байт клиента с разрезанием в каждой точке,
  недопустимые переходы и статус транзакции.

## Пример использования

```nova
import postgres.pgnet.{connect, Connection}

// единственное место, где движется сокет:
consume conn = match connect("127.0.0.1", 5432, "postgres", "pw", "postgres") {
    Ok(consume c) => c
    Err(e) => return
}
ro evs = conn.simple_query("SELECT 1")?   // []PgEvent до ReadyForQuery
ro status = conn.transaction_status()     // u8: 'I' | 'T' | 'E'
ro fin = conn.close()                     // финальный сброс с Terminate
```

Чистое ядро без адаптера:

```nova
import postgres.{Session, PgEvent}

consume s = Session.new("postgres", "pw", "postgres", nonce)
ro out = s.drain()                    // StartupMessage — отправить первым
ro evs = s.feed(bytes_from_server)?   // переживает любую нарезку байт
s.query("SELECT 1")?
```

## Запуск

```sh
./nova.sh test src        # тесты пакета
./nova.sh check src       # типизация
```

Живой дым (нужен Docker; фиксированный порт — см. примечание о приёмке Д8а):

```sh
docker run --rm -d --name nova-pg-smoke -e POSTGRES_PASSWORD=smoke -p 55440:5432 postgres:15
./nova.sh build examples/smoke/src/main.nv -o target/smoke.exe
./target/smoke.exe
docker stop nova-pg-smoke
```

Живые проверки сессии и прогон критерия 5 (порт выбирает Docker):

```sh
./nova.sh build examples/session-check/src/main.nv -o target/session-check.exe
./target/session-check.exe
docker run --rm -d --name nova-pg-d8b -e POSTGRES_PASSWORD=smoke -p 127.0.0.1::5432 postgres:15
docker port nova-pg-d8b 5432                     # -> 127.0.0.1:<порт>
./nova.sh build examples/live/src/main.nv -o target/live.exe
./target/live.exe 127.0.0.1 <порт>
docker stop nova-pg-d8b
```

Лицензия: MIT OR Apache-2.0.
