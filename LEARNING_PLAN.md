# План навчання InfluxDB

## Мета

Навчитися передбачати вартість записів і запитів, проєктувати безпечну схему та діагностувати навантаження до того, як воно перевантажить InfluxDB Cloud 1.

Основна платформа курсу — **InfluxDB Cloud 1 / Enterprise v1 (TSM + InfluxQL)**. Спільні принципи time series вивчаємо так, щоб вони залишилися корисними після можливої міграції на InfluxDB 3. Відмінності v1, v2 і v3 завжди позначаємо явно.

## Вихідний рівень і підхід

Вихідна база: впевнене розуміння MongoDB і реляційних SQL-баз. Тому курс не повторює загальну теорію БД, а порівнює знайомі поняття з InfluxDB:

- таблиця / collection → measurement (але відповідність не повна);
- row / document → point;
- index → індекс measurement + tags;
- partition → time-based shard group (наближена аналогія);
- TTL → retention policy (наближена аналогія);
- materialized aggregate → continuous query + downsampled measurement.

Кожен урок має чотири частини:

1. ментальна модель;
2. невеликий синтетичний приклад;
3. production-антипатерн і його ціна;
4. коротка практична вправа та перевірка розуміння.

Орієнтир: 60–90 хвилин на урок. Практика виконується локально або на синтетичних даних. Production за замовчуванням використовується лише для обмежених read-only спостережень.

## Головна модель вартості

Упродовж курсу оцінюємо кожну операцію через п'ять множників:

> **вартість запиту ≈ часовий діапазон × кількість серій × щільність точок × кількість полів × складність операцій**

Це не точна формула рушія, а дисципліна для попередньої оцінки ризику. Для запису додаємо ще кількість нових series keys, число зачеплених shard groups, розмір batch і частоту flush/retry.

## Навчальний маршрут

### Урок 0. Карта нашої системи і безпечна лабораторія

**Питання:** що саме працює у нас і де проходить межа між навчанням та production?

- підтвердити версію Cloud 1 / Enterprise, доступні API та клієнти;
- намалювати шлях даних: producer → Node.js client/Telegraf → API → database/RP → dashboards/consumers;
- зафіксувати типові measurement, RP, частоту запису, обсяг, основні запити та симптоми інцидентів;
- підготувати локальний InfluxDB v1 або інший безпечний стенд із синтетичними даними;
- створити production safety checklist.

**Результат:** коротка карта системи без секретів і список двох-трьох реальних інцидентів для подальших case studies.

### Урок 1. Що таке time series база і навіщо InfluxDB

**Питання:** чому це не просто SQL-таблиця з колонкою `timestamp`?

- подія/вимірювання, час події та час надходження;
- append-heavy workload, ordered time ranges, агрегація по вікнах;
- compression, retention, downsampling і time-based sharding;
- чим TSDB відрізняється від PostgreSQL/MySQL і MongoDB;
- сильні сторони InfluxDB і випадки, де звичайна SQL-БД краща;
- чому широкий запит за два роки принципово небезпечний без агрегації.

**Практика:** порівняти одну модель telemetry у SQL, MongoDB та line protocol; передбачити, які запити будуть дешевими й дорогими.

**Критерій:** можна своїми словами пояснити, яку роботу виконує TSDB замість застосунку і чому час є частиною фізичної організації даних.

### Урок 2. Логічна та фізична модель InfluxDB v1

**Питання:** чи всі записи однакові і що насправді створює series?

- database, retention policy, measurement, point;
- tag set, field set, timestamp, field types;
- series key і series cardinality;
- індексовані tags проти неіндексованих fields;
- shard і shard group, TSM та TSI на концептуальному рівні;
- дублікати: однакові measurement + tag set + timestamp та злиття field sets.

**Практика:** розкласти 10 line protocol points на measurements, series і shards; вручну порахувати cardinality.

**Критерій:** до запису нової tag-комбінації можна сказати, чи створить вона нову series і як це вплине на пам'ять індексу.

### Урок 3. Схема: tags, fields і вибух cardinality

**Питання:** що індексувати, а що ніколи не класти в tag?

- схема від майбутніх запитів, а не від форми JSON-події;
- UUID, request ID, URL, hash та інші high-cardinality значення;
- залежні й незалежні tags;
- надто багато measurements або field keys;
- оцінка cardinality до релізу;
- безпечні `SHOW ... CARDINALITY` та ризик `EXACT` на великій базі.

**Практика:** переробити небезпечну схему application events та скласти cardinality budget.

**Критерій:** схема пояснюється через конкретні query patterns і має оцінку верхньої межі series.

### Урок 4. Запис: line protocol і write path

**Питання:** як один point і великий batch проходять через систему?

- line protocol, escaping, типи, timestamp precision;
- batching, compression, flush interval, backpressure і retry;
- partial failures, idempotency та duplicate-point semantics;
- type conflicts;
- backfill: чому запис за великий історичний діапазон зачіпає багато shards;
- майбутні timestamps і помилкові timestamps у nanoseconds/seconds;
- `/write` v1 проти `/api/v2/write` compatibility endpoint.

**Практика:** створити безпечний batch writer із контрольованим batch size, retry policy і метриками.

**Критерій:** перед bulk write можна оцінити points/sec, payload size, кількість shards, нових series та поведінку при timeout.

### Урок 5. Node.js client у нашій версійній реальності

**Питання:** які частини v2 JavaScript client справді сумісні з Cloud 1?

- клієнт, wire protocol та database engine — різні рівні;
- v2 write endpoint на InfluxDB 1.8+;
- відображення `bucket` → `database/retention-policy`, ігнорування `org`;
- auth і конфігурація без hardcoded secrets;
- write options: batch size, flush interval, timeout, retries і buffer limits;
- InfluxQL `/query` окремо від v2/Flux query examples;
- migration-friendly adapter навколо клієнта.

**Практика:** розібрати поточний redacted Node.js writer і позначити всі параметри, що впливають на навантаження.

**Критерій:** жоден приклад із документації v2 не переноситься на Cloud 1 без перевірки endpoint і semantics.

### Урок 6. InfluxQL: як читати тільки потрібні дані

**Питання:** як сформувати селективний і обмежений запит?

- `SELECT`, `FROM`, `WHERE time`, tag filters і field filters;
- aggregates, selectors, transformations;
- `GROUP BY time()` і tags, `fill()`;
- `LIMIT` проти `SLIMIT`;
- regex та інші операції, що розширюють scan;
- chunked responses для великих результатів;
- чому `LIMIT` результату не завжди обмежує всю виконану роботу.

**Практика:** побудувати запит поступово — від однієї години й однієї series до необхідного production range.

**Критерій:** кожен запит має явний time bound, селективний series filter та виправдану resolution.

### Урок 7. Вартість запиту і захист кластера

**Питання:** чому запит за два роки може «покласти» кластер?

- кількість shard groups і TSM blocks, яких торкається запит;
- raw points проти server-side aggregation;
- series fan-out, buckets і обсяг відповіді;
- `EXPLAIN` / `EXPLAIN ANALYZE` у безпечному середовищі;
- `SHOW QUERIES`, slow query logging і `KILL QUERY`;
- `query-timeout`, `max-select-point`, `max-select-series`, `max-select-buckets`;
- concurrency limits і application-side guardrails;
- downsample-first запити для довгих періодів.

**Практика:** взяти довгий запит з [карти production](notes/system-map.md#запити-дашборда-для-oil-quality). Оцінити вартість читання `oil_quality` і `oil_quality_daily`, звірити сенс метрик і побудувати безпечні варіанти для одного та кількох сенсорів.

**Критерій:** ризик оцінюється до виконання, а не після спостереження CPU/OOM.

### Урок 8. Retention policies, continuous queries і downsampling

**Питання:** як зберігати довгу історію, не запитуючи роки raw data?

- duration, replication factor і shard group duration;
- lifecycle raw → hourly → daily aggregates;
- continuous queries, resample та запізнілі дані;
- вибір aggregation і точності;
- межі retention deletion та shard groups;
- перевірка коректності downsampled даних.

**Практика:** спроєктувати RP і CQ для короткої raw retention та дворічної аналітики.

**Критерій:** dashboard ніколи не читає більше деталізації, ніж може показати.

### Урок 9. Спостережуваність і incident response

**Питання:** як відрізнити expensive query, write flood, cardinality explosion та shard problem?

- `_internal`, `SHOW STATS`, `SHOW DIAGNOSTICS` і Cloud 1 monitoring;
- query latency/concurrency, write throughput/errors, memory, CPU, disk, series growth;
- симптом → гіпотеза → перевірка → mitigation;
- runbook для runaway query;
- runbook для write storm/backfill;
- які дані зібрати для InfluxData Support.

**Практика:** tabletop-розбір двох інцидентів із таймлайном і доказами.

**Критерій:** перша реакція зменшує blast radius і зберігає діагностичні дані.

### Урок 10. Рев'ю робочого навантаження

**Питання:** де наші конкретні ризики?

- інвентар writer-ів, measurements, RPs, cardinality та top queries;
- аналіз redacted line protocol і Node.js config;
- оцінка [кейсу з тегом компанії](notes/system-map.md#кейс-для-уроку-з-тегом-компанії) для фільтрації, історії передачі сенсорів і cardinality;
- risk matrix: probability × cluster impact;
- quick wins, guardrails і довші schema changes;
- production checklist для query/write changes.

**Результат:** пріоритизований backlog технічних змін із доказами.

### Урок 11. Що зміниться при міграції на InfluxDB 3

**Питання:** які знання переносяться, а які треба перевчити?

- незмінні принципи: time bounds, resolution, batching, observability, lifecycle;
- storage/query architecture v3 на високому рівні;
- SQL та InfluxQL, v1/v2 compatibility APIs;
- відмінності cardinality, schema і operational limits;
- compatibility ≠ identical performance;
- план експериментів і критерії міграції без переписування всього одразу.

**Результат:** comparison matrix Cloud 1 → Cloud 3 і список migration spikes.

## Рекомендований порядок перших занять

Починаємо не з API, а з причинно-наслідкової моделі:

1. Урок 1 — чому існують TSDB.
2. Урок 2 — point, series, shard і RP.
3. Урок 7 — розбір дворічного запиту на концептуальному рівні.
4. Урок 3 — cardinality та схема.
5. Урок 4 — write path і bulk writes.
6. Урок 5 — ваш Node.js client.
7. Уроки 6, 8, 9 — запити, downsampling та operations.

Урок 0 збираємо поступово, не затримуючи теорію: достатньо redacted прикладів, без production credentials.

## Артефакти, які створимо

- `lessons/01-what-is-a-tsdb.md` та наступні конспекти;
- `labs/` із синтетичними datasets і вправами;
- `checklists/query-safety.md`;
- `checklists/write-safety.md`;
- `runbooks/runaway-query.md`;
- `runbooks/write-storm.md`;
- `notes/system-map.md` без секретів;
- `progress.md` із результатами коротких перевірок.

## Вхідні дані для адаптації курсу

У міру навчання корисно додати лише redacted інформацію:

- приблизна версія/план Cloud 1 та регіон;
- Node.js package name і version;
- один приклад line protocol;
- один типовий і один небезпечний InfluxQL query;
- орієнтовні write rate, batch size і retention;
- measurement/tag/field names без чутливих значень;
- що означає «кластер ліг»: timeout, CPU, OOM, рестарт, write rejection чи інший симптом.

## Основні джерела

- [Огляд платформи і версій](https://docs.influxdata.com/platform/)
- [Enterprise v1 — правильна документація для Cloud 1](https://docs.influxdata.com/enterprise_influxdb/v1/)
- [Запис даних у Enterprise v1](https://docs.influxdata.com/enterprise_influxdb/v1/guides/write_data/)
- [Запити до Enterprise v1](https://docs.influxdata.com/enterprise_influxdb/v1/guides/query_data/)
- [Schema design і data layout](https://docs.influxdata.com/enterprise_influxdb/v1/concepts/schema_and_data_layout/)
- [InfluxQL](https://docs.influxdata.com/enterprise_influxdb/v1/query_language/)
- [InfluxQL query management](https://docs.influxdata.com/enterprise_influxdb/v1/troubleshooting/query_management/influxql_query_management/)
- [Node.js v2 client](https://docs.influxdata.com/influxdb/v2/api-guide/client-libraries/nodejs/)
- [InfluxDB documentation MCP](https://docs.influxdata.com/platform/mcp/server/)

## Прогрес

| Урок | Статус | Артефакт / нотатка |
|---|---|---|
| 0. Карта системи | Не розпочато | |
| 1. Що таке TSDB | Пройдено 2026-11-18 | `lessons/01-what-is-a-tsdb.md`, результати в `progress.md` |
| 2. Модель InfluxDB v1 | Матеріал підготовлено | `lessons/02-influxdb-v1-data-model.md` |
| 3. Schema і cardinality | Пройдено 2026-11-24 | [Результати практики](progress.md#урок-3-теги-поля-і-cardinality--пройдено-2026-11-24) |
| 4. Write path | Не розпочато | |
| 5. Node.js client | Не розпочато | |
| 6. InfluxQL | Не розпочато | |
| 7. Query cost | Не розпочато | |
| 8. Retention і downsampling | Не розпочато | |
| 9. Operations | Не розпочато | |
| 10. Workload review | Не розпочато | |
| 11. Cloud 3 migration | Не розпочато | |
