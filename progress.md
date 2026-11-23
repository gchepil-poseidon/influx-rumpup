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

## Урок 2. Логічна та фізична модель InfluxDB v1 — у процесі, 2026-11-22

Формат: інтерактивна сесія за матеріалом [`lessons/02-influxdb-v1-data-model.md`](lessons/02-influxdb-v1-data-model.md).

Статус: основну частину пройдено. Урок ще не завершено; залишилося приблизно 10–15 хвилин фінального закріплення.

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
| Фінальний приклад `device_health` | Частково: правильно 3 points і 2 indexed series; треба закріпити field streams і повторне використання series key |

### Що треба закріпити

- У кожної indexed tag-комбінації є окремий stream для кожного field key. У фінальному прикладі дві series × два fields = чотири field streams, незалежно від кількості values у кожному stream.
- Залежний tag не збільшує кількість комбінацій лише поки зв'язок справді однозначний у записаних series. `A1/us-east` і `A1/eu-west` — різні series keys.
- Індекс поводиться як множина унікальних keys: повернення до вже наявного `A1/us-east` не створює ще одну series; абсолютно новий `A1/us-south` створює.
- Не плутати кількість writes, logical points після duplicate merge, indexed series identities та field-level streams.

### Робочий приклад, використаний у розмові

Redacted масштаб: приблизно 5 000 значень `asset_id`, один point на п'ять хвилин; кількість assets зросла приблизно зі 100 у 2017 році до 5 000 у 2022 році та вийшла на плато. Це використано лише для закріплення різниці між point volume і series cardinality. Рішення щодо додавання `location_id` відкладено на прохання користувача; не продовжувати schema review під час завершення уроку 2.

### Звідки продовжити наступну сесію

1. Не повторювати весь урок.
2. Коротко відновити модель «індекс — множина унікальних series keys».
3. Повторити фінальний приклад із двома fields і попросити порахувати indexed series та field streams.
4. Дати одну коротку перевірку RP/shard group і одну перевірку duplicate/precision.
5. Пройти фінальні контрольні питання з розділу 19 уроку, акцентуючи три identity/key layers.
6. Якщо відповіді правильні, позначити урок 2 завершеним у цьому файлі й перейти за рекомендованим порядком до концептуального розбору вартості запитів з уроку 7.

Готова фраза для продовження: **«Продовжуємо урок 2 з checkpoint у `progress.md`; почни з фінального закріплення, не повторюй уже пройдене».**
