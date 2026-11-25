# Урок 4. Запис: line protocol і write path

Платформа: InfluxDB Cloud 1 / Enterprise v1 (TSM, `/write` + line protocol). Приклади — з [карти системи](../notes/system-map.md#write-path-три-буфери-в-памяті-процесу). Урок теоретичний: практику користувач робить у реальному проєкті.

## Ментальна модель

```
node-influx buffer ──POST /write?db=prod&precision=n──▶ data node (координатор)
   1. парсинг line protocol
   2. timestamp → shard group (7 діб у prod.autogen)
   3. shard → 2 власники (replicaN=2), недоступний → hinted handoff
   4. на кожному власнику: WAL + cache у RAM → TSM-файл → compaction
```

- Один HTTP-запит ≠ один запис на диск. Batch «свіжих» даних лягає в 1 shard group, історія за рік — у ~53. У кожному зачепленому шарді піднімається свій cache, який скидається в TSM після 25 MB або 10 хв без записів ([FAQ про історичні дані](https://docs.influxdata.com/enterprise_influxdb/v1/troubleshooting/frequently-asked-questions/#what-are-the-configuration-recommendations-and-schema-guidelines-for-writing-sparse-historical-data), [backfill і shard duration](https://docs.influxdata.com/enterprise_influxdb/v1/concepts/schema_and_data_layout/#shard-group-duration-for-backfilling)).
- **Одиниця batch — рядок (point).** Fields множать байти, але не кількість рядків. 10 000 points × 40 fields ≈ 10,7 MB без gzip.
- Ліміти: `max-body-size` 25 MB за замовчуванням → `413`; рекомендований batch ~5 000 рядків (орієнтир ефективності, не ліміт); кластерний `write-timeout` 10 с ([max-body-size](https://docs.influxdata.com/enterprise_influxdb/v1/administration/configure/config-data-nodes/#max-body-size), [write-timeout](https://docs.influxdata.com/enterprise_influxdb/v1/administration/configure/config-data-nodes/#write-timeout)).

## Відповіді сервера і що з ними робити

| Відповідь | Що реально сталося | Правильна реакція |
|---|---|---|
| `204` | усе записано | — |
| `400` partial write / `field type conflict ... dropped=N` | валідні points **записані**, відкинуто лише погані | не повторювати, залогувати відкинуті рядки |
| `5xx`, таймаут | частина могла записатися | повторити обмежено, з backoff і jitter |

Повтор безпечний: point = measurement + tag set + timestamp, тож повтор з явними timestamp перезаписує ті самі points. Ціна повтору — навантаження, а не дублікати. Клієнтський таймаут не відкочує запис на сервері ([HTTP response summary](https://docs.influxdata.com/enterprise_influxdb/v1/guides/write_data/#http-response-summary), [hinted handoff](https://docs.influxdata.com/enterprise_influxdb/v1/concepts/clustering/#hinted-handoff)).

## Наш бекенд

| Буфер | Поведінка | Мінімальна зміна |
|---|---|---|
| `influxBatchWriter` / `…Rp` | будь-яка помилка → batch відкинуто | `5xx`/таймаут: 2–3 повтори з backoff; `400`: лог відкинутих рядків |
| `influxBatchWriterBulk` | `400` → відкинуто; інше → повтор кожні 5 с без ліміту, буфер росте в heap | ліміт повторів, експоненційний backoff + jitter, межа буфера → пауза імпорту (backpressure) |
| обидва | немає таймера flush, хвіст губиться при scale-down; 4–8 незалежних буферів | таймер flush і flush на `SIGTERM` |

Також: `maxRetries: 4` не діє з одним хостом. Після таймауту хост вимкнено на backoff, і **всі** запити інстансу (алерти, cron) отримують `No host available`. Без gzip; `precision=n` — виграш від грубішої precision малий (~1% payload), не перший важіль.

**Перевірено метриками (2026-11-25):** `Couldn't write` у Cloud Logging за місяць — 3 випадки, ~1 000 points на весь потік. Втрати основного буфера реальні в коді, але на практиці рідкісні → низький пріоритет.

**Обмеження з боку FTP (за словами користувача):** CSV не більше доби і 2–3 MB. У коді бекенду цього обмеження немає, воно тримається на FTP-джерелах.

## Timestamp і типи

| Помилка | Наслідок |
|---|---|
| ms передано як ns | 1970 рік, новий shard group у безстроковій `autogen`, дашборди дані не бачать |
| ns з `precision=ms` | вихід за 2262 → `400` |
| годинник пристрою в майбутньому | `futureWriteLimit = 0s` → shard group у майбутньому, `LAST()` показує «майбутнє» |

Тип field фіксується першим записом **у шарді**. Інший тип у тому самому шарді → `400 dropped=N`; у новому тижневому шарді може закріпитися інший тип. JS-число `5` іде як float, `5i` — лише зі схемою `INTEGER` у node-influx.

`/api/v2/write` у 1.8+ — compatibility endpoint (`bucket` = `db/rp`). Бекенду він не потрібен: node-influx пише у v1 `/write`.

## Smoothing `INTO` як write amplification

Синтетично: 1 point / 5 хв, вікно 31 год, 8 statements, 200 QX-пристроїв.

- 12 точок/год × 31 год = **372** точки × 8 = **2 976** field values на пристрій за тик;
- × 200 = **≈ 595 000** за хвилину ≈ **10 000/с**;
- сирий потік тих самих пристроїв: 200 × 40 / 300 с ≈ **27/с** → smoothing пише в **~370 разів** більше;
- на добу на пристрій 4,3 млн перезаписів проти 2 304 нових значень (~1/1 860).

Реальну кількість активних QX-пристроїв ще треба виміряти.

## Чекліст перед bulk write

1. Points = сенсори × точок на добу × доби (fields сюди **не** множити).
2. Batch = points / розмір batch; МБ ≈ points × байт на рядок.
3. Shard groups = діапазон / 7 діб (+1 на межі).
4. Нові series = нові комбінації measurement + tags. Fields дають field streams, а не series.
5. Що буде при таймауті: хто повторює, скільки разів, з якою паузою, де росте черга.
