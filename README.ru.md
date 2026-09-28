# nova-postgres

Протокол PostgreSQL (клиент/сервер) для [Nova](https://github.com/nv-lang/nova),
написанный чистыми функциями: байты превращаются в сообщения протокола, сообщения —
в байты, соединение — чистый автомат состояний, а аутентификация (SCRAM-SHA-256,
MD5) идёт чистыми шагами состояния. Ядро протокола само сокет не открывает — байты
носит вызывающий, так же устроен `nova-tls`. Тонкий адаптер `pgnet` над `std.net` —
единственное место, где движется сокет.

**Статус: в работе.** Готовы срезы Д8а (протокол как чистые функции), Д8б (автомат
сессии + адаптер сети) и Д8в (шаг согласования TLS, структурные ошибки адаптера)
плана Nova 286 (драйвер БД). Кодеки типов и фасад эффекта `Db` — в следующих
срезах. TLS компонуется потребителем через `nova-tls` (см. `examples/live`).

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
    `ParseComplete`, `BindComplete`, `NoData`, `EmptyQueryResponse`,
    `NotificationResponse` — асинхронное сообщение LISTEN/NOTIFY):
    короткий буфер отвечает «нужно ещё байтов», плохой кадр — структурная
    ошибка `PostgresError`, паники нет.
  - `Session` — соединение как чистый автомат (consume-тип): серверные байты
    на вход, клиентские на выход, события `PgEvent` (`Authenticated`,
    `Parameter`, `BackendKey`, `Ready` со статусом транзакции, `RowDesc`,
    `Row` с сырыми значениями `[]ParamValue`, `Complete`, `EmptyQuery`,
    `Notice`, `QueryError` с SQLSTATE, `Notify` — уведомление
    LISTEN/NOTIFY, легальное в любой фазе, даже посреди ответа). Фазы: `Authenticating → Startup →
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
- `src/pgnet/` — модуль `postgres.pgnet`: тонкий адаптер над `std.net`
  (открытый текст). `open_tcp` / `request_ssl` (шлёт SSLRequest и
  классифицирует байт ответа чистой `SslAnswer.classify`: `S` / `N` /
  любой иной байт — структурная ошибка протокола) / `connect_plain`
  (старт + аутентификация) строят готовое `Connection`;
  `simple_query` / `extended_query` гонят один круг запроса;
  `transaction_status` читает `I`/`T`/`E`; `close` отправляет финальный
  сброс. Строки идут потоком через само соединение: `conn.query(q)`
  отправляет один запрос, затем `for row in conn` / `conn.next()` выдают
  следующую запись в рантайме (читая с сокета ровно столько, сколько нужно);
  после цикла `conn.error()` держит ошибку, завершившую обход (читать, если
  цикл оборвался рано — полный for-in всегда доходит до ReadyForQuery, даже
  через серверную ошибку). Недочитанный ответ не может рассинхронизировать
  протокол: следующий запрос дочитает его сам; `conn.skip_answer()` — явная
  форма для досрочного выхода. `row.text_at(i)` декодирует текстовую
  колонку; `row.int_at` / `i16_at` / `i32_at` / `i64_at` / `f64_at` /
  `bool_at` — типизированные колонки (`bool_at` понимает формат 't'/'f';
  целые Postgres знаковые, без u*-вариантов). LISTEN/NOTIFY: уведомление
  складывается в очередь в момент прихода — в простое или посреди ответа —
  и читается `conn.next_event()` (блокирующе; только между ответами) или
  `conn.try_next_event()` (неблокирующий опрос) как
  `Notification { pid, channel, payload }`. Отказы — структурные
  варианты `ConnError` (`Net`, `RefusedTls`,
  `Protocol`, `Auth`, `Server` с SQLSTATE и полями) — они матчатся, а не
  разбираются по тексту. Клиентский nonce SCRAM берётся из системного
  генератора здесь, на краю. Сам TLS компонуется потребителем через
  `nova-tls` (см. `examples/live`; pgnet не импортирует типы tls — типы
  сторонних пакетов не переживают межмодульную границу этого компилятора,
  см. баннер модуля в `src/pgnet/conn.nv`).
- `examples/smoke/` — живой дым Д8а (сырой протокол поверх `std.net`).
- `examples/session-check/` — весь диалоговый набор Д8б (каждый записанный
  обмен против эталонных байт, каждый обмен ещё и разрезанный в каждой
  точке, недопустимые переходы, статус транзакции) как исполняемая
  программа: печатает `SESSION CHECKS OK`.
- `examples/live/` — живой прогон Д8в против двух одноразовых контейнеров
  PostgreSQL в Docker на свободных портах: TLS со своим CA (SCRAM поверх
  TLS 1.3, `pg_stat_ssl`, весь поток Д8б, статусы транзакции T/E/I),
  структурные ошибки TLS (несовпадение имени, неизвестный CA) и режимы
  открытого текста (Require → структурный RefusedTls, Prefer/Disable →
  открытый текст). Объявляет зависимость `tls` от nova-tls. Режим `multi`
  — два запроса подряд по одному соединению с обходом строк в for-in;
  режим `rowerr` — ошибка 42601 внутри итерации (for-in до конца, затем
  `conn.error()`) и следующий запрос по тому же соединению; режим `notify` —
  LISTEN/NOTIFY вживую: блокирующий `next_event()` и уведомление, пришедшее
  посреди запроса и дочитанное `try_next_event()`.
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

Поток строк, один запрос за раз по тому же соединению:

```nova
consume conn = match connect("127.0.0.1", 5432, "postgres", "pw", "postgres") {
    Ok(consume c) => c
    Err(e) => return
}
conn.query("SELECT generate_series(1, 5)")?
mut sum = 0
for row in conn {
    sum += row.int_at(0)?
}
match conn.error() {              // ошибка SQL обрывает цикл; читается здесь
    Some(e) => println("server error: ${e.to_str()}"),
    None => ()
}                                 // следующий запрос дочитает остаток сам —
                                 // забыть нечего
ro answer = conn.simple_query("SELECT 42")? // то же соединение, снова годно
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

Живые проверки сессии и TLS-прогон (порты выбирает Docker; рецепт TLS-
контейнера со своим CA — в `BRIEF-D8c.md`):

```sh
./nova.sh build examples/session-check/src/main.nv -o target/session-check.exe
./target/session-check.exe
# TLS-контейнер со своим CA (рецепт в BRIEF-D8c.md) и обычный:
docker run --rm -d --name nova-pg-d8c-tls  -e POSTGRES_PASSWORD=smoke -p 127.0.0.1::5432 postgres:15 bash -c '...'
docker run --rm -d --name nova-pg-d8c-plain -e POSTGRES_PASSWORD=smoke -p 127.0.0.1::5432 postgres:15
docker port nova-pg-d8c-tls 5432                 # -> 127.0.0.1:<tls-порт>
docker port nova-pg-d8c-plain 5432               # -> 127.0.0.1:<plain-порт>
docker cp nova-pg-d8c-tls:/var/lib/postgresql/ca.crt target/pg-ca.crt
./nova.sh build examples/live/src/main.nv -o target/live.exe
./target/live.exe 127.0.0.1 <tls-порт>  ca target/pg-ca.crt localhost
./target/live.exe 127.0.0.1 <tls-порт>  mismatch target/pg-ca.crt
./target/live.exe 127.0.0.1 <tls-порт>  system target/pg-ca.crt localhost
./target/live.exe 127.0.0.1 <plain-порт> require
./target/live.exe 127.0.0.1 <plain-порт> prefer target/pg-ca.crt
./target/live.exe 127.0.0.1 <plain-порт> disable
docker stop nova-pg-d8c-tls nova-pg-d8c-plain
```

Лицензия: MIT OR Apache-2.0.
