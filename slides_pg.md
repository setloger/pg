---
marp: true
theme: default
paginate: true
style: |
  section {
    font-family: 'JetBrains Mono', 'Fira Code', monospace, sans-serif;
    font-size: 18px;
    background: #ffffff;
    color: #1a1a2e;
    padding: 40px 50px;
  }
  h1 {
    font-size: 32px;
    color: #1a1a2e;
    border-bottom: 3px solid #336791;
    padding-bottom: 10px;
    margin-bottom: 20px;
  }
  h2 {
    font-size: 26px;
    color: #336791;
    margin-bottom: 14px;
  }
  h3 {
    font-size: 20px;
    color: #555;
    margin-bottom: 10px;
  }
  code {
    background: #f4f6f9;
    border-radius: 4px;
    padding: 1px 5px;
    font-size: 15px;
    color: #336791;
  }
  pre {
    background: #1e2a3a;
    color: #c9d1d9;
    border-radius: 8px;
    padding: 16px;
    font-size: 13px;
    line-height: 1.5;
    overflow: hidden;
  }
  pre code {
    background: transparent;
    color: #c9d1d9;
    font-size: 13px;
  }
  table {
    font-size: 14px;
    border-collapse: collapse;
    width: 100%;
  }
  th {
    background: #336791;
    color: #fff;
    padding: 6px 10px;
    text-align: left;
  }
  td {
    border: 1px solid #dde;
    padding: 5px 10px;
  }
  tr:nth-child(even) td {
    background: #f0f4f8;
  }
  .tag {
    display: inline-block;
    background: #336791;
    color: #fff;
    border-radius: 4px;
    padding: 2px 10px;
    font-size: 13px;
    margin-bottom: 8px;
  }
  .warn {
    background: #fff3cd;
    border-left: 4px solid #f0ad4e;
    padding: 8px 14px;
    border-radius: 4px;
    font-size: 15px;
    margin: 10px 0;
  }
  .info {
    background: #e8f4fd;
    border-left: 4px solid #336791;
    padding: 8px 14px;
    border-radius: 4px;
    font-size: 15px;
    margin: 10px 0;
  }
  section.title-slide {
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: flex-start;
    background: linear-gradient(135deg, #1a1a2e 0%, #16213e 60%, #0f3460 100%);
    color: #fff;
    padding: 60px 70px;
  }
  section.title-slide h1 {
    font-size: 38px;
    color: #fff;
    border-color: #336791;
    line-height: 1.3;
    margin-bottom: 30px;
  }
  section.title-slide h2 {
    font-size: 22px;
    color: #a8c5e0;
    margin-bottom: 10px;
  }
  section.title-slide .subtitle {
    font-size: 16px;
    color: #7fa8c9;
  }
  section.section-header {
    background: #336791;
    color: #fff;
    display: flex;
    flex-direction: column;
    justify-content: center;
  }
  section.section-header h1 {
    font-size: 36px;
    color: #fff;
    border-color: rgba(255,255,255,0.4);
  }
  section.section-header h2 {
    color: #c9e2f5;
    font-size: 20px;
  }
---

<!-- _class: title-slide -->

# Архитектура и эксплуатация PostgreSQL 16

<div class="subtitle">
PostgreSQL → WAL → Patroni → etcd
</div>

---

<!-- Слайд 2: Процессная модель -->

# Process-per-connection

**Каждое соединение = отдельный OS-процесс (~5–10 МБ RAM)**

```
[Клиент 1] ──┐
[Клиент 2] ──┤──► [pgBouncer / Odyssey] ──► [postmaster]
[Клиент N] ──┘                                    │
                                     ┌─────────────┼─────────────┐
                                [backend 1]  [backend 2]  [backend N]
                                     │              │              │
                                  [shared_buffers + OS Page Cache]
```

<div class="info">
Без пулера: <code>max_connections=500</code> → 2–5 ГБ RAM только на коннекты
</div>

**Решение:** pgBouncer / Odyssey держит постоянный пул прогретых процессов

---

<!-- Слайд 3: Управление инстансом -->

# Управление инстансом и безопасность

<div style="display: grid; grid-template-columns: 1fr 1fr; gap: 20px;">

<div>

**Управление кластером (Debian)**

```bash
# Инициализация нового кластера
pg_createcluster 16 main

# Старт / стоп / статус
pg_ctlcluster 16 main start
pg_ctlcluster 16 main status

# НИКОГДА от root:
sudo -u postgres pg_ctlcluster 16 main start
```

</div>

<div>

**Проверка прав на PGDATA**

```bash
stat /var/lib/postgresql/16/main
# Owner: postgres — OK ✅
# Owner: root     — СЛОМАЕТ при рестарте ❌
```

<div class="warn">
PGDATA никогда не должен принадлежать root
</div>

</div>
</div>

---

<!-- Слайд 4: Shared Buffers vs OS Cache -->
<!-- _class: section-header -->

# Блок 2: Архитектура памяти



---

# Shared Buffers vs OS Cache

<div style="display: grid; grid-template-columns: 1fr 1fr; gap: 24px;">

<div>

**Двухуровневый кэш — «жизнь страницы»**

```
Запрос страницы
      │
      ▼
[shared_buffers] ──── hit? ──► Возврат (быстро)
      │ miss
      ▼
[OS Page Cache]  ──── hit? ──► → shared_buffers
      │ miss
      ▼
    [Диск]  ──► read() → OS Cache → shared_buffers
```

</div>

<div>

**Рекомендации по размеру**

| RAM сервера | shared_buffers | OS Cache |
|---|---|---|
| 16 ГБ | 4 ГБ | 12 ГБ |
| 64 ГБ | 16 ГБ | 48 ГБ |
| 256 ГБ | 64 ГБ | 192 ГБ |

<div class="info">
Правило: ~25% RAM → shared_buffers
</div>

</div>
</div>

---

<!-- Слайд 5: work_mem и temp_buffers -->

# Локальная память процессов: work_mem

<div class="warn">
<b>Зона риска OOM:</b> work_mem выделяется на каждую операцию внутри запроса
</div>

```
work_mem = 64 МБ
Запрос: 3 сортировки × 64 МБ = 192 МБ на 1 запрос

200 активных коннектов × 192 МБ = 37 ГБ
                                    ↑
                              OOM Killer 💀
```

| Параметр | Назначение | Безопасный диапазон |
|---|---|---|
| `work_mem` | Сортировка, Hash Join | 4–64 МБ |
| `maintenance_work_mem` | VACUUM, CREATE INDEX | 256 МБ – 2 ГБ |
| `temp_buffers` | Временные таблицы | 8–64 МБ |

---

<!-- Слайд 6: Уровни конфигурации -->

# Управление конфигурацией — 4 уровня приоритета

```
┌──────────────────────────────────────────────────────────────────┐
│  УРОВЕНЬ 4 · SESSION                                             │
│  SET work_mem = '256MB';   — действует только в текущей сессии   │
├──────────────────────────────────────────────────────────────────┤
│  УРОВЕНЬ 3 · ROLE / USER                                         │
│  ALTER ROLE analyst SET work_mem = '128MB';                      │
├──────────────────────────────────────────────────────────────────┤
│  УРОВЕНЬ 2 · DATABASE                                            │
│  ALTER DATABASE analytics SET work_mem = '64MB';                 │
├──────────────────────────────────────────────────────────────────┤
│  УРОВЕНЬ 1 · GLOBAL  (postgresql.conf / ALTER SYSTEM)            │
│  ALTER SYSTEM SET work_mem = '32MB'; → SELECT pg_reload_conf();  │
└──────────────────────────────────────────────────────────────────┘
```

| Требует RESTART | Требует RELOAD |
|---|---|
| `shared_buffers`, `max_connections`, `wal_level` | `work_mem`, `log_min_duration`, `autovacuum` |

```sql
SELECT name FROM pg_settings WHERE pending_restart = true;
```

---

<!-- Слайд 7: MVCC -->
<!-- _class: section-header -->

# Блок 3: MVCC и изоляция транзакций



---

# Многоверсионность (MVCC)

**Читающие не блокируют пишущих. UPDATE = INSERT новой версии + логическое удаление**

```
Таблица: users
┌────────────────────────────────────────────────────┐
│ tuple v1 │ xmin=100, xmax=150 │ id=1, name="Alice" │  ← мёртвый tuple
│ tuple v2 │ xmin=150, xmax=0   │ id=1, name="Bob"   │  ← живой tuple
└────────────────────────────────────────────────────┘

Снимок транзакции A (txid=160):
  Видит: xmin <= 160 AND (xmax=0 OR xmax > 160)
  → видит tuple v2 ✅   tuple v1 ❌
```

<div class="info">
<b>xmin</b> — транзакция, создавшая строку · <b>xmax</b> — транзакция, удалившая строку
</div>

---

<!-- Слайд 8: Долгие транзакции -->

# Долгие транзакции — эффект домино

```
t=0:00  BEGIN; — (забыли COMMIT, висит idle in transaction)
         └── удерживает снимок → Autovacuum не чистит dead tuples!

t=0:30  ALTER TABLE orders ADD COLUMN ...
         └── ждёт AccessExclusiveLock ← заблокирован tx выше

t=0:31  SELECT * FROM orders ...   ЖДЁТ 🔴
t=0:31  INSERT INTO orders ...     ЖДЁТ 🔴
t=0:35  ... +200 запросов в очереди ЖДЁТ 🔴

t=0:37  Connection Pool исчерпан → HTTP 503 ❌
```

**Мониторинг и устранение:**

```sql
SELECT pid, now() - xact_start AS age, state, query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
  AND now() - xact_start > interval '30 seconds'
ORDER BY age DESC;

SELECT pg_terminate_backend(<pid>);
```

---

<!-- Слайд 9: Bloat -->
<!-- _class: section-header -->

# Блок 4: Bloat и Autovacuum



---

# Разрастание (Bloat) таблиц и индексов

**После серии UPDATE без VACUUM:**

```
┌─────────────────────────────────────────────────────────┐
│ 💀dead │ 💀dead │ ✅live │ 💀dead │ 💀dead │ ✅live │ ...│
└─────────────────────────────────────────────────────────┘
  Размер файла таблицы: 10 ГБ
  Живых данных:          2 ГБ  ← читаем 10 ГБ ради 2 ГБ данных
```

**Диагностика bloat:**

```sql
SELECT relname,
       pg_size_pretty(pg_total_relation_size(oid)) AS total_size,
       n_dead_tup,
       n_live_tup,
       round(n_dead_tup * 100.0 / nullif(n_live_tup + n_dead_tup, 0), 1) AS dead_pct
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC
LIMIT 10;
```

---

# Анатомия VACUUM и Autovacuum

```
Autovacuum Launcher
      │
      ▼  Порог = autovacuum_vacuum_threshold (50) + scale_factor (20%) × n_live_tup
  [VACUUM Worker]
      ├── Очищает dead tuples → освобождает место ВНУТРИ файлов
      ├── Обновляет FSM (Free Space Map)
      ├── Обновляет VM (Visibility Map)  → ускоряет Index-Only Scan
      └── Замораживает старые XID        → предотвращает XID Wraparound
```

| | VACUUM | VACUUM FULL |
|---|---|---|
| Блокировка | Нет | Exclusive Lock |
| Возврат места ОС | ❌ | ✅ |
| Нагрузка на диск | Умеренная | Высокая (копирует таблицу) |
| В рабочее время | ✅ | ❌ |

<div class="warn">
VACUUM FULL в рабочее время — табу в высоконагруженных системах
</div>

---

<!-- Слайд 11: WAL и фоновые процессы -->
<!-- _class: section-header -->

# Блок 5: Надёжность данных и WAL



---

# Жизненный цикл записи и фоновые процессы

```
COMMIT транзакции
      │
      ▼
[WAL Buffer] ──► walwriter ──► [WAL файл на диске] ✅ commit confirmed
      │
[shared_buffers]  ←── dirty page (помечена, но ещё не на диске)
      │
      ├── bgwriter ──────────► постепенно → [OS Page Cache] → [Диск]
      │
      └── checkpointer ──────► все dirty pages → [Диск] (checkpoint)

Принцип: WAL на диске = транзакция зафиксирована. Данные — позже.
```

<div class="info">
<b>walwriter</b> — WAL buffer на диск · <b>bgwriter</b> — грязные страницы в ОС · <b>checkpointer</b> — контрольная точка
</div>

---

# Тюнинг контрольной точки и Crash Recovery

<div style="display: grid; grid-template-columns: 1fr 1fr; gap: 20px;">

<div>

**Конфигурация**

```ini
# postgresql.conf

# Макс. объём WAL до checkpoint
max_wal_size = 4GB
# дефолт 1GB — мало для продакшна

# «Размазать» checkpoint на 90%
checkpoint_completion_target = 0.9

# Мониторинг частоты:
# log_checkpoints = on
```

</div>

<div>

**Crash Recovery**

```
  Checkpoint         Crash       Restart
      │                │            │
 ─────●────────────────✕────────────►
                                    │
                               Replay WAL
                             от checkpoint
                             до crash point
                             → База консистентна ✅
```

</div>
</div>

---

# Защита от повреждения данных: Контрольные суммы

```bash
# Проверить статус (кластер должен быть остановлен)
pg_checksums --check -D /var/lib/postgresql/16/main

# Включить (однократно)
pg_checksums --enable -D /var/lib/postgresql/16/main

# Убедиться на работающем кластере:
SHOW data_checksums;
-- on ✅
```

**Цепочка обнаружения ошибок:**

```
Диск → Контроллер (silent corruption?) → Postgres читает страницу
     → проверяет CRC32 → ошибка в лог вместо тихой порчи данных
```

<div class="info">
Накладные расходы: ~1–2% CPU. Включайте всегда при инициализации новых кластеров.
</div>

---

<!-- Слайд 14: Иерархия структуры -->
<!-- _class: section-header -->

# Блок 6–7: Организация данных и диск



---

# Иерархия: Кластер, Базы, Схемы

```
PostgreSQL Cluster (PGDATA)
│
├── postgres      (служебная БД)
├── template0     (эталон — нельзя подключиться напрямую)
├── template1     ◄── CREATE DATABASE клонирует отсюда
│
└── myapp_db
    ├── public    (schema)
    │   ├── users   (table)
    │   └── orders  (table)
    └── analytics  (schema)
        └── events  (table)

На диске: $PGDATA/base/<OID базы>/<OID таблицы>
```

<div class="info">
<code>CREATE DATABASE</code> — не создаёт с нуля, а физически клонирует файлы из <code>template1</code>
</div>

---

# Файловая структура и слои данных

```
$PGDATA/base/16384/           ← OID базы данных
  ├── 2619                   ← основной файл таблицы (Mainfork)
  ├── 2619_fsm               ← Free Space Map (FSM)
  ├── 2619_vm                ← Visibility Map (VM)
  ├── 2619.1                 ← сегмент 2 (после 1 ГБ)
  └── 2619.2                 ← сегмент 3
```

```sql
SELECT relname, relfilenode, pg_size_pretty(pg_relation_size(oid))
FROM pg_class
WHERE relname = 'orders';
```

| Файл | Назначение |
|---|---|
| Mainfork | Страницы данных (8 КБ каждая) |
| FSM | Карта свободного места → быстрый INSERT |
| VM | Карта видимости → Index-Only Scan, VACUUM |

---

# Длинные строки и механизм TOAST

```
INSERT INTO docs (content) VALUES ('<100KB JSONB>');

Основная страница (8 КБ):
┌─────────────────────────────────────────┐
│ id=1 │ [TOAST pointer 24 bytes] │ ...   │
└────────────────────────────────────────┘
          │
          └──► TOAST таблица (pg_toast_XXXXX):
               ┌─────────────────────────────┐
               │ chunk_id=1, seq=0, data=...  │
               │ chunk_id=1, seq=1, data=...  │
               └─────────────────────────────┘

-- Смена алгоритма сжатия:
ALTER TABLE docs ALTER COLUMN content SET COMPRESSION lz4;
```

| | pglz (дефолт) | lz4 |
|---|---|---|
| Степень сжатия | Хорошая | Чуть хуже |
| Скорость | Медленная | В 3–5× быстрее |
| CPU нагрузка | Высокая | Низкая |

---

<!-- Слайд 17: pg_hba.conf -->
<!-- _class: section-header -->

# Блок 8: Безопасность и доступ



---

# Аутентификация: pg_hba.conf и pg_ident.conf

```
# /etc/postgresql/16/main/pg_hba.conf
# TYPE  DATABASE  USER      ADDRESS           METHOD

# Локальные скрипты — через Unix socket
local   all       postgres                    peer

# Маппинг системный пользователь → роль БД
local   all       all                         peer map=mymap

# Сетевые подключения — только scram-sha-256
host    myapp_db  app_user  10.0.0.0/8        scram-sha-256

# ЗАПРЕЩЕНО в продакшне:
# host  all  all  0.0.0.0/0  trust   ← без пароля
# host  all  all  0.0.0.0/0  md5     ← устаревший алгоритм

# Применить БЕЗ рестарта:
SELECT pg_reload_conf();
```

<div class="warn">
Правила читаются сверху вниз — применяется первое совпавшее. md5 уязвим с 2024 — только scram-sha-256.
</div>

---

<!-- Слайд 18: Observability -->
<!-- _class: section-header -->

# Блок 9: Продакшн-наблюдаемость



---

# Накопительная статистика и утилиты ОС

<div style="display: grid; grid-template-columns: 1fr 1fr; gap: 20px;">

<div>

**Мониторинг через ps (ОС)**

```bash
ps aux | grep postgres
# postgres: app_user mydb SELECT
# postgres: app_user mydb idle
# postgres: app_user mydb idle in transaction ⚠️
# postgres: autovacuum worker
```

`update_process_title = on` — бэкенды показывают текущий SQL

</div>

<div>

**Агрегат по pg_stat_activity**

```sql
SELECT state, count(*)
FROM pg_stat_activity
WHERE pid <> pg_backend_pid()
GROUP BY state
ORDER BY count DESC;
-- idle in transaction > 10 → алерт 🚨
```

</div>
</div>

---

# Профилирование запросов: pg_stat_statements

**Подключение расширения (требует рестарта!)**

```ini
# postgresql.conf
shared_preload_libraries = 'pg_stat_statements'
pg_stat_statements.track = all
```

```sql
-- После рестарта:
CREATE EXTENSION pg_stat_statements;
```

**Топ-5 самых дорогих запросов:**

```sql
SELECT
    round(total_exec_time::numeric, 2) AS total_ms,
    calls,
    round(mean_exec_time::numeric, 2)  AS avg_ms,
    round(stddev_exec_time::numeric, 2) AS stddev_ms,
    left(query, 80) AS query
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 5;
```

---

<!-- Слайд 20: Бэкапы -->
<!-- _class: section-header -->

# Блок 10: Бэкапы и Disaster Recovery



---

# Физические бэкапы vs Логические дампы

| | pg_dump (логический) | pg_basebackup (физический) |
|---|---|---|
| Скорость создания | Медленно (CPU) | Быстро (I/O) |
| Скорость восстановления | Очень медленно | Быстро |
| PITR | ❌ | ✅ (+ WAL archive) |
| Частичное восстановление | ✅ | ❌ |
| Применение | < 10 ГБ, dev/test | Продакшн > 100 ГБ |

**Таймлайн PITR:**

```
 Base Backup        WAL архив              Restore Point
     │               │││││││││               │
 ────●───────────────███████████─────────────►
   00:00                                   14:35:22

recovery.signal + restore_command = «воспроизвести WAL до 14:35:22»
recovery_target_time = '2024-01-15 14:35:22'
```

---

<!-- Слайд 21: Физическая репликация -->
<!-- _class: section-header -->

# Блок 11–12: Репликация



---

# Потоковая физическая репликация

```
[Master]                      [Standby (read-only)]
   │                                  │
   │  walsender ──── TCP ──► walreceiver
   │                                  │
  WAL                          применяет WAL
                                      │
                            Клиент читает ──► конфликт репликации?

wal_level = replica           hot_standby = on
```

| Параметр | Дефолт | Рекомендация |
|---|---|---|
| `max_standby_streaming_delay` | 30s | 300s (для аналитики) |
| `hot_standby_feedback` | off | on (снижает конфликты) |
| `wal_level` | replica | replica (мин. для HA) |

---

# Логическая репликация: Publisher / Subscriber

```
[Master DB — Publisher]              [Subscriber]
   │                                      │
  WAL → Logical Decoding             pglogical worker
         (pgoutput)                  применяет INSERT/UPDATE/DELETE
   │
  WAL Sender ──── logical slot ──► WAL Receiver
```

```sql
-- На Publisher:
CREATE PUBLICATION mypub FOR TABLE orders, users;

-- На Subscriber:
CREATE SUBSCRIPTION mysub
  CONNECTION 'host=master dbname=mydb'
  PUBLICATION mypub;
```

<div class="info">
Применение: частичная репликация, аналитические витрины, миграция между мажорными версиями без даунтайма
</div>

---

<!-- Слайд 23: Patroni -->
<!-- _class: section-header -->

# Блок 13: Архитектура Patroni (HA)



---

# Архитектура: Распределённый консенсус (DCS)

```
    ┌──────────────────────────────────────┐
    │            etcd cluster              │
    │   (распределённое хранилище правды)  │
    └──────┬───────────────────┬───────────┘
           │ heartbeat (TTL)   │ watch
           │                   │
 ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
 │  Patroni Agent   │  │  Patroni Agent   │  │  Patroni Agent   │
 │  [NODE 1]        │  │  [NODE 2]        │  │  [NODE 3]        │
 │  PostgreSQL      │  │  PostgreSQL      │  │  PostgreSQL      │
 │  MASTER ★        │  │  Replica         │  │  Replica         │
 └──────────────────┘  └──────────────────┘  └──────────────────┘

Leader Lock в etcd:
  /service/my-cluster/leader = "node1"  (TTL=30s)
  TTL истёк → node2/node3 захватывает лок → PROMOTE
```

<div class="warn">
Split-Brain защита: если мастер теряет связь с etcd — Patroni переводит его PostgreSQL в read-only
</div>

---

<!-- Слайд 24: Failover и HAProxy -->
<!-- _class: section-header -->

# Блок 14: Failover и маршрутизация



---

# Алгоритм Failover и маршрутизация трафика

<div style="display: grid; grid-template-columns: 1fr 1fr; gap: 20px;">

<div>

**Тайминги цикла**

```yaml
# patroni.yml
bootstrap:
  dcs:
    loop_wait: 10     # проверки (сек)
    ttl: 30           # Leader Lock (сек)
    retry_timeout: 10 # лимит к DCS (сек)

# Время Failover ≈ ttl (30 сек)
# Правило: ttl ≥ 2 × loop_wait
```

</div>

<div>

**Маршрутизация через HAProxy**

```
Клиент (запись) ──► HAProxy:5432
  /primary → HTTP 200 → NODE 1 ★
  /primary → HTTP 503 → NODE 2

Клиент (чтение) ──► HAProxy:5433
  /replica → HTTP 200 → NODE 2
  /replica → HTTP 200 → NODE 3

Failover:
  NODE 1 падает
  NODE 2 → Leader Lock → /primary 200
  HAProxy перестраивает за <1 сек
```

</div>
</div>

---

<!-- Слайд 25: Pause и synchronous_mode -->
<!-- _class: section-header -->

# Блок 15: Высокие нагрузки и обслуживание



---

# Ложный Failover и режим паузы

<div style="display: grid; grid-template-columns: 1fr 1fr; gap: 20px;">

<div>

**Сценарий ложного Failover**

```
Нагруженный CPU (VACUUM FULL / CREATE INDEX)
      │
      ▼
Patroni (Python) не получает CPU
      │
      ▼
Heartbeat в etcd не отправлен
      │
      ▼
TTL истёк → Failover
(PostgreSQL жив и здоров)
```

</div>

<div>

**Команды управления**

```bash
# Заморозить автоматику
patronictl -c /etc/patroni/patroni.yml pause

# Состояние кластера
patronictl -c /etc/patroni/patroni.yml list

# Принудительный switchover
patronictl -c /etc/patroni/patroni.yml switchover \
  --master node1 --candidate node2

# Вернуть автоматику
patronictl -c /etc/patroni/patroni.yml resume
```

</div>
</div>

<div class="info">
<code>synchronous_mode</code> — Patroni динамически управляет <code>synchronous_standby_names</code> → контроль RPO
</div>

---

<!-- Слайд 26: Bootstrap и топология -->
<!-- _class: section-header -->

# Блок 16: Динамическая топология



---

# Bootstrap новых узлов и каскадная репликация

```
Новый пустой узел запущен
      │
      ▼
Patroni Agent стартует
      │
      ├──► Подключается к etcd → находит лидера (node1)
      │
      ├──► Выбирает источник для клонирования:
      │         clonefrom=true  → реплика (снижаем нагрузку на мастер)
      │         clonefrom=false → мастер
      │
      ├──► pg_basebackup или pgBackRest restore
      │
      ├──► Генерирует recovery.signal + postgresql.auto.conf
      │
      └──► PostgreSQL стартует → накатывает WAL → реплика готова ✅
```

```yaml
# Тег для каскадной репликации:
tags:
  clonefrom: true
```

<div class="info">
Patroni берёт всю низкоуровневую работу с WAL и конфигами — кластер управляется декларативно через DCS
</div>

