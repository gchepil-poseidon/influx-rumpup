# Карта production InfluxDB для курсу

Станом на 2026-11-24. Ця карта фіксує схему й робочі сценарії бази `prod` для подальших уроків про InfluxDB Cloud 1 / Enterprise v1. Для вправ використовуємо редаговані зразки або локальні дані з тією самою структурою й характером навантаження.

## Схема й потік даних

Сенсор надсилає телеметрію → система записує raw points в InfluxDB → Node.js cron читає raw, групує й записує summary → дашборд читає телеметрію та підсумки за сенсорами. У сценарії дашборда також є перевірка алертів за даними сенсорів. Точний механізм і частота перевірки ще відкриті.

| Елемент | Що відомо |
| --- | --- |
| База й retention policies | Production-база `prod` має `autogen` і `short_lived`. Фактичні параметри наведено нижче. Розподіл writers і measurements між policies ще відкритий. |
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

- [ ] **На робочому комп’ютері** відкрити доступ до коду Node.js cron або надати знеособлений фрагмент. Перевірити, хто й за яким розкладом записує `oil_quality_daily` та `oil_quality_health_indicator`, який raw-діапазон читається і як обробляються повторні запуски.

1. Один типовий Node.js cron, який читає raw і пише summary. Потрібні редагований InfluxQL, фактичне вікно читання за запуск, частота запуску, приблизна кількість сенсорів та measurement, куди потрапляє результат. Це дасть перший кейс для уроку про вартість запитів.
2. Один типовий alert lookup. Які fields читає, за який час, для скількох сенсорів і що запускає перевірку.
3. Коли ці дані вже є під рукою, приблизна частота надсилання telemetry за типом сенсора, типові діапазони дашборда, розподіл writers між `autogen` і `short_lived` та відомі симптоми під час піків. Для уточнення карти достатньо коду, конфігурації або наявних метрик.

Практичні вправи залишають первинну форму запиту, зв’язок `measurement → modelId`, кількість полів і типовий час читання. Для зразків використовуємо псевдоніми сенсорів, без production credentials, адрес підключення й клієнтських ідентифікаторів.
