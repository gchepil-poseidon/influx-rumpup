# Урок 2. Логічна та фізична модель InfluxDB v1

> Урок завершено 2026-11-24. Результати усної вправи записані в [`progress.md`](../progress.md#урок-2-логічна-та-фізична-модель-influxdb-v1--пройдено-2026-11-24).

## Навіщо цей урок

Більшість дорогих помилок в InfluxDB починається не з синтаксису query, а з неправильної ментальної моделі даних.

Після уроку ви повинні вміти:

- прочитати line protocol без клієнтської бібліотеки;
- відрізнити database, retention policy, measurement, tag, field, series і point;
- пояснити, що індексується, а що зберігається як value;
- передбачити, чи створить write нову indexed series;
- пояснити, як timestamp спрямовує point у shard group;
- описати поведінку duplicate point;
- вручну оцінити cardinality простої схеми;
- розрізняти indexed series identity, TSM storage key та point identity.

Орієнтовний час: 90 хвилин.

## 1. Один приклад, який будемо розкладати

```text
weather,region=us-east,site=nyc,sensor=s-17 temperature=21.5,humidity=47i 1794945600000000000
```

Він описує один point із двома field values:

| Частина | Значення |
|---|---|
| Measurement | `weather` |
| Tags | `region=us-east`, `site=nyc`, `sensor=s-17` |
| Fields | `temperature=21.5`, `humidity=47i` |
| Timestamp | `1794945600000000000` ns |

Line protocol має загальну форму:

```text
<measurement>[,<tag-key>=<tag-value>...] <field-key>=<field-value>[,...] [timestamp]
```

Перший неекранований пробіл відділяє measurement/tags від fields. Другий — fields від timestamp. Тому line protocol є whitespace-sensitive.

## 2. Ієрархія InfluxDB Cloud 1 / v1

Практично корисна ієрархія:

```text
InfluxDB cluster
└── database: telemetry
    └── retention policy: raw_30d
        └── measurement: weather
            ├── indexed series identities: measurement + tag set
            ├── field values ordered by time
            └── points routed into time-based shard groups
```

Це не буквальна схема всіх внутрішніх файлів. Це карта, яка допомагає зрозуміти query і write cost.

## 3. Database

Database — верхній логічний контейнер v1 для measurements, users/permissions, retention policies та continuous queries.

SQL-аналогія: database або schema, залежно від системи. Аналогія неповна, але корисна.

Приклад:

```sql
CREATE DATABASE "telemetry"
```

Cloud 1 використовує database/retention-policy модель. Це важлива відмінність від InfluxDB v2, де основним контейнером є bucket.

## 4. Retention policy

Retention policy (RP) визначає:

- як довго зберігати дані;
- replication factor у кластері;
- shard group duration;
- чи є RP default для database.

Приклад концептуальної схеми:

```text
telemetry.autogen    → raw data, default RP
telemetry.raw_30d    → raw data за 30 днів
telemetry.hourly_2y  → hourly aggregates за 2 роки
telemetry.daily_inf  → daily aggregates без обмеження строку
```

Повне ім'я measurement в InfluxQL може мати форму:

```text
"database"."retention_policy"."measurement"
```

Наприклад:

```sql
SELECT mean("temperature")
FROM "telemetry"."raw_30d"."weather"
WHERE time >= now() - 1h
GROUP BY time(5m), "site"
```

RP — не просто TTL label. Вона також впливає на time-based physical layout через shard group duration.

## 5. Measurement

Measurement — логічна група points одного типу.

Наближена SQL-аналогія — table. Але measurement:

- не має класичної row schema в тому самому сенсі;
- може містити points із різними field sets;
- використовує tags як індексовані dimensions;
- завжди працює разом із timestamp.

Приклади добрих measurement names:

```text
cpu
http_request
weather
device_health
```

Небезпечний pattern — кодувати змінні дані в measurement name:

```text
cpu_api_01
cpu_api_02
cpu_api_03
```

Зазвичай краще:

```text
cpu,host=api-01 usage=31.2
cpu,host=api-02 usage=54.0
cpu,host=api-03 usage=27.8
```

Тут `host` є dimension, за якою можна фільтрувати й групувати.

## 6. Tags

Tag — індексована string key/value dimension.

У прикладі:

```text
region=us-east
site=nyc
sensor=s-17
```

Tags природно використовувати для значень, за якими часто виконуємо:

- filtering;
- grouping;
- series selection.

Наприклад:

```sql
WHERE "region" = 'us-east' AND "site" = 'nyc'
GROUP BY "sensor"
```

### Ціна tag

Tags пришвидшують пошук series, але кожна нова унікальна комбінація tag values може створити нову indexed series identity.

Небезпечні tag values:

- request ID;
- trace ID;
- random UUID;
- email;
- URL із dynamic parameters;
- hash;
- unbounded user-generated string.

Індекс дає швидкість лише тоді, коли його розмір залишається керованим.

## 7. Fields

Field зберігає саме виміряне значення. Fields не використовуються як indexed series dimensions.

У прикладі:

```text
temperature=21.5
humidity=47i
```

Field values можуть бути числовими, boolean або string відповідно до підтримуваних v1 типів.

Важлива line protocol деталь:

```text
temperature=21.5  # float
humidity=47i      # integer через суфікс i
online=true       # boolean
status="ready"   # string у подвійних лапках
```

Без `i` значення `47` буде прочитане як float, а не integer.

### Чому не все робити field

Field filter може вимагати читання values, бо field не є індексованою dimension. Якщо значення регулярно визначає, **які series треба знайти**, воно може бути tag.

### Чому не все робити tag

Кожне унікальне tag combination розширює індекс. Numerical measurements, UUID та інші high-cardinality values зазвичай належать у fields.

## 8. Tag чи field: практичне правило

Поставте два запитання:

1. Чи використовується значення для регулярного series selection або `GROUP BY`?
2. Чи має значення bounded/reasonable cardinality?

| Властивість | Типовий вибір | Чому |
|---|---|---|
| `region` | Tag | Частий filter/group, мало значень |
| `host` | Tag | Частий series selector |
| `status_code` | Залежить | Tag для bounded HTTP codes; field, якщо не фільтруємо |
| `temperature` | Field | Numerical measurement |
| `request_id` | Field або інше сховище | Майже унікальне значення |
| `error_message` | Field/лог-система | Unbounded text |
| `customer_id` | Залежить | Ризик cardinality; вирішується query patterns і budget |

Це не механічне правило. Схему визначають запити плюс cardinality budget.

## 9. Point

Point — один time-stamped запис у measurement.

У line protocol одна строка представляє один point, але point може містити кілька fields:

```text
weather,sensor=s-17 temperature=21.5,humidity=47i,battery=92i 1794945600000000000
```

Це один point із трьома field values, а не три незалежні HTTP writes.

Points одного measurement не зобов'язані мати ідентичні field sets:

```text
weather,sensor=s-17 temperature=21.5 1794945600000000000
weather,sensor=s-17 humidity=47i 1794945660000000000
```

Проте дуже sparse і хаотична схема може ускладнити queries та operations. «Schemaless» не означає «designless».

## 10. Series: важливе термінологічне уточнення

У документації різних поколінь InfluxData слово `series` може підкреслювати різні рівні. Щоб не змішувати їх, у цьому курсі ми завжди називатимемо рівень явно.

### 10.1. Indexed series identity

Для schema/cardinality reasoning основна індексована ідентичність утворюється з:

```text
measurement + tag set
```

Наприклад:

```text
weather,region=us-east,site=nyc,sensor=s-17
```

Зміна field value не створює нову indexed tag combination. Зміна tag value — створює.

### 10.2. TSM storage key

Storage engine організовує field values за ключем, який концептуально включає:

```text
measurement + tag set + field key
```

Тому `temperature` і `humidity` для однакових tags є окремими time-ordered value streams усередині storage engine.

### 10.3. Point identity для duplicate-write semantics

Офіційна v1 документація визначає duplicate point за:

```text
measurement + tag set + timestamp
```

Якщо ці три частини однакові, field sets об'єднуються, а конфлікт однакового field key у v1 вирішується новим значенням.

### Практичний висновок

Коли хтось каже «у нас мільйон series», треба уточнити:

- результат якої команди або метрики;
- indexed tag combinations чи field-level storage series;
- для якої database/RP/measurement;
- estimate чи exact count.

Точна назва метрики важливіша за суперечку про короткий термін.

## 11. Duplicate points

Початковий point:

```text
weather,sensor=s-17 temperature=21.5,humidity=47i 1794945600000000000
```

Новий write із тією самою measurement, tag set і timestamp:

```text
weather,sensor=s-17 temperature=22.0,battery=92i 1794945600000000000
```

У v1 результатом буде об'єднаний field set:

```text
temperature=22.0,humidity=47i,battery=92i
```

`temperature` оновлено, `humidity` залишено, `battery` додано.

### Чому це важливо

- retry може бути idempotent лише якщо identity і values продумані;
- coarse timestamp precision збільшує ймовірність випадкових collisions;
- timestamp — частина data model, а не просто metadata;
- overwrite semantics v1 не слід автоматично переносити на v3 без перевірки.

## 12. Timestamp і precision

Timestamp у line protocol є Unix time. Якщо precision не вказана, API v1 зазвичай трактує integer timestamp як nanoseconds.

Одна й та сама секунда може бути записана як:

```text
1794945600          # seconds
1794945600000       # milliseconds
1794945600000000    # microseconds
1794945600000000000 # nanoseconds
```

API має знати precision. Помилка тут може відправити дані на десятиліття в минуле або майбутнє, створюючи несподівані shards.

Використовуйте найгрубішу precision, яка зберігає потрібну точність. Це також може покращувати compression, але треба враховувати duplicate collisions.

## 13. Shard і shard group

Retention policy ділить time axis на shard groups. Timestamp point визначає, у який shard group він потрапить.

Спрощений приклад із shard group duration `1d`:

```text
2026-11-16 00:00 ─ 2026-11-17 00:00  → shard group A
2026-11-17 00:00 ─ 2026-11-18 00:00  → shard group B
2026-11-18 00:00 ─ 2026-11-19 00:00  → shard group C
```

### Чому це важливо для query

Query за одну годину торкається малого time range. Query за два роки потенційно торкається сотень shard groups.

### Чому це важливо для write

Live writes переважно йдуть у поточний shard group. Historical backfill за два роки розподіляє writes між багатьма старими shard groups, створюючи інший cache/compaction/I/O workload.

### Retention deletion

InfluxDB видаляє прострочені дані через shard groups. Тому фактична межа видалення пов'язана із завершенням shard group, а не з індивідуальним `DELETE` кожного point.

## 14. Cardinality

Cardinality — кількість унікальних series identities відповідно до конкретної метрики/команди.

Для попередньої оцінки indexed combinations корисна верхня межа:

```text
measurements × незалежні tag-value combinations
```

Приклад:

- 1 measurement `weather`;
- 1 000 sensors;
- 3 regions;
- 2 modes.

Наївна верхня межа:

```text
1 × 1 000 × 3 × 2 = 6 000
```

Але якщо кожен sensor фізично належить лише одному region, `sensor` і `region` залежні. Реальна кількість може бути:

```text
1 000 sensors × 2 modes = 2 000 combinations
```

Не всі tag cardinalities треба сліпо перемножувати. Потрібно знати залежності.

### Cardinality explosion

Додамо `request_id` як tag, де щодня 10 мільйонів унікальних значень:

```text
weather,sensor=s-17,request_id=550e8400-e29b-41d4-a716-446655440000 ...
```

Тоді індексовані combinations ростуть майже разом із кількістю requests. Це руйнує головну перевагу series index і може збільшувати memory cost.

## 15. Як query використовує модель

```sql
SELECT mean("temperature")
FROM "telemetry"."raw_30d"."weather"
WHERE time >= now() - 24h
  AND "region" = 'us-east'
  AND "site" = 'nyc'
GROUP BY time(5m), "sensor"
```

Концептуально InfluxDB:

1. визначає database/RP/measurement;
2. через tag index знаходить matching series identities;
3. через time range визначає потрібні shard groups;
4. читає `temperature` field values із TSM;
5. формує 5-minute windows;
6. повертає result series, згруповані за `sensor`.

Це пояснює, чому:

- tag filter може різко зменшити series fan-out;
- field filter не замінює tag index;
- time range визначає кількість storage partitions;
- `GROUP BY time()` визначає bucket count;
- кожен додатковий field збільшує value streams, які треба читати.

## 16. Production-антипатерни цього уроку

### Антипатерн 1: UUID як tag

Кожен write створює нову tag combination. Це майже event index, а не bounded time series.

### Антипатерн 2: numerical value як tag

```text
weather,temperature=21.5317 value=1
```

Значення має високу варіативність і роздуває series index. `temperature` повинна бути field.

### Антипатерн 3: неправильна timestamp precision

Seconds передано як nanoseconds або навпаки. Дані потрапляють у неочікуваний час і shards.

### Антипатерн 4: measurement per entity

```text
temperature_sensor_000001
temperature_sensor_000002
```

Ускладнює queries і schema management. Зазвичай entity ID має бути tag або, при надмірній cardinality, потребує перегляду архітектури.

### Антипатерн 5: tag і field з однаковим key

Результати стають неоднозначними й потребують `::tag` / `::field` qualification. Краще уникати однакових назв.

## 17. Практична вправа A: розбір points

```text
cpu,host=api-01,region=us-east usage=31.2,temp=62.1 1794945600000000000
cpu,host=api-01,region=us-east usage=38.4,temp=63.0 1794945660000000000
cpu,host=api-02,region=us-east usage=54.0,temp=67.2 1794945600000000000
cpu,host=api-01,region=us-east usage=40.0 1794945660000000000
```

Дайте відповіді:

1. Скільки line protocol points записано?
2. Скільки унікальних indexed tag combinations?
3. Які field keys існують?
4. Що станеться з другим point після четвертого write?
5. Які value streams концептуально існують у TSM?

## 18. Практична вправа B: оцінка cardinality

Схема:

```text
http_request,service=api,region=us-east,status=200,method=GET,request_id=... duration_ms=42i
```

Відомо:

- 20 services;
- 4 regions;
- 6 status codes;
- 5 methods;
- 50 мільйонів request IDs на день.

### Завдання

1. Оцініть верхню межу без `request_id`.
2. Поясніть, що змінює `request_id` як tag.
3. Запропонуйте безпечнішу схему.
4. Визначте, де шукати повний request-level event, якщо він потрібен для debugging.

## 19. Контрольні питання

1. Яка різниця між tag і field?
2. Що визначає RP крім строку зберігання?
3. Чи створює нове numerical field value нову indexed tag combination?
4. Які частини визначають duplicate point у v1?
5. Що відбудеться з field sets при duplicate write?
6. Чому historical backfill відрізняється від live write?
7. Чому `sensor_id` і `region` не завжди треба перемножувати при оцінці cardinality?
8. Назвіть три різні identity/key layers, які ми розрізняємо.
9. Чому integer у line protocol має суфікс `i`?
10. Які три кроки зменшують query work ще до aggregation?

## 20. Відповіді для самоперевірки

### Вправа A

1. Чотири writes/line protocol lines.
2. Дві indexed tag combinations: `api-01/us-east` і `api-02/us-east` у measurement `cpu`.
3. `usage`, `temp`.
4. Measurement, tag set і timestamp збігаються. `usage=38.4` буде замінено на `usage=40.0`; `temp=63.0` залишиться через union field sets.
5. Для кожної tag combination окремі time-ordered streams `usage` і `temp`, якщо field присутній.

### Вправа B

1. Наївна верхня межа: `20 × 4 × 6 × 5 = 2 400` combinations. Реальна може бути нижчою через залежності.
2. `request_id` робить майже кожен request новою indexed combination: десятки мільйонів щодня.
3. Наприклад: `http_request,service=api,region=us-east,status=200,method=GET duration_ms=42i,request_id="..."`. Але навіть як field повний request-level workload може бути невідповідним для InfluxDB залежно від retention/query needs.
4. У log/event system, object storage або іншому сховищі, оптимізованому для high-cardinality event search. В InfluxDB можна залишити bounded aggregates.

### Контрольні питання

1. Tag — індексована string dimension для series selection/grouping; field — виміряне value, не індексована series dimension.
2. Replication factor, shard group duration і default status у database.
3. Ні. Зміна field value не змінює measurement+tag combination.
4. Measurement + tag set + timestamp.
5. Field sets об'єднуються; у v1 конфлікт однакового field key виграє новий write.
6. Backfill торкається багатьох старих shard groups, caches і compactions; live write переважно працює з поточним shard group.
7. Бо вони можуть бути залежними: один sensor має лише один region.
8. Indexed identity: measurement+tags; TSM field stream key: measurement+tags+field key; duplicate point identity: measurement+tags+timestamp.
9. Без `i` числове значення читається як float.
10. Звузити database/RP/measurement, вибрати series через tags і обмежити time range/shards.

## 21. Checklist для schema review

- [ ] Measurement описує стабільний тип даних, а не конкретну entity.
- [ ] Tags відповідають регулярним filters/groups.
- [ ] У кожного tag є bounded або обґрунтований cardinality budget.
- [ ] UUID, request IDs і random values не потрапили в tags без свідомого рішення.
- [ ] Numerical measurements зберігаються як fields.
- [ ] Tag keys і field keys не дублюють назви.
- [ ] Timestamp precision явна й протестована.
- [ ] Retention policy відповідає query horizon.
- [ ] Historical backfill оцінено за кількістю shard groups.
- [ ] Duplicate/retry semantics відомі writer-у.

## 22. Джерела

- [InfluxDB Enterprise v1 glossary](https://docs.influxdata.com/enterprise_influxdb/v1/concepts/glossary/)
- [Schema design and data layout](https://docs.influxdata.com/enterprise_influxdb/v1/concepts/schema_and_data_layout/)
- [InfluxDB storage engine](https://docs.influxdata.com/enterprise_influxdb/v1/concepts/storage_engine/)
- [Line protocol tutorial](https://docs.influxdata.com/enterprise_influxdb/v1/write_protocols/line_protocol_tutorial/)
- [Enterprise v1 FAQ: duplicate points і historical data](https://docs.influxdata.com/enterprise_influxdb/v1/troubleshooting/frequently-asked-questions/)
- [Explore schema with InfluxQL](https://docs.influxdata.com/enterprise_influxdb/v1/query_language/explore-schema/)
- [Retention policy management](https://docs.influxdata.com/enterprise_influxdb/v1/query_language/manage-database/)

## 23. Що далі

Урок 3 використає цю модель для schema design і cardinality budget. Ми візьмемо кілька реальних query patterns, спроєктуємо tags/fields, порахуємо верхню межу series та перевіримо, що схема не створює unbounded index growth.
