# Карта production InfluxDB для курсу

Станом на 2026-11-25: схема — з розмови 2026-11-24, підключення й фонові задачі — з коду 2026-11-25. Ця карта фіксує схему й робочі сценарії бази `prod` для подальших уроків про InfluxDB Cloud 1 / Enterprise v1. Для вправ використовуємо редаговані зразки або локальні дані з тією самою структурою й характером навантаження.

## Схема й потік даних

Сенсор надсилає телеметрію → система записує raw points в InfluxDB → фонові задачі бекенду читають raw і записують похідні measurements (деталі нижче) → дашборд читає телеметрію та підсумки за сенсорами. У сценарії дашборда також є перевірка алертів за даними сенсорів. Алерти перевіряє бекенд щохвилини, див. `checkAllDevicesForAlarms` нижче.

| Елемент | Що відомо |
| --- | --- |
| База й retention policies | Production-база `prod` має `autogen` і `short_lived`. Фактичні параметри наведено нижче. З коду: у `short_lived` пише лише `bad_data`, усе інше йде в default `autogen`. |
| Measurements | Назви за типом сенсора або телеметрії, наприклад `wear`, `oil_quality`, `vibration`, `viscosity`. Окремі похідні measurements включають `oil_quality_daily`, `oil_quality_processed`, `oil_quality_health_indicator` і `viscosity_processed`. |
| Ідентичність сенсора | `modelId` — tag з ID конкретного сенсора та основний фільтр. Один сенсор належить до одного типу. У зразках дашборд запитує той самий `modelId` в `oil_quality` та `oil_quality_health_indicator`. Наявність інших тегів треба перевіряти для кожного сценарію. |
| Fields | Зазвичай близько 20–50 fields на сенсор. У `wear` поруч зберігаються raw-значення, cumulative-показники `cum_*` і поля стану пристрою. Джерело, семантика та поведінка скидання `cum_*` ще відкриті. |
| Cardinality | Вимір зроблено для `prod`. Поточний орієнтир зі слів користувача — близько **53 000 series**. У чаті раніше наведено **56 901**. Перед порівнянням з наступним виміром варто звірити точне число з наявним результатом. Біля `wear → modelId` в `prod.autogen` Chronograf показував **11 853** значення тегу. |

Число біля `modelId` є орієнтиром для series у межах показаної схеми `wear`, якщо інших тегів немає. Стан кластера оцінюємо за динамікою cardinality разом із RAM, latency, помилками й навантаженням у години пік. [Cardinality в Enterprise v1](https://docs.influxdata.com/enterprise_influxdb/v1/query_language/spec/#show-series-cardinality)

## Retention policies бази prod

Параметри з `SHOW RETENTION POLICIES ON "prod"`.

| Policy | Строк зберігання | Розмір shard group | Replication factor | За замовчуванням |
| --- | --- | --- | --- | --- |
| `autogen` | Безстроково (`0s`) | 7 діб (`168h`) | 2 | Так |
| `short_lived` | 5 діб (`120h`) | 1 доба (`24h`) | 1 | Ні |

У результаті обидва поля `futureWriteLimit` і `pastWriteLimit` мають `0s` для кожної policy. `short_lived` видаляє дані цілими добовими shard groups, коли строк зберігання сплив для всього інтервалу групи. Надані запити `oil_quality`, `oil_quality_daily` і `oil_quality_health_indicator` явно читають із `prod.autogen`. П’ятиденний строк `short_lived` стосується даних, які записують саме туди. Наявність безстрокової policy пояснює, як може накопичуватися історія на диску. Внесок у RAM або піки затримки потрібно звіряти з моніторингом. [Параметри retention policies](https://docs.influxdata.com/enterprise_influxdb/v1/query_language/manage-database/#retention-policy-management), [Видалення shard groups](https://docs.influxdata.com/enterprise_influxdb/v1/concepts/schema_and_data_layout/#short-shard-group-duration)

## Підключення з бекенду (з коду, 2026-11-25)

Джерело — репозиторій бекенду, файли `src/db/influx.js`, `src/services/influxClient.js`, `src/services/influxBatchWriter*.js`, `cron/prod/cron.yaml`, `app*.yaml`. Секрети, хости й назви баз не копіюємо.

### Клієнт і протокол

| Що | Факт із коду | Наслідок |
| --- | --- | --- |
| Бібліотека | npm `influx` (node-influx) **5.12**, не `@influxdata/influxdb-client` | Це **v1-клієнт**: пише через `/write` (line protocol), читає через `/query` (InfluxQL). Питання «чи сумісний v2 client з Cloud 1» для бекенду не стоїть. Урок 5 перебудовано під node-influx. |
| Конфіг | `INFLUX_HOST`, `INFLUX_DB`, `INFLUX_USERNAME`, `INFLUX_PASSWORD`, `INFLUX_METHOD` (`http`/`https`) з env; порт не задано → дефолт 8086 | Один хост. Секрети приходять через `RUNTIME_ENV_SECRET`, у репозиторії їх немає. |
| Пул для віддаленого хоста | `requestTimeout: 60000`, `maxRetries: 4` (у `DEBUG=true` — 1) | Див. нижче: з одним хостом ретраї фактично не спрацьовують. |
| Локальний хост (`localhost`/`influx`) | лише `requestTimeout: 60000` | Локальний стенд (docker) поводиться так само, крім ретраїв. |
| Окремий клієнт | `influxClientDemoDb.js` для demo-бази | Окреме навантаження від demo-копіювання (див. cron). |

### Що робить `setupInflux()` при кожному старті інстансу

1. `getDatabaseNames()` → `SHOW DATABASES`.
2. Якщо бази з `INFLUX_DB` немає — **`CREATE DATABASE`**. Помилка в env (опечатка в назві) на production тихо створить нову порожню базу, і writers почнуть писати туди.
3. `SHOW RETENTION POLICIES ON <db>`; якщо немає `short_lived` — `CREATE RETENTION POLICY "short_lived" ... DURATION 5d REPLICATION 1` без `SHARD DURATION` і без `DEFAULT`.

Висновки:

- Звідси походять параметри `short_lived` з таблиці вище: для duration від 2 днів до 6 місяців v1 за замовчуванням ставить shard group duration 1 день — це і є `24h`. [CREATE RETENTION POLICY](https://docs.influxdata.com/enterprise_influxdb/v1/query_language/manage-database/#create-retention-policies-with-create-retention-policy)
- `short_lived` не default-policy, тож усе, що пишеться без явного RP, потрапляє в безстрокову `autogen`.
- Помилки на старті лише логуються, застосунок стартує навіть без бази.
- Застосунку потрібні права admin/`CREATE` на старті. Для least-privilege варто знати, що без них boot лише залогує помилку.

### Write path: три буфери в пам'яті процесу

`dataMethods.js` формує points так: `measurement = measurementKey` з конфігурації атрибутів пристрою, fields — значення атрибутів, tags — `modelId` (Mongo `_id` пристрою або asset) плюс необов'язкові `userId`, `oilSampleTypeId`. Timestamp — наносекунди (`unixTStoNanoTS`), `precision` за замовчуванням `n`.

| Буфер | Хто використовує | Коли пише | RP | Що з помилками |
| --- | --- | --- | --- | --- |
| `influxBatchWriter` | основний потік телеметрії (`save…`, `saveRaw` без RP) | ≥1000 points і ≥5 с від минулого запису, або >60 с і є points; максимум 10 000 points за запит | default (`autogen`) | **Будь-яка помилка → batch відкидається** (`removeAllBad = true`), хоч коментар каже «keep in buffer». Таймаут, 5xx або «No host available» = втрачені дані. |
| `influxBatchWriterRp` | лише `bad_data` (`badDataMethods.js`) | так само | `short_lived` | так само відкидає |
| `influxBatchWriterBulk` | імпорти, `saveMultiplePoints*`, міграційні скрипти | кожні 5 с, до 10 000 points | default | При 400 відкидає, при інших помилках **повторює той самий batch кожні 5 с без ліміту**, буфер росте без межі. |

Додаткові деталі, важливі для уроку 4:

- Перевірка «час флашити» у двох основних буферах викликається лише при надходженні нових points; таймера в production немає. Хвіст буфера чекає наступного запису й зникає при рестарті інстансу.
- Буфер живе в пам'яті кожного інстансу. Default-сервіс у prod — App Engine flex з автоскейлінгом **4–8 інстансів**, тож буферів, флашів і одночасних writes до Influx у стільки ж разів більше.
- `bad_data` має tag `errorCode` і string-fields `rawPayload` / `rawCsvRow` із повним сирим payload. Це великі string fields, тому вони свідомо в 5-денній `short_lived`.
- `writePoints` без `gzip`; batch до 10 000 points × 20–50 fields — це помітний HTTP-payload.

### Таймаути й ретраї node-influx (перевірено в `node_modules/influx/lib/src/pool.js`)

- Ретрай спрацьовує тільки на мережеві помилки (`ETIMEDOUT`, `ECONNRESET` тощо), HTTP 5xx і клієнтський таймаут. 4xx (наприклад, `partial write` або field type conflict) не ретраїться.
- Перед ретраєм хост вимикається на backoff 300 мс → … → 10 с, а ретрай можливий лише якщо **є інший доступний хост**. Хост у нас один, тому `maxRetries: 4` фактично не працює.
- Поки хост вимкнений, **усі** запити цього інстансу (writes, queries, `/ping` health-check) одразу отримують `No host available`. Один повільний запит понад 60 с може спричинити каскад: writer відкидає batch, алерти й cron-запити падають.
- Клієнтський таймаут 60 с обриває HTTP-з'єднання. Чи скасовує це запит на сервері Cloud 1, треба перевірити через `SHOW QUERIES` на безпечному стенді, а не припускати.

### Що з цього випливає для інцидентів

| Гіпотеза | Як перевірити |
| --- | --- |
| Повільний запит → backoff → тихі втрати telemetry | Логи `Couldn't write N points` і `No host available` поруч у часі з повільними запитами. |
| Bulk-імпорт під час проблем з кластером не відпускає навантаження | Логи `BULK ERROR` з повтором кожні 5 с, ріст heap інстансу. |

## Джерела навантаження і клієнти

Майже погодинні піки свідомо не прив'язуємо до однієї причини. Навантаження на Influx мають кілька незалежних джерел, і в такому проєкті пік може дати будь-яке з них.

| Джерело | Як потрапляє в Influx | Характер навантаження |
| --- | --- | --- |
| Сенсори й gateway | HTTP-імпорт у default-сервіс бекенду → буфер `batchWrites` | Постійний потік raw points і щохвилинні обчислення з `cronService` |
| FTP-імпорт (`ftp_jobs`) | Великі CSV з багатьма даними → імпорт бекенду | Сплески записів, backfill за історичні періоди, далі перерахунки |
| API-сервіс (`api-service`, маршрути `/api`) | Зовнішні скрипти клієнтів | Можуть читати великі обсяги даних, частота і діапазони не контролюються нами |
| Клієнтські застосунки | Через REST бекенду, **не напряму** в Influx | Див. таблицю нижче |
| Chronograf | Напряму InfluxQL, шаблони з `:dashboardTime:` | Внутрішні дашборди. Звідси походять зразки запитів `wear` та `oil_quality` вище. |
| Фонові задачі бекенду | `cronService` і App Engine cron | Див. [фонові задачі](#хто-читає-й-пише-фонові-задачі-з-коду-2026-11-25) |

### Клієнтські застосунки

Монорепозиторій `integrated-platform` (single-spa) містить Oil App, Admin App, common dashboard та i-alert. У жодному з них немає InfluxDB-клієнта або адреси бази. Усі дані йдуть через маршрути бекенду `/data/*` (`src/routes/queryDataRoutes.js`).

| Застосунок | Маршрути, що читають Influx |
| --- | --- |
| **Oil App** — головний читач телеметрії | `chartData`, `tableData`, `qxChartData`, `qxEngineeringData`, `healthIndicatorData`, `oilQualityDeviceLatestData`, `dataExplorerMultiData`, `wearDebrisTable`, `getAlarmCharts`, `viscEstimate`, `allRul`, FFT (`getFFTsInRange`, `getWaterfallFFTs`), звіти `getReport`, експорт `exportData`, `requestDataExport`, `exportMultiData` |
| Admin App | Лише `indicatorWidget` |
| i-alert | `chartData`, `exportData`, `getReport` |
| `frontend_v2` (окремий репозиторій) | Ті самі маршрути, що й Oil App; схоже на попередню версію |

Захисти на графіках бекенду (`dataMethods.js`): кількість точок у відповіді обмежується приблизно 10 000 на графік і 1 000 на series через автоматичний `GROUP BY time(...)`, а для експорту є оцінка до 11 млн точок. Діапазон `t1…t2` не обмежується. Як у [уроці 1](../lessons/01-what-is-a-tsdb.md): `GROUP BY time()` зменшує відповідь, але не обсяг прочитаних raw-даних. Довгий графік у Oil App читає весь raw-діапазон.

## Хто читає й пише: фонові задачі (з коду, 2026-11-25)

Шаблони запитів скорочено, `<id>` — `modelId`. Жоден запит фонових задач не вказує RP, тож усе читається й пишеться в default `autogen`. Обчислені точки отримують детермінований timestamp (час вихідної точки або опівніч), тому повторний прохід перезаписує ті самі points через field merge, а не дублює їх.

### Два механізми запуску

| Механізм | Де | Скільки копій працює |
| --- | --- | --- |
| In-process `setInterval(…, 60 с)` | `src/services/cronService.js`, вмикається `RUN_CRONS=true` (так у `.env.prod`; живе значення — у Secret Manager, не звірено) | **На кожному інстансі** default-сервісу prod (автоскейлінг 4–8). Lock або leader election не знайдено. Захисту від накладання тиків немає. |
| App Engine HTTP cron | `cron/prod/cron.yaml` → `/cron/*` | Один виклик на розклад. Чи App Engine пропускає запуск, поки попередній ще працює, — не перевірено. |

### 60-секундні задачі

| Задача | Що вибирає | Fan-out | Що робить з Influx |
| --- | --- | --- | --- |
| `runPostImportCalculations` | Пристрої wear і QX, що прислали дані за останні ~13 хв (атомарний claim документів `DeviceDataHistory`) | До 60 пристроїв паралельно на інстанс. 45 с `withTimeout` звільняє слот, але не скасовує запит. | Wear: `getLatestPriorToTs`, `SELECT LAST(*) FROM wear` з lookback до 91 доби, `SELECT * FROM wear` за діапазон, запис `cum_*`. QX: engine/oil hours, health indicator і oil change з **7-денним буфером** `oil_quality_processed`, для v2 HI — 60 діб `oil_quality_daily`. Потім **smoothing**: до 8 statements `SELECT MOVING_AVERAGE(...) INTO "<m>" FROM "<m>" WHERE time ≥ t1−30 год … GROUP BY *`. Разом приблизно 12–20 HTTP-запитів на HI-пристрій за тик. |
| `checkAllDevicesForAlarms` | Документи останніх 20 хв, по яких уже пройшли обчислення (claim не атомарний) | 120 пристроїв × 5 алармів паралельно, до ~600 одночасних перевірок на інстанс | Приблизно один запит на точку × аларм × умову: `SELECT MAX/MIN/SPREAD("<field>") FROM "<m>" WHERE "modelId"='<id>' AND time …` у вікні 2 год. Для атрибутів із query function — вкладені `MOVING_AVERAGE(NON_NEGATIVE_DERIVATIVE(CUMULATIVE_SUM(...)))` з вікном до 14 діб. У Influx не пише. |
| `checkAllDevicesForInactivityAlarms` | Mongo `latestDataTime` | 120 | Influx не читає, поки аларм не спрацює. |
| `runProcessLargePackets` | До 5 пакетів понад 250 samples (статус не атомарний) | 100 samples паралельно | Для `oilQualityFixed` — по одному 3-годинному raw-читанню `oil_quality` (~16 fields) на sample, далі запис raw через `batchWrites`. |
| `processOilChangeQueue` | Черга `oilChangeRecalc` після створення або редагування oil change | Послідовно | Перерахунок oil hours і HI від `подія − 2 доби` до now (подія може бути до 2 років тому), `DELETE FROM "oil_quality_daily" WHERE "modelId"=… AND time …` і перерахунок daily. **Ймовірний баг:** `Job.find(query, {sort, limit})` передає опції як projection, тому ліміт 100 не діє. |

Ключовий запит `getLatestPriorToTs`, який викликають майже всі обчислення:

```sql
SELECT "<field>" FROM "<m>"
WHERE time >= <t1 - 19h> AND time < <t1> AND "modelId" = '<id>'
ORDER BY time DESC LIMIT 1
-- порожньо → той самий запит з lookback 30 діб, потім 91 доба
```

Уточнено в [уроці 7](../lessons/07-query-cost.md): обчислення беруть пристрої зі свіжими даними, тому зазвичай спрацьовує лише 19-годинний запит. Гілки 30 і 91 доба — last resort для першої точки після прогалини (простій, backfill, FTP-імпорт історії): три послідовні запити й ≈ 21 shard group, переважно планування. Ризик — у кратності: × fields × пристрої × хвилини × інстанси.

### App Engine cron, що торкаються Influx

| Route (розклад) | Що робить |
| --- | --- |
| `runBackfillingDataFix` (щогодини) | Пристрої, що закінчили backfill за останні 65 хв, до 10 паралельно. Wear — повний перерахунок 30-денними шматками. QX HI — **`DELETE FROM "oil_quality_health_indicator"` за діапазон до 6 місяців**, перерахунок, через 3–5 хв smoothing `INTO`, а також DELETE і перерахунок daily без `await`. **Баг:** якщо діапазон понад 6 місяців, `ts1` обрізається, але перерахунок пропускається. |
| `calculateVisc100C` (щогодини) | В'язкісні пристрої з даними за добу, до 10 паралельно: `COUNT` і `SELECT *` з `viscosity_processed` за 23,5 год з field-фільтрами, запис у `viscosity_processed`. |
| `recalculateMergedDeviceData` (щохвилини) | Зазвичай 0 записів. Після заміни пристрою — повний перерахунок до 730 діб. |
| `deleteDeviceReplacementDuplicateData` (01:00 UTC) | **`DROP SERIES WHERE "modelId" = '<new id>'`** без measurement і без часу — по всіх measurements і shards бази. |
| `runDemoDataCopyNew` (02:00 UTC) | Видаляє й копіює demo-дані (не трасовано). |
| `calculateQxDailyValues` (03:00 UTC) | **Саме він пише `oil_quality_daily`.** QX-пристрої з даними за 3 доби, послідовно, близько 12 запитів на пристрій: `MEAN/SPREAD/COUNT("processed_*")` з `oil_quality_processed` за минулу добу та `LAST` з `oil_quality_daily`. Timestamp — опівніч у часовому поясі контейнера. |
| `setSecondsBetweenPts` (щодня) | `MODE(ELAPSED(...))` за добу для нових пристроїв. Лише читання. |
| `generateMonthlyReports` (2-го числа, 04:00) | Читання для звітів (не трасовано). |

Також є `SHOW TAG VALUES WITH KEY = "<tag>"` без `FROM`/`WHERE` по всій базі (`getOilSampleTypes`) і ручні `DELETE` у `editDataMethods.js`, зокрема варіант без measurement — по всіх measurements.

### Ризики для кластера (гіпотези з коду, потребують підтвердження метриками)

| # | Ризик | Чим перевірити |
| --- | --- | --- |
| 1 | 60-секундні задачі × 4–8 інстансів без lock і overlap guard; повільні тики накладаються | Логи `cronService` з різних інстансів у ту саму хвилину, колекція `cronStatus`, `SHOW QUERIES` у пік |
| 2 | Smoothing `INTO` щохвилини перезаписує ~31 год даних кожного активного QX-пристрою | Write throughput у моніторингу Cloud 1 проти кількості активних QX-пристроїв |
| 3 | Широкі lookback: 19 год → 30 діб → 91 доба, `LAST(*)` до 91 доби, 7-денні буфери на кожен тик | Slow query log, тривалість запитів за шаблоном |
| 4 | Таймаути не скасовують роботу (`withTimeout` = `Promise.race`). Після 60 с хост вимикається, і всі запити інстансу падають з `No host available` | Логи таймаутів і `No host available` поруч у часі |
| 5 | Погодинний backfill з `DELETE` до 6 місяців | Тривалість і обсяг `DELETE` у логах і моніторингу |
| 6 | Щоденний `DROP SERIES` по всіх measurements | Латентність і CPU близько 01:00 UTC |
| 7 | Bulk writer повторює batch без ліміту → write storm після збою | Логи `BULK ERROR`, heap інстансу |
| 8 | Основний writer відкидає batch при будь-якій помилці | Логи `Couldn't write N points` |

Окремо від навантаження є проблеми коректності, які варто передати команді: `Job.find` ігнорує limit і sort, backfill понад 6 місяців нічого не перераховує, а `runProcessLargePackets` сортує newest-first, хоча коментар каже oldest-first.

## Робочі сценарії для уроків

| Сценарій | Як використати в курсі |
| --- | --- |
| Графік одного сенсора | Вибір за `modelId` і часом. Порівняти вартість короткого raw-графіка та довгого графіка з агрегатів. |
| Графік кількох сенсорів | Перевірити, чи потрібна окрема лінія на сенсор або середнє по всіх точках. Це два різні результати, коли сенсори репортять з різною частотою. |
| Агрегація Node.js cron | Виміряти вікно читання, частоту, повторне читання, паралельність і поведінку запису summary при повторі job. |
| Перевірка алертів | З’ясувати, чи перевірка прив’язана до refresh і чи робиться окремий запит на кожен сенсор. Порівняти найсвіжіший стан із перевіркою всіх подій у вікні, якщо цього вимагає правило. |
| Огляд компанії з десятками сенсорів | Можливий сценарій з приблизно 50 сенсорами. Порахувати показник на сенсор і ранжувати приблизно до top‑10 у бекенді. Широка аналітика по всьому парку не є типовим навантаженням за описом. |

## Розібраний запит wear

Редагований приклад дашборда обирає два `modelId`, обмежує час змінними дашборда, робить 53 виклики `MEAN()` і групує тільки через `GROUP BY time(...) FILL(null)`. Такий запит дає спільне середнє за точками обох сенсорів у кожному вікні. Для окремих ліній групуємо також за `modelId`. Фактичні значення `modelId` і точне розташування цього запиту в production-додатку для вправ не потрібні. [InfluxQL `GROUP BY`](https://docs.influxdata.com/enterprise_influxdb/v1/query_language/explore-data/#the-group-by-clause)

Скорочена форма запиту зі зразка з двома полями та псевдонімами сенсорів. Кваліфікатор `dev.autogen.wear` походить із тексту запиту. Скриншот схеми `wear` і вимір cardinality стосуються `prod`.

```sql
SELECT MEAN("board_temp"), MEAN("cum_fe_mass")
FROM "dev"."autogen"."wear"
WHERE time > :dashboardTime: AND time < :upperDashboardTime:
  AND ("modelId" = 'sensor-a' OR "modelId" = 'sensor-b')
GROUP BY time(:interval:) FILL(null)
```

Для `wear` варто звірити, які fields читає графік і що має означати агрегат для кожного з них. Середня температура, остання версія firmware, рівень накопиченого лічильника і його приріст за період відповідають різним запитам. Raw і cumulative можуть лишатися в одному measurement, якщо приходять разом і мають спільний життєвий цикл. Рішення про поділ залежить від потреб читання, частоти запису та retention. [InfluxQL `MEAN()` та `LAST()`](https://docs.influxdata.com/enterprise_influxdb/v1/query_language/functions/)

## Запити дашборда для oil quality

Усі три зразки читають `prod.autogen`, обмежують час змінними дашборда й агрегують через `GROUP BY time(:interval:) FILL(null)`. Це запити на читання для графіків. З них видно форму даних, які читає дашборд. Джерело й графік запису похідних даних звіримо з кодом.

| Measurement | Поля в запиті | Зріз і питання для розбору |
| --- | --- | --- |
| `oil_quality` | 58 викликів `MEAN()`. Сирі сигнали, зокрема `qw_s0mag_raw` і `qw_s1phase_raw`, температура, вологість та інші показники. | Один сенсор. Які поля потрібні для конкретного графіка і який часовий діапазон він зазвичай читає? |
| `oil_quality_daily` | 15 викликів `MEAN()`. Денні середні, розкид, різниці та відсоткові зміни, зокрема `daily_avg_bulk` і `daily_spread_hf`. | Два сенсори. `GROUP BY time(...)` створює спільний результат за їхніми точками. Коли в одне вікно потрапляють кілька денних середніх, `MEAN()` дає середнє денних значень. Для відновлення середнього за raw samples потрібні відповідні `sum` і `count` або середні з вагами. |
| `oil_quality_health_indicator` | 45 викликів `MEAN()`. Health metrics, стани, індикатори заміни оливи й точки від останньої заміни. | Той самий сенсор, що й у першому запиті. Якщо `current_state_*` і `oil_changed_*` є кодами або прапорцями, вибір агрегації залежить від того, чи графік показує останній стан, факт події за вікно або частку точок з прапорцем. |

Скорочені шаблони зберігають форму фільтра й агрегації. Ідентифікатори сенсорів замінено псевдонімами.

```sql
SELECT MEAN("qw_s0mag_raw"), MEAN("relative_humidity")
FROM "prod"."autogen"."oil_quality"
WHERE time > :dashboardTime: AND time < :upperDashboardTime:
  AND "modelId" = 'sensor-a'
GROUP BY time(:interval:) FILL(null)

SELECT MEAN("daily_avg_bulk"), MEAN("daily_spread_hf")
FROM "prod"."autogen"."oil_quality_daily"
WHERE time > :dashboardTime: AND time < :upperDashboardTime:
  AND ("modelId" = 'sensor-b' OR "modelId" = 'sensor-c')
GROUP BY time(:interval:) FILL(null)

SELECT MEAN("health_indicator_combined"), MEAN("oil_changed_bulk")
FROM "prod"."autogen"."oil_quality_health_indicator"
WHERE time > :dashboardTime: AND time < :upperDashboardTime:
  AND "modelId" = 'sensor-a'
GROUP BY time(:interval:) FILL(null)
```

Для порівняння графіків перевіряємо сенс показника й обсяг читання окремо. Короткий графік може читати кілька потрібних raw fields. Для довгого графіка перевіряємо, чи є в daily measurement показник, який відповідає тому самому бізнес-питанню. Для окремих ліній денних показників додаємо групування за `modelId`. [InfluxQL `GROUP BY`](https://docs.influxdata.com/enterprise_influxdb/v1/query_language/explore-data/#the-group-by-clause), [InfluxQL `MEAN()` та `LAST()`](https://docs.influxdata.com/enterprise_influxdb/v1/query_language/functions/)

## Кейс для уроку з тегом компанії

Компанія має 50 сенсорів і хоче графік їхніх денних health metrics та top‑10. Порівнюємо два підходи. У першому бекенд бере список `modelId` з каталогу сенсорів і передає його у запит. У другому writer додає `companyId` як tag, і InfluxQL може відбирати series за компанією.

Якщо кожен `modelId` має одне значення `companyId` у всьому записаному періоді й усі точки мають однакову схему тегів, цей вимір не множить cardinality на кількість компаній. Під час переходу старий tag set і tag set з `companyId` можуть співіснувати. Передавання сенсора іншій компанії теж може створити новий series key. Для вибору підходу з’ясовуємо, кому належить історія сенсора після передачі та чи є часті запити, які виграють від індексованого фільтра за компанією. [Теги та series keys в Enterprise v1](https://docs.influxdata.com/enterprise_influxdb/v1/concepts/schema_and_data_layout/)

## Наступні дані для карти

- [x] ~~Код Node.js cron~~ — розібрано 2026-11-25, див. [фонові задачі](#хто-читає-й-пише-фонові-задачі-з-коду-2026-11-25). `oil_quality_daily` пишуть щоденний `calculateQxDailyValues` (03:00 UTC) і `processOilChangeQueue`. `oil_quality_health_indicator` пишуть щохвилинний `runPostImportCalculations` і погодинний backfill.
- [x] ~~Alert lookup~~ — `checkAllDevicesForAlarms`: вікно 2 год на точку, `MAX/MIN/SPREAD` по одному полю, запуск щохвилини.
- [x] ~~Розподіл writers між RP~~ — у `short_lived` пише лише `bad_data`, усе інше йде в `autogen`.
- [ ] Звірити точну cardinality (53 000 чи 56 901) зі збереженим результатом.
- [ ] Підтвердити живе значення `RUN_CRONS` у Secret Manager і фактичну кількість інстансів prod у години пік.
- [ ] Метрики: що відбувається близько 01:00 UTC (`DROP SERIES`). Причину майже погодинних піків свідомо не шукаємо: джерел багато (див. [джерела навантаження](#джерела-навантаження-і-клієнти)).
- [ ] Частота telemetry за типом сенсора (`secondsBetweenPts`) і кількість активних QX та wear пристроїв — для оцінки обсягу smoothing.
- [ ] Типові діапазони графіків у Oil App і хто ще користується Chronograf-дашбордами (`:dashboardTime:`).
- [ ] Семантика й поведінка скидання `cum_*`.
- [ ] Чи скасовує Cloud 1 запит, коли клієнт обриває HTTP-з'єднання (перевірити на локальному стенді).

Практичні вправи залишають первинну форму запиту, зв’язок `measurement → modelId`, кількість полів і типовий час читання. Для зразків використовуємо псевдоніми сенсорів, без production credentials, адрес підключення й клієнтських ідентифікаторів.
