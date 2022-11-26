# Нагрузочное тестирование и тюнинг PostgreSQL

## Цели:
1. Сделать нагрузочное тестирование PostgreSQL.
1. настроить параметры PostgreSQL для достижения максимальной производительности.

## Дополнительные материалы:
1. Утилита для настройки Postgre: https://pgtune.leopard.in.ua.

## Описание выполнения домашнего задания

### Шаг 1. Настроить кластер на максимальную производительность

> Игнорируем возможную потерю данных (надежность) с учетом настройки сервера. Задача обеспечить максимальную производительность.

### Результат

Определим известные настройки сервера **PostgreSQL**, которые позволяют достичь высокой производительности. В качестве машинны будем использовать развернутый в **Docker** контейнер на локальной машине с **MacOS**.

```sql
-- Определим настройки с помощью утилиты https://pgtune.leopard.in.ua

-- DB Version: 14
-- OS Type: mac
-- DB Type: oltp
-- Total Memory (RAM): 8 GB
-- CPUs num: 4
-- Data Storage: ssd

ALTER SYSTEM SET max_connections = '300';
ALTER SYSTEM SET shared_buffers = '2GB';
ALTER SYSTEM SET effective_cache_size = '6GB';
ALTER SYSTEM SET maintenance_work_mem = '512MB';
ALTER SYSTEM SET checkpoint_completion_target = '0.9';
ALTER SYSTEM SET wal_buffers = '16MB';
ALTER SYSTEM SET default_statistics_target = '100';
ALTER SYSTEM SET random_page_cost = '1.1';
ALTER SYSTEM SET work_mem = '3495kB';
ALTER SYSTEM SET min_wal_size = '2GB';
ALTER SYSTEM SET max_wal_size = '8GB';
ALTER SYSTEM SET max_worker_processes = '4';
ALTER SYSTEM SET max_parallel_workers_per_gather = '2';
ALTER SYSTEM SET max_parallel_workers = '4';
ALTER SYSTEM SET max_parallel_maintenance_workers = '2';
ALTER SYSTEM SET synchronous_commit = 'off';

select pg_reload_conf();
select * from pg_catalog.pg_file_settings

```

Получили такой список измененных настроек.

![Список настроек](/images/scr-dz12-01.png)

### Шаг 2. Нагрузить кластер через доступную утилиту

> Нагрузим наш кластер с новыми настройками утилитой **pgbench**. Используем стандартный тест.

### Результат

```bash
# Инициализация таблиц для работы тестов
postgres@64294a4ab873:/usr/lib/postgresql/14/bin$ pgbench -i postgres
# Подаем нагрузку, отписываем результат через каждые 30 секунд
postgres@64294a4ab873:/usr/lib/postgresql/14/bin$ pgbench -c8 -j4 -P 30 -T 600 postgres
```

Зафиксируем результат до настройки. Средняя производительность = 590 tps.

Зафиксируем результат после настройки. Средняя производительность = 4291 tps.

```bash
pgbench (14.4 (Debian 14.4-1.pgdg110+1))
starting vacuum...end.
progress: 30.0 s, 4865.4 tps, lat 1.641 ms stddev 2.601
progress: 60.0 s, 4247.4 tps, lat 1.882 ms stddev 2.268
progress: 90.0 s, 4177.8 tps, lat 1.914 ms stddev 2.382
progress: 120.0 s, 4339.1 tps, lat 1.844 ms stddev 2.165
progress: 150.0 s, 4311.0 tps, lat 1.855 ms stddev 2.182
progress: 180.0 s, 4349.2 tps, lat 1.839 ms stddev 2.286
progress: 210.0 s, 4296.4 tps, lat 1.861 ms stddev 2.179
progress: 240.0 s, 4310.4 tps, lat 1.855 ms stddev 2.284
progress: 270.0 s, 4359.6 tps, lat 1.834 ms stddev 2.139
progress: 300.0 s, 4375.4 tps, lat 1.828 ms stddev 2.170
progress: 330.0 s, 4257.7 tps, lat 1.878 ms stddev 2.392
progress: 360.0 s, 4250.8 tps, lat 1.881 ms stddev 2.229
progress: 390.0 s, 4281.3 tps, lat 1.867 ms stddev 2.242
progress: 420.0 s, 4348.3 tps, lat 1.838 ms stddev 2.119
progress: 450.0 s, 4370.2 tps, lat 1.831 ms stddev 2.187
progress: 480.0 s, 4325.4 tps, lat 1.849 ms stddev 2.227
progress: 510.0 s, 4256.6 tps, lat 1.879 ms stddev 2.176
progress: 540.0 s, 4171.4 tps, lat 1.917 ms stddev 2.267
progress: 570.0 s, 4161.5 tps, lat 1.922 ms stddev 2.263
progress: 600.0 s, 3773.7 tps, lat 2.119 ms stddev 2.608
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 4
duration: 600 s
number of transactions actually processed: 2574868
latency average = 1.863 ms
latency stddev = 2.273 ms
initial connection time = 51.487 ms
tps = 4291.684483 (without initial connection time)
```


### Шаг 3. Написать какого значения tps удалось достичь

> Удалось добиться значения в 4291 tps. Основной прирост дала настройка `synchronous_commit = 'off'`. С другими настройками можно немного поиграться, например, количество соединений + workmem. Можно так же на период проведения теста отключить avtoacuum ... но это решение чисто для получения результата.
