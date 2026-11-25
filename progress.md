# Прогрес навчання

## Урок 1. Що таке TSDB — пройдено 2026-11-18

Формат: інтерактивна сесія за матеріалом [`lessons/01-what-is-a-tsdb.md`](lessons/01-what-is-a-tsdb.md).

### Що засвоєно

- TSDB оптимізує append-потік, часові діапазони, агрегацію по вікнах і lifecycle даних.
- Внутрішня модель InfluxDB v1 (TSM) на концептуальному рівні:
  - дані групуються за series (measurement + tag set); series key зберігається один раз;
  - стовпчики часу та значень стискаються (delta/RLE для timestamps, XOR/Gorilla для float);
  - шлях запису: WAL → in-memory cache → TSM-файли → compaction;
  - shard groups ріжуть дані за часом; retention видаляє цілі shard groups;
  - індекс (TSI) працює по tags; fields не індексуються;
  - TSM-індекс блоку зберігає лише min/max **time**, а не статистику значень.
- Виконання запиту: tags → series → shards/блоки за часом → розпакування й агрегація.
- **Вартість ≈ series × часовий діапазон × щільність точок.** Розмір відповіді не означає вартість.
- `GROUP BY time()` зменшує результат, але не обсяг читання з raw-даних.
- Обсяг читання зменшують агрегати (continuous query), tag-фільтри та коротший діапазон.
- Для злиття середніх треба зберігати `sum` + `count`, бо середнє із середніх неправильне.
- `LIMIT` не обмежує роботу бази.
- Кожен запит, навіть `last()`, повинен мати `WHERE time`.

### Результати перевірок

| Перевірка | Результат |
|---|---|
| Проблеми time series у Postgres | Добре: розмір, індекси, повільна агрегація. Доповнено: retention через DELETE, bloat |
| Чому retention дешевий | ✅ Уточнення: shard містить усі measurements RP; дані живуть до duration + shard duration |
| Розбір дворічного `mean()` | Частково: правильна ідея часткових агрегатів. Помилка: вважав, що min/max уже пораховані в TSM |
| Як зробити дворічний графік дешевим | Частково: запропонував `GROUP BY time()`, але це не зменшує читання |
| Raw + 1 series проти агрегатів + 2 000 series | ❌ Обрав агрегати. Правильно: raw з однією series дешевший у ~5,5 раза |
| Вправа з 2 000 пристроїв | 4/5. `last()` для одного пристрою — природна дешева задача для Influx |
| `LIMIT`, `request_id` у tag, три питання перед запитом | ✅ |

### Прогалини, до яких повернутися

- Термінологія: новий tag value створює нову **series**, а не shard чи колонку. Розібрати в уроці 2.
- Не плутати `mean` і `min`.
- Інтуїція «агрегати завжди дешевші» хибна: series множать вартість. Закріпити в уроці 7.

### Особистий досвід

- `LIMIT 1` на важкому запиті вже клав базу: обмеження результату не обмежило читання.

### Наступний крок

Урок 2: database, RP, measurement, tag set, field set, series key, cardinality, shard і shard group.

## Урок 2. Логічна та фізична модель InfluxDB v1 — пройдено 2026-11-24

Формат: інтерактивна сесія за матеріалом [`lessons/02-influxdb-v1-data-model.md`](lessons/02-influxdb-v1-data-model.md).

Основну частину пройдено 2026-11-22, фінальну усну вправу — 2026-11-24.

### Що пройдено

- Розбір line protocol на measurement, tag set, field set і timestamp.
- Point як один time-stamped запис, який може містити кілька field values.
- Розрізнення трьох рівнів:
  - indexed series identity: measurement + tag set;
  - TSM field stream: measurement + tag set + field key;
  - duplicate point identity: measurement + tag set + timestamp.
- Зміна field value не створює нову indexed series; нова tag-комбінація створює.
- Database → retention policy → measurement → time-based shard group → series/field streams.
- Timestamp може спрямувати ту саму series в інший shard group.
- RP визначає retention duration, replication factor, shard group duration і default status.
- Cardinality для незалежних і залежних tags; залежні dimensions не треба механічно перемножувати.
- Tag проти field:
  - tags — індексовані string dimensions для series selection і grouping;
  - fields — виміряні values, які не формують tag index.
- Field types у line protocol: float, integer із суфіксом `i`, boolean і quoted string.
- Timestamp precision (`s`, `ms`, `us`, `ns`) та ризик випадкових duplicate collisions при надто грубій precision.
- Duplicate writes у v1: field sets об'єднуються, а нове значення того самого field key перемагає.
- Measurement має описувати стабільний тип даних; entity ID не слід кодувати в measurement name.
- Не варто використовувати однакову назву для tag key і field key.

### Результати перевірок

| Перевірка | Результат |
|---|---|
| 3 line protocol записи з `api-01` / `api-02` | Початково не помічено різницю в `host`; після уточнення point/series/stream модель зрозуміла |
| Field update з тим самим timestamp | ✅ Правильно: інші fields залишаються, нове значення конфліктного field key перемагає |
| Cardinality: 20 services × 4 regions × 5 methods | ✅ 400 для незалежних tags; 100, якщо region однозначно залежить від service |
| `region`, `temperature`, `request_id`, HTTP method як tag/field | ✅ Правильна класифікація; зрозумілий ризик `request_id` як tag |
| Перша вправа з `device_health` | Правильно визначено 3 points і 2 indexed series. Підрахунок field streams і повторний запис у наявну series закріплено в фінальній вправі. |
| Фінальна вправа з `device_health` | ✅ Правильно пораховано 3 indexed series і 6 field streams, по 2 на series. |
| Наступна доба та повторний write | ✅ `A1/us-east` повторно використовує наявний series key, а timestamp після межі доби UTC спрямовує point у наступний добовий shard group. Повторний write із тим самим timestamp оновлює `battery` і зберігає `temperature`. |

### Що закріплено

- Для кожного field key, присутнього в конкретній indexed series, є окремий field stream. У фінальній вправі це 3 series × 2 fields = 6 field streams.
- Залежний tag не збільшує кількість комбінацій лише поки зв'язок справді однозначний у записаних series. `A1/us-east` і `A1/eu-west` — різні series keys.
- Індекс поводиться як множина унікальних keys: повернення до вже наявного `A1/us-east` не створює ще одну series; абсолютно новий `A1/us-south` створює.
- Не плутати кількість writes, logical points після duplicate merge, indexed series identities та field-level streams.

### Робочий приклад, використаний у розмові

Redacted масштаб: приблизно 5 000 значень `asset_id`, один point на п'ять хвилин; кількість assets зросла приблизно зі 100 у 2017 році до 5 000 у 2022 році та вийшла на плато. Це використано лише для закріплення різниці між point volume і series cardinality. Рішення щодо додавання `location_id` відкладено на прохання користувача.

### Наступний крок

За [рекомендованим порядком](LEARNING_PLAN.md#рекомендований-порядок-перших-занять) далі йде концептуальна частина уроку 7 про вартість запитів. Почати з оцінки дворічного InfluxQL-запиту на синтетичному прикладі.

## Урок 3. Теги, поля і cardinality — пройдено 2026-11-24

Практика на синтетичному `http_requests` для InfluxDB Cloud 1 / Enterprise v1.

- Вибирали tags за запитами дашборда. `GROUP BY` працює з tags, а `COUNT(field)` рахує значення поля в кожній групі.
- Пораховано 36 фактичних комбінацій із `status_code` і верхню межу 960 series після додавання до 20 маршрутів до 48 можливих комбінацій. Межа перевищує умовний бюджет 800.
- Уточнено підрахунок для повного URL. 10 000 запитів з унікальними URL дають 10 000 series, якщо кожен запис має одну комбінацію решти тегів. Повторюваний URL у двох різних tag sets дає дві series. Для групування за маршрутом підходить нормалізований шаблон `route`.
- У вправі з бюджетом 500 обрано `route` і `region` як tags, а `status_code` як field. Верхня межа 240 series. Для кодів 502, 503 і 504 запропоновано три запити з field-фільтрами. Обговорили повторне читання, підрахунок у застосунку, фіксовані поля та окремий measurement з агрегатами.
- Для грубої оцінки в обраній базі обрано `SHOW SERIES CARDINALITY`. `EXACT` може дорого коштувати на великій базі. Додаткові умови на кшталт `FROM` або `WHERE` також можуть спричинити точний підрахунок.
- Динамічні назви measurements збільшують кількість series. Динамічні field keys збільшують кількість TSM-потоків навіть за сталого числа індексованих series. У вправі з p95 правильно обрано рядковий field `tenant_id`. Уточнено, що `latency_ms` зберігається як числовий field у стабільному measurement `http_requests`.

Для наступних занять підійдуть задачі з кількома робочими рішеннями й різною вартістю запису, читання та зберігання.

Джерела Enterprise v1: [дизайн схеми](https://docs.influxdata.com/enterprise_influxdb/v1/concepts/schema_and_data_layout/), [InfluxQL `GROUP BY`](https://docs.influxdata.com/enterprise_influxdb/v1/query_language/explore-data/#the-group-by-clause), [InfluxQL `PERCENTILE()`](https://docs.influxdata.com/enterprise_influxdb/v1/query_language/functions/#percentile), [InfluxQL `SHOW SERIES CARDINALITY`](https://docs.influxdata.com/enterprise_influxdb/v1/query_language/spec/#show-series-cardinality).

### Розбір бази prod і підсумок сесії

- Зафіксували [схему production](notes/system-map.md) і знеособлені приклади `wear`, `oil_quality`, `oil_quality_daily` та `oil_quality_health_indicator`. Measurement описує тип телеметрії, а індексований `modelId` відповідає основному запиту дашборда за конкретним сенсором. Точне значення базової оцінки cardinality ще треба звірити зі збереженим результатом.
- Підтвердили налаштування retention. `prod.autogen` безстрокова, використовує тижневі shard groups і `replicaN=2`. `prod.short_lived` має п’ятиденний строк, добові shard groups і `replicaN=1`. Надані oil-запити явно читають `prod.autogen`. Фактичний розподіл writers між policies ще відкритий.
- `GROUP BY time(...)` у наданому `oil_quality_daily` усереднює точки двох сенсорів спільно. Для окремих графіків потрібне також групування за `modelId`. Заміна raw на daily вимагає звірки значення метрики, а для відновлення середнього за raw samples потрібні ваги або `sum` і `count`.
- На 12-годинному дашборді цього кластера CPU переважно близько 22–26% на найзавантаженіших лініях, RAM близько 55–70% із коротким піком понад 90%, а частота стартів запитів має майже погодинні піки. За цим зрізом причина піка RAM лишається відкритою. Параметри retention для діагностики не змінювали.

**Точка продовження для уроку 7.** Розібрати один реальний запит дашборда `oil_quality` з фактичним часом, кількістю сенсорів і полями, потрібними на графіку. За можливості зіставити його з відповідним daily-показником. На робочому комп’ютері лишається відкрити [код Node.js cron](notes/system-map.md#наступні-дані-для-карти) або принести знеособлений фрагмент для звірки.

## 2026-11-25. Розбір коду бекенду (робочий комп'ютер)

Не урок, а доповнення [карти системи](notes/system-map.md) з коду бекенду.

- Бекенд використовує v1-клієнт npm `influx` (node-influx 5.12): `/write` + InfluxQL `/query`. Урок 5 перебудовано під цей клієнт.
- `setupInflux()` на старті кожного інстансу виконує `SHOW DATABASES`, за потреби `CREATE DATABASE`, і створює `short_lived` (5d, replication 1, shard group 1d за замовчуванням).
- У `short_lived` пише лише `bad_data`, усе інше йде в `autogen`.
- Три in-memory буфери до 10 000 points. Основний відкидає batch при будь-якій помилці, bulk повторює без ліміту.
- `maxRetries: 4` фактично не діє з одним хостом. Після таймауту хост вимикається на backoff, і всі запити інстансу падають з `No host available`.
- Чотири 60-секундні задачі (`RUN_CRONS=true`) працюють на кожному з 4–8 інстансів без lock і без захисту від накладання. Найважчі за кодом: smoothing `MOVING_AVERAGE … INTO` щохвилини і `getLatestPriorToTs` з lookback до 91 доби.
- App Engine cron: погодинний `runBackfillingDataFix` з `DELETE` до 6 місяців, `calculateVisc100C`, щоденний `DROP SERIES` о 01:00 UTC. Причину майже погодинних піків свідомо не шукаємо: джерел багато, зокрема FTP-імпорт великих CSV і зовнішні скрипти через API-сервіс.
- Клієнтські застосунки (Oil App, Admin App, i-alert) напряму в Influx не ходять, а читають через `/data/*` бекенду. Бекенд стискає відповідь через `GROUP BY time()`, але діапазон читання не обмежує.

**Наступний урок:** 7 (вартість запиту) за [рекомендованим порядком](LEARNING_PLAN.md#рекомендований-порядок-перших-занять) — це єдиний із ранніх уроків, який ще не пройдено. Реальний кейс — `getLatestPriorToTs` (`ORDER BY time DESC LIMIT 1` з lookback 19 год → 30 діб → 91 доба).

## Урок 7. Вартість запиту і захист кластера — пройдено 2026-11-25 (концептуально)

Формат: інтерактивна сесія, конспект у [`lessons/07-query-cost.md`](lessons/07-query-cost.md). Практику з `EXPLAIN` на локальному стенді відкладено: Docker Desktop не був запущений.

### Що засвоєно

- Запит у v1 має три етапи: планування (shards × series × fields), розпакування TSM blocks і відповідь. `GROUP BY time()` і `LIMIT` зменшують лише відповідь.
- Шард без даних series відповідає швидко, але кожен шард треба відкрити. У разі порожніх lookback основна робота припадає на планування.
- У цій системі головний ризик — кратність дешевих запитів (частота × паралельність × 4–8 інстансів без lock), а не окремий важкий запит.
- Shard group 30 днів дозволено, але `ALTER` діє лише на нові групи, а при наших `INTO`, `DELETE` і backfill більші шарди дорожче перекомпактовуються. Це не перший важіль.
- «У логах немає» ≠ «не відбувається»: перевіряти рівень логера та версію перед висновком.

### Результати перевірок

| Перевірка | Результат |
|---|---|
| `getLatestPriorToTs` з прогалиною 40 діб | Частково. Правильно: основна робота в плануванні. Помилка: один запит замість трьох послідовних (≈ 21 shard group) |
| Рік raw `oil_quality`, 58 fields | ❌ Арифметика: 13–15 тис. замість ≈105 000 на field (≈6,1 млн значень). ✅ Відповідь ~1000 точок, daily у сотні разів дешевше, сенс і покриття полів треба звірити |
| Ранжування A–D | Частково. Обрано C > D > B > A. Недооцінено частоту й паралельність: B пише щохвилини (~1 860 перезаписів точки), A дає тисячі одночасних запитів. C — meta-запит по індексу, а не по даних |
| Практичне заперечення | ✅ Слушно: 91-денна гілка `getLatestPriorToTs` — last resort, бо обчислення беруть пристрої зі свіжими даними. Доказ із логів поки слабкий: лог має рівень `debug` |

### Прогалини, до яких повернутися

- Швидкий підрахунок точок: точок на добу × дні × fields. Закріпити в уроці 8.
- Оцінюючи ризик, завжди множити на частоту, паралельність і кількість інстансів.

### Відкриті дії

- Перевірити рівень winston у production і що логує `withTimeout` при спрацюванні.
- Шукати в логах бекенду `No host available` і таймаути поруч із піками.
- Провести локальну практику з `EXPLAIN` (див. конспект).

Джерела Enterprise v1: [EXPLAIN / EXPLAIN ANALYZE](https://docs.influxdata.com/enterprise_influxdb/v1/query_language/spec/#explain-analyze), [query management](https://docs.influxdata.com/enterprise_influxdb/v1/troubleshooting/query_management/influxql_query_management/), [shard group duration](https://docs.influxdata.com/enterprise_influxdb/v1/concepts/schema_and_data_layout/#shard-group-duration-recommendations), [filter meta queries by time](https://docs.influxdata.com/enterprise_influxdb/v1/query_language/explore-schema/#filter-meta-queries-by-time).

## Урок 4. Запис: line protocol і write path — пройдено 2026-11-25 (теоретично)

Формат: інтерактивна сесія, конспект у [`lessons/04-write-path.md`](lessons/04-write-path.md). Урок теоретичний: практику користувач робить у реальному проєкті, локальний стенд не плануємо.

### Що засвоєно

- Write path v1/Enterprise: координатор → shard group за timestamp → 2 власники → WAL + cache → TSM → compaction. Кожен зачеплений шард тримає свій cache.
- `400` partial write означає, що валідні points записані. `5xx`/таймаут можна повторювати, бо запис з явним timestamp ідемпотентний.
- Політика помилок наших буферів перевернута: основний відкидає, bulk повторює без меж. Правильний напрям — circuit breaker, наростаючий backoff і обмеження буфера (запропоновано самостійно).
- Smoothing `INTO` — головний write amplifier: ~370× сирого потоку на синтетиці. Користувач погодився, що це треба виправити.

### Результати перевірок

| Перевірка | Результат |
|---|---|
| Розмір batch 10 000 × 40 fields | ✅ ~10 MB. Сплутано `max-body-size` 25 MB (вкладаємося) з рекомендацією 5 000 рядків (перевищуємо) |
| Сценарій «кластер гальмує 90 с» | Частково. ✅ ріст навантаження, circuit breaker і backoff. Помилка: основний буфер не зберігає batch, а відкидає. Не розглянуто `No host available` для алертів і cron |
| Smoothing amplification | ✅ висновок. Арифметика: 31 × 12 = 600 замість 372 |
| Фінальна оцінка FTP-імпорту | Частково. ✅ ~5 shard groups, висновок «розбивати й розтягувати». Помилки: points помножено на fields (18 млн замість 432 000 points, 44 batch, ~4 хв), поля названо series (сенсори вже існують → 0 нових series, 40 fields — це field streams) |

### Факти з production, принесені на уроці

- `Couldn't write` у Cloud Logging за місяць: 3 випадки, ~1 000 points на весь потік. Втрати основного буфера реальні в коді, але рідкісні → низький пріоритет.
- FTP-джерела формують CSV не більше доби й 2–3 MB. У коді бекенду цього обмеження немає.

### Прогалини, до яких повернутися

- Points ≠ field values: batch і shard рахуються в points, fields множать лише байти й field streams.
- Fields не створюють series (повтор з уроку 2).
- Користувач просить менше ручної арифметики й більше концептуальних висновків. Розрахунки давати готовими, а перевіряти розуміння одиниць.

Джерела Enterprise v1: [write API і коди відповіді](https://docs.influxdata.com/enterprise_influxdb/v1/guides/write_data/#http-response-summary), [line protocol](https://docs.influxdata.com/enterprise_influxdb/v1/write_protocols/line_protocol_reference/), [backfill і shard duration](https://docs.influxdata.com/enterprise_influxdb/v1/concepts/schema_and_data_layout/#shard-group-duration-for-backfilling), [historical data FAQ](https://docs.influxdata.com/enterprise_influxdb/v1/troubleshooting/frequently-asked-questions/#what-are-the-configuration-recommendations-and-schema-guidelines-for-writing-sparse-historical-data), [write consistency і hinted handoff](https://docs.influxdata.com/enterprise_influxdb/v1/concepts/clustering/#write-consistency), [`write-timeout`](https://docs.influxdata.com/enterprise_influxdb/v1/administration/configure/config-data-nodes/#write-timeout).
