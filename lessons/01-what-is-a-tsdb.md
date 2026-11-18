# Урок 1. Що таке time series база і навіщо InfluxDB

## Навіщо цей урок

Мета — побудувати модель, яка дозволяє ще до запуску запиту поставити правильне запитання: **скільки роботи доведеться виконати базі?**

Після уроку ви повинні вміти:

- пояснити, чому InfluxDB — не просто SQL-база з колонкою `timestamp`;
- назвати workload, для якого TSDB оптимізована;
- пояснити сильні сторони та обмеження InfluxDB порівняно з SQL і MongoDB;
- приблизно оцінити ризик запиту за часовим діапазоном, кількістю series і resolution;
- зрозуміти, чому запит за два роки raw data може перевантажити кластер;
- визначити, коли InfluxDB узагалі не є найкращим сховищем.

Орієнтовний час: 75–90 хвилин.

## 1. Почнемо не з InfluxDB, а з даних

Time series — це послідовність значень, прив'язаних до часу. Типова точка відповідає на чотири запитання:

1. **Що вимірюємо?** Наприклад, CPU, температуру або latency.
2. **Де або для кого?** Host, sensor, customer, region.
3. **Коли?** Timestamp події.
4. **Яке значення?** Наприклад, `73.2`, `200`, `true`.

Приклад telemetry-події у JSON:

```json
{
  "metric": "cpu_usage",
  "host": "api-17",
  "region": "us-east",
  "timestamp": "2026-11-17T20:00:00Z",
  "value": 73.2
}
```

Ті самі дані можна покласти в SQL, MongoDB або InfluxDB. Різниця не в тому, **чи здатна** база зберегти такий запис. Різниця в тому, **який повторюваний workload вона робить дешевим**.

## 2. Типовий workload time series

Для telemetry, metrics та sensor data характерні такі властивості:

- записи надходять постійним потоком;
- нові дані переважно додаються, а не редагуються;
- більшість запитів має часовий діапазон;
- свіжі дані читають детально, старі — агреговано;
- запити часто групують значення у часові вікна: 1 хвилина, 1 година, 1 день;
- старі raw data можна автоматично видаляти;
- однакова форма даних повторюється мільйони або мільярди разів;
- важливі не окремі записи, а тренди, rate, percentiles, min/max/mean.

TSDB проєктує storage engine, індекси, compression і lifecycle навколо саме цих припущень.

## 3. Чому не просто SQL із `timestamp`

У SQL можна створити таблицю:

```sql
CREATE TABLE cpu_usage (
  host       text,
  region     text,
  time       timestamptz,
  value      double precision
);
```

І з правильними індексами, partitioning та extensions така система може працювати дуже добре. Отже, теза не така:

> «SQL не вміє time series».

Правильна теза:

> «TSDB робить time-series припущення частиною своєї базової моделі та операційного API».

InfluxDB v1 одразу дає:

- timestamp як обов'язкову вісь даних;
- групування зберігання за time ranges;
- time-oriented query language;
- retention policies;
- continuous queries для downsampling;
- line protocol для великого потоку вимірювань;
- compression, розраховану на послідовності значень у часі.

У SQL ці можливості часто треба окремо проєктувати: partitioning, TTL jobs, materialized aggregates, index strategy та maintenance.

## 4. Порівняння з MongoDB і SQL

| Питання | SQL | MongoDB | InfluxDB Cloud 1 / v1 |
|---|---|---|---|
| Основна одиниця | Row | Document | Point |
| Головна організація | Table + indexes | Collection + indexes | Time + measurement + series |
| Типові зміни | CRUD | CRUD, flexible documents | Переважно append/write |
| Зв'язки | Сильні joins і constraints | Embedding/references | Не для складних joins |
| Індексування | Явно вибрані columns | Явно вибрані fields | Measurement і tags індексуються концептуально; fields — ні |
| Lifecycle | Partition/TTL/jobs | TTL index | Retention policy |
| Старі aggregates | Materialized view/job | Aggregation pipeline/job | Continuous query + окрема RP |
| Основний query shape | Різноманітний | Document/aggregation | Time range + series filter + aggregation |
| Типова небезпека | Bad join/full scan/lock | Full scan/large pipeline | Надто багато series × shards × raw points |

Це навмисно спрощене порівняння. Воно показує оптимізаційний центр кожної системи, а не повний список можливостей.

## 5. Ментальна модель InfluxDB

Уявіть не таблицю, а велику кількість окремих часових стрічок:

```text
host=api-01, metric=cpu:    20:00 31% ─ 20:01 38% ─ 20:02 35% ─ ...
host=api-02, metric=cpu:    20:00 54% ─ 20:01 61% ─ 20:02 59% ─ ...
host=api-01, metric=memory: 20:00 72% ─ 20:01 72% ─ 20:02 73% ─ ...
```

Кожна така стрічка — series у практичній ментальній моделі. Дані всередині неї природно впорядковані за часом.

Запит є дешевим, коли база може швидко відповісти:

1. які саме series потрібні;
2. який відрізок часу потрібен;
3. з якою resolution потрібна відповідь.

Запит стає небезпечним, коли відповідь хоча б на одне запитання звучить як «майже все».

## 6. Три виміри небезпечного запиту

### 6.1. Широкий часовий діапазон

Чим довший діапазон, тим більше shard groups, TSM blocks і точок може знадобитися прочитати.

### 6.2. Широкий series fan-out

Запит без селективного tag filter може звернутися до тисяч або мільйонів series.

### 6.3. Надто висока resolution

Для графіка на 1 000 пікселів за два роки не потрібно повертати кожну секунду. Raw resolution може означати мільярди значень, які клієнт фізично не здатен відобразити.

Наша робоча евристика:

> **вартість ≈ часовий діапазон × series × points per unit of time × fields × складність операцій**

Це не формула query planner. Це спосіб не запускати запит наосліп.

## 7. Числовий приклад дворічного запиту

Припустимо:

- 10 000 series;
- одна точка кожні 10 секунд;
- запит за 2 роки;
- без попереднього downsampling.

Для однієї series:

```text
2 × 365 × 24 × 60 × 6 = 6 307 200 points
```

Для 10 000 series:

```text
6 307 200 × 10 000 = 63 072 000 000 points
```

Це понад 63 мільярди точок до врахування кількох fields, декодування, агрегації, мережі та пам'яті клієнта.

Навіть якщо query повертає лише середнє значення, база може спочатку бути змушена прочитати величезний набір raw data, якщо немає відповідного downsampled measurement.

### Що змінює `LIMIT`?

`LIMIT 1000` обмежує кількість повернутих points на series, але не слід автоматично вважати, що він завжди обмежив усю роботу storage engine. Особливо коли query спочатку виконує aggregate, grouping, sorting або охоплює багато series.

### Безпечніші варіанти

- зменшити time range;
- вибрати конкретні tags/series;
- використати `GROUP BY time()` з resolution, потрібною графіку;
- читати hourly/daily downsampled measurement;
- виконувати довгий export контрольованими time chunks;
- встановити application-side і server-side query guardrails.

## 8. Чому InfluxDB може відчуватися «крихкішою» за SQL

Причина не обов'язково в тому, що InfluxDB «гірша». Частіше відрізняється профіль помилки.

У типовій SQL-системі розробник уже звик думати про:

- indexes;
- query plan;
- joins;
- locks;
- sequential scans;
- pagination.

В InfluxDB нові небезпечні змінні менш очевидні:

- series cardinality;
- tag/field choice;
- number of shard groups;
- time-window count;
- raw resolution;
- continuous-query/downsampling design;
- historical backfill across many shards.

SQL-досвід корисний, але `EXPLAIN`-інтуїцію потрібно доповнити time-series інтуїцією.

## 9. Сильні сторони InfluxDB

InfluxDB добре підходить, коли потрібно:

- постійно приймати time-stamped measurements;
- швидко читати останні значення або обмежені time ranges;
- агрегувати дані у часові вікна;
- зберігати багато повторюваних numerical values;
- автоматично видаляти старі дані;
- підтримувати окремі raw і downsampled рівні;
- будувати monitoring, IoT, telemetry та operational dashboards.

## 10. Коли InfluxDB може бути неправильним вибором

Слід зупинитися й подумати про інше сховище, якщо workload потребує:

- складних joins між багатьма сутностями;
- частих updates/deletes довільних старих записів;
- transactional business workflows;
- uniqueness/foreign-key constraints;
- пошуку довільного тексту;
- document-centric CRUD;
- запитів, у яких час не є основною віссю;
- висококардинальних довільних event attributes, за якими треба робити ad hoc analytics.

Часто правильна архітектура використовує кілька сховищ: SQL для business entities, object storage для raw archives та InfluxDB для operational time series.

## 11. Production-антипатерни цього уроку

### Антипатерн 1: «Візьмемо все, а на клієнті відфільтруємо»

Проблема: база, мережа і Node.js process оплачують усі непотрібні дані.

### Антипатерн 2: «Додамо `LIMIT`, отже query безпечна»

Проблема: limit результату не завжди дорівнює limit роботи.

### Антипатерн 3: «Dashboard за два роки має читати raw seconds»

Проблема: екран не може показати таку resolution; потрібен downsampling.

### Антипатерн 4: «TSDB сама автоматично впорається з будь-яким часом»

Проблема: time-oriented storage оптимізує правильний workload, але не скасовує фізичну вартість читання та запису.

## 12. Практична вправа

Є система з 2 000 devices. Кожен device записує:

- `temperature` раз на 5 секунд;
- `battery` раз на хвилину;
- `firmware_version` при зміні;
- `error_message` при помилці.

Потрібні запити:

1. остання температура одного device;
2. середня температура всіх devices за регіоном за 24 години;
3. денний temperature trend за два роки;
4. пошук усіх error messages, що містять певний текст;
5. поточний firmware кожного device.

### Завдання

Для кожного запиту дайте відповіді:

- чи є InfluxDB природним основним сховищем;
- який time range;
- яка потрібна resolution;
- скільки приблизно series може бути залучено;
- чи потрібен raw або downsampled dataset;
- яке інше сховище може знадобитися.

Не проєктуйте tags і fields детально — це тема наступних уроків.

## 13. Контрольні питання

1. Чому наявність `timestamp` у SQL-таблиці ще не робить її TSDB?
2. Які три параметри роблять time-series query широкою?
3. Скільки points містить одна series із частотою 1 point/10s за один день?
4. Чому дворічний графік не повинен за замовчуванням читати raw seconds?
5. Чи гарантує `LIMIT 1000`, що база прочитає не більше 1 000 points?
6. Назвіть два workload, для яких SQL або MongoDB природніші за InfluxDB.
7. Яке головне питання треба поставити до виконання query?

## 14. Відповіді для самоперевірки

1. Бо TSDB — це не формат row, а набір storage/query/lifecycle оптимізацій навколо append-heavy даних, time ranges, windows, retention і downsampling.
2. Time range, кількість залучених series та resolution/кількість raw points. Додатково впливають fields і операції.
3. `24 × 60 × 6 = 8 640` points.
4. Тому що дисплей не покаже мільйони точок, а читання й агрегація raw data витратять CPU, пам'ять і I/O. Потрібні більші time windows або попередній downsampling.
5. Ні. Він обмежує результат відповідно до semantics query, але aggregation та інші операції можуть вимагати значно більше читання.
6. Наприклад, transactional orders із joins/constraints — SQL; document CRUD або flexible business documents — MongoDB.
7. «Скільки series, shard/time ranges і raw values доведеться обробити, щоб отримати цю відповідь?»

## 15. Мінімальний checklist перед широким query

- [ ] Є явний `WHERE time ...`.
- [ ] Time range мінімальний для задачі.
- [ ] Є селективний tag/series filter.
- [ ] Resolution відповідає результату, який реально буде показано або використано.
- [ ] Для довгої історії перевірено наявність downsampled measurement.
- [ ] Оцінено приблизну кількість points і series.
- [ ] Query спершу перевірена на меншому діапазоні.
- [ ] Відомо, як побачити та зупинити runaway query.

## 16. Джерела

- [InfluxData platform: порівняння поколінь](https://docs.influxdata.com/platform/)
- [InfluxDB Enterprise v1 concepts](https://docs.influxdata.com/enterprise_influxdb/v1/concepts/)
- [Schema design and data layout](https://docs.influxdata.com/enterprise_influxdb/v1/concepts/schema_and_data_layout/)
- [InfluxQL query management](https://docs.influxdata.com/enterprise_influxdb/v1/troubleshooting/query_management/influxql_query_management/)
- [InfluxDB Enterprise v1 query guide](https://docs.influxdata.com/enterprise_influxdb/v1/guides/query_data/)

## 17. Що далі

Урок 2 розкладе InfluxDB v1 на конкретні сутності: database, retention policy, measurement, tags, fields, series, point, shard і shard group. Після нього ми зможемо точніше пояснити, чому дві візуально схожі line protocol lines можуть мати зовсім різну ціну для індексу та storage engine.
