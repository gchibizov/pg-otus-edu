# Настройка autovacuum с учетом оптимальной производительности

## Цели:
1. Запустить нагрузочный тест pgbench с профилем нагрузки DWH.
1. Настроить параметры autovacuum для достижения максимального уровня устойчивой производительности.

## Описание выполнения домашнего задания

### Шаг 1. Создать инстанс Yandex Cloud, развернуть PostgreSQL, применить настройки из файла.

> В качестве платформы с облачными сервисами будем использовать Yandex Cloud. Подготовим инстанс по уже известным шагам. Опишем команды для установки PostgreSQL 14 и выполнения необходимых действий.

**Результат**

Создана виртуальная машина (**gchibizov-pg-20220808**):
![Виртуальная машина](/images/scr-dz08-01.png)

Команда для установки PostgreSQL:

```bash
sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-14
```

Настройки для сервера PostgreSQL:

```bash
# DB Version: 11
# OS Type: linux
# DB Type: dw
# Total Memory (RAM): 4 GB
# CPUs num: 1
# Data Storage: hdd

# Код для консоли psql, посмотреть размещение файла конфигурации
[psql] show config_file

max_connections = 40
shared_buffers = 1GB
effective_cache_size = 3GB
maintenance_work_mem = 512MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 500
random_page_cost = 4
effective_io_concurrency = 2
work_mem = 6553kB
min_wal_size = 4GB
max_wal_size = 16GB
```
![Настройки PostgreSQL на ВМ](/images/scr-dz08-02.png)

pie title Эффективность настройки автоваккума по профилям
    "Профиль 1" : 600
    "Профиль 2" : 400
    "Профиль 3" : 500






```bash
-- посмотрим виртуальный id транзакции
SELECT txid_current();
CREATE TABLE test(i int);
INSERT INTO test VALUES (10),(20),(30);

SELECT i, xmin,xmax,cmin,cmax,ctid FROM test;

SELECT *, xmin,xmax,cmin,cmax,ctid FROM actor;

-- посмотрим мертвые туплы
SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_TABLEs WHERE relname = 'test';

update test set i = 100 where i = 10;

https://postgrespro.ru/docs/postgrespro/12/pageinspect
CREATE EXTENSION pageinspect;
\dx+
SELECT lp as tuple, t_xmin, t_xmax, t_field3 as t_cid, t_ctid FROM heap_page_items(get_raw_page('test',0));
SELECT * FROM heap_page_items(get_raw_page('test',0)) \gx

-- попробуем изменить данные и откатить транзакцию и посмотреть


\echo :AUTOCOMMIT
\set AUTOCOMMIT OFF
commit;

insert into test values(50),(60),(70);

select * from test;

rollback;
commit;

-- объяснения про побитовую маску
-- https://habr.com/ru/company/postgrespro/blog/445820/


-----vacuum
vacuum verbose test;
SELECT lp as tuple, t_xmin, t_xmax, t_field3 as t_cid, t_ctid FROM heap_page_items(get_raw_page('test',0));
SELECT pg_relation_filepath('test');

vacuum full test;
SELECT pg_relation_filepath('test');

SELECT name, setting, context, short_desc FROM pg_settings WHERE name like 'vacuum%';

------Autovacuum
SELECT name, setting, context, short_desc FROM pg_settings WHERE name like 'autovacuum%';

SELECT * FROM pg_stat_activity WHERE query ~ 'autovacuum' \gx

select c.relname,
current_setting('autovacuum_vacuum_threshold') as av_base_thresh,
current_setting('autovacuum_vacuum_scale_factor') as av_scale_factor,
(current_setting('autovacuum_vacuum_threshold')::int +
(current_setting('autovacuum_vacuum_scale_factor')::float * c.reltuples)) as av_thresh,
s.n_dead_tup
from pg_stat_user_tables s join pg_class c ON s.relname = c.relname
where s.n_dead_tup > (current_setting('autovacuum_vacuum_threshold')::int
+ (current_setting('autovacuum_vacuum_scale_factor')::float * c.reltuples));


CREATE TABLE student(
  id serial,
  fio char(100)
) WITH (autovacuum_enabled = off);

CREATE INDEX idx_fio ON student(fio);

INSERT INTO student(fio) SELECT 'noname' FROM generate_series(1,500000);

update student set fio = 'name';

ALTER TABLE student SET (autovacuum_enabled = on);

```


