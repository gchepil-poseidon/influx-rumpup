# Урок 7. Вартість запиту і захист кластера

Платформа: InfluxDB Cloud 1 / Enterprise v1 (TSM + InfluxQL). Приклади взято з [карти системи](../notes/system-map.md).

## Ментальна модель

Запит проходить три етапи ([EXPLAIN ANALYZE](https://docs.influxdata.com/enterprise_influxdb/v1/query_language/spec/#explain-analyze)):

1. **Планування.** `WHERE time` визначає shard groups (у `prod.autogen` одна група = 7 діб). У кожному шарді індекс шукає series за tag-фільтром, і для кожної series × field створюється cursor. Вартість ≈ shards × series × fields. В Enterprise шарди можуть бути на інших data nodes.
2. **Виконання.** Cursors розпаковують TSM blocks (до 1000 значень одного field). Вартість ≈ точки в діапазоні × fields. Агрегація йде після розпакування.
3. **Відповідь.** Серіалізація й мережа. Лише цю частину зменшують `LIMIT` і `GROUP BY time()`.

| Що зменшує | Планування | Читання blocks | Відповідь |
|---|---|---|---|
| Коротший `WHERE time` | так | так | так |
| Точний tag-фільтр | так | так | так |
| Менше fields | так | так | так |
| `GROUP BY time()` | ні | ні | так |
| `LIMIT` | ні | інколи (raw `ORDER BY time DESC`) | так |
| Daily замість raw | — | у сотні разів | так |

TSM-індекс блоку знає min/max time, тож шард без даних series у діапазоні відповідає швидко, але його все одно треба відкрити й опитати.

## Кейси з production

### `getLatestPriorToTs`

`ORDER BY time DESC LIMIT 1`, lookback 19 год → 30 діб → 91 доба, три послідовні запити. Для пристрою з прогалиною 40 діб: три запити і ≈ 1–2 + 5–6 + 14 ≈ 21 відкритий shard group. Основна робота — планування й порожні шарди, розпаковується приблизно один блок.

Уточнення з уроку: `runPostImportCalculations` бере пристрої, які щойно прислали дані, тому зазвичай спрацьовує лише 19-годинний запит. Довгі гілки — last resort: перша точка після простою, backfill або FTP-імпорт історії. Ризик створює кратність дешевого запиту (fields × пристрої × хвилини × 4–8 інстансів), а не глибина lookback. Лог гілки 91 доби в коді має рівень `debug`, тому його відсутність у Cloud Logging нічого не доводить, поки не перевірено рівень winston у prod.

### Графік Oil App за рік

Синтетично: 1 сенсор, `oil_quality`, 58 fields, 1 точка на 5 хв.

- Raw: 288 × 365 ≈ 105 000 значень на field × 58 ≈ **6,1 млн значень ≈ 6 100 blocks**, ≈ 53 shard groups. Клієнт отримує ~1000 точок на series.
- `oil_quality_daily`: 365 × 15 ≈ **5 500 значень**, приблизно в 1 100 разів менше.
- Перед підміною перевірити покриття (15 проти 58 полів) і сенс (середнє середніх правильне лише для рівних діб, інакше потрібні `sum` + `count`).

Такі запити рідкісні, але небезпечні хвостом: refresh, 60-секундний таймаут node-influx і зовнішні скрипти через API-сервіс. Захист продукту: raw до N діб, довші діапазони йдуть через daily, явний raw за рік — асинхронним експортом.

## Ранжування ризиків (A–D)

| # | Місце | Проблема | Швидкий фікс |
|---|---|---|---|
| B | `runPostImportCalculations`, smoothing `MOVING_AVERAGE ... INTO` | Щохвилини перезаписує ~31 год кожного QX-пристрою: точка переписується ~1 860 разів, постійне навантаження на write і compaction | Перераховувати лише нові точки плюс хвіст, потрібний вікну moving average |
| D | `checkAllDevicesForAlarms`, атрибути з query function | Вкладені `MOVING_AVERAGE(NON_NEGATIVE_DERIVATIVE(CUMULATIVE_SUM(...)))` за 14 діб щохвилини, щоб перевірити 2 год (~170× надлишкового читання) | Один раз записувати похідний ряд у processed measurement, аларм читає 2 год |
| A | `checkAllDevicesForAlarms`, `MAX/MIN/SPREAD` за 2 год | Форма запиту добра, але ~600 одночасних запитів на інстанс × 4–8 інстансів без lock | Lock або leader для cron, семафор на кількість одночасних Influx-запитів, один запит на пристрій і field замість запиту на точку |
| C | `getOilSampleTypes`, `SHOW TAG VALUES WITH KEY` без `FROM`/`WHERE` | Обходить індекс усіх ~470 тижневих shard groups і всіх measurements | Додати `FROM` і `WHERE time > now() - 30d` або кешувати значення |

Висновок: у цій системі кластер найімовірніше навантажують **часті дешеві запити без координації**, а не рідкісні важкі.

## Shard group duration 30 днів замість 7

- Дозволено: для безстрокової RP документація рекомендує 52 тижні або більше. Правило: shard group ≥ 2× найчастішого діапазону запитів ([рекомендації](https://docs.influxdata.com/enterprise_influxdb/v1/concepts/schema_and_data_layout/#shard-group-duration-recommendations)).
- `ALTER RETENTION POLICY "autogen" ON "prod" SHARD DURATION 30d` діє лише на нові shard groups. Історія лишається тижневою.
- Ризик для нас: *updates and deletes to larger shards incur higher costs for recompaction*. У нас щохвилинний smoothing `INTO`, погодинний backfill з `DELETE`, FTP-імпорт історії, `DROP SERIES`.
- Читання 30-денного графіка виграє лише в плануванні (5–6 шардів → 1–2), кількість blocks та сама.
- Рішення: не перший важіль. Спочатку виміряти локально через `EXPLAIN`, виправити форми запитів, потім — зміна з погодженням, через підтримку Cloud 1 і з планом відкату (`ALTER` назад на `168h`).

## Захисти на сервері

Налаштування Enterprise v1 у секції `[cluster]`, за замовчуванням вимкнені (`0`) ([query management](https://docs.influxdata.com/enterprise_influxdb/v1/troubleshooting/query_management/influxql_query_management/)):

- `query-timeout` вбиває запит після заданого часу;
- `max-concurrent-queries` відхиляє нові запити понад ліміт;
- `max-select-point` / `max-select-series` / `max-select-buckets` обмежують обсяг одного `SELECT`;
- `log-queries-after` пише `Detected slow query` з `qid` для `KILL QUERY`.

Фактичні значення Cloud 1 можна побачити через `SHOW DIAGNOSTICS` (секція `config-coordinator`) або дізнатися в підтримки.

## Діагностика без здогадок

- Logs Explorer шукає тексти логів, а не назви функцій. «У логах немає» ≠ «не відбувається»: спочатку перевірити рівень логера, sampling і версію деплою.
- Корисні рядки в логах бекенду: `No host available`, `Couldn't write`, `BULK ERROR`, таймаути.
- Самі запити видно на боці Influx: `SHOW QUERIES` у пік і slow query log.

## Відкладена практика

Локальний стенд InfluxDB 1.8 у Docker, синтетичний сенсор із прогалиною 40 діб, вимірювання через `EXPLAIN` / `EXPLAIN ANALYZE`:

1. `getLatestPriorToTs`: 19 год → 30 → 91 доба;
2. рік raw `MEAN` 58 fields проти daily;
3. 30-денний запит при `SHARD DURATION 7d` і `30d`.
