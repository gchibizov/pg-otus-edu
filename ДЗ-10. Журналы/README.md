# Работы с журналами

## Цели:
1. Уметь работать с журналами и контрольными точками.
1. Уметь настраивать параметры журналов.

## Дополнительные материалы:
1. WAL в PostgreSQL: 3. Контрольная точка: https://habr.com/ru/company/postgrespro/blog/460423/.

## Описание выполнения домашнего задания

### Шаг 1. Настройте выполнение контрольной точки раз в 30 секунд

> Используем сервер postgres, устновленный в контейнере Docker, на локальной машине MacOS. Настроим выполнение контрольной точки.

### Результат

Для того, чтобы выполнить настройку воспользуемся DBeaver. Список команд:

```sql
-- Список основных настроек для выполнения контрольных точек
select * from pg_catalog.pg_settings where name like 'checkpoint%';
-- Измени значение настройки
alter system set checkpoint_timeout = 30;
-- Перегрузим настройки
select pg_catalog.pg_reload_conf()

```
![Настройки checkpoint](/images/scr-dz10-01.png)


### Шаг 2. 10 минут c помощью утилиты pgbench подавайте нагрузку

> Тесты ранее уже запускали, повторим процедуру.

### Результат

Перед тем, как запустить тесты, подготовим системные счетчики для статистики (каталог *pg_stat*). Можно выполнить сброс счетчиков через функцию *pg_stat_reset()*, но ее работа для меня остатется пока под вопросом ... поэтому надежнее почистить каталог *pg_stat*. После сброса статистики желательно выполнить команду *ANALYZE*. Развернут тестовый сервер, поэтому пока можем себе такое позволить.

P.S. Нашел более простой способ: *select pg_stat_reset_shared('wal')*.

```bash
# Инициализация таблиц для работы тестов
postgres@64294a4ab873:/usr/lib/postgresql/14/bin$ pgbench -i postgres
# Подаем нагрузку, отписываем результат через каждые 30 секунд
postgres@64294a4ab873:/usr/lib/postgresql/14/bin$ pgbench -c8 -P 30 -T 600 postgres

```
![Нагрузка под pgbench](/images/scr-dz10-02.png)


### Шаг 3. Измерить объем журнальных файлов + проверить данные статистики

> Оценим объем журнальных файлов. Для анализа будем использовать представления *pg_ls_waldir()* + *pg_stat_wal* + *pg_stat_bgwriter*. Используем простые метрики.

### Результат

Важный момент, на который стоит обратить внимание, это настройки логирования. Нужно понимать, что кластер *Postgre* развернут в *Docker*. Поэтому там явно будут прописаны специфические настройки. Я столкнулся с тем, что *Postgre* не собирал все логи в одно место - каталог *log*. Был отключен *logging_collector*. Если для каталога с логами не задан особый путь (указана просто каталог *log*), то каталог будет создан в каталоге с данными.

```sql
-- Команды для проверки настроек логирования
select * from pg_catalog.pg_settings where name in ('log_checkpoints', 'logging_collector', 'log_directory', 'log_filename', 'data_directory')
alter system set logging_collector = 'on';
alter system set log_checkpoints = 'on';
-- Представления для просмотра количества журнальных файлов
select * from pg_ls_waldir() limit 10;
select count(1) from pg_ls_waldir();
-- Представления для статистики работы с контрольными точками и WAL
select * from pg_catalog.pg_stat_wal
select * from pg_catalog.pg_stat_bgwriter
-- Представление для дополнителькной информации по выполнению последней контрольной точки
select * from pg_catalog.pg_control_checkpoint()

```

Количество журнальных файлов в среднем генериломь 5 штук на каждую контрольную точку. Это в районе 83,9 Мб.
![Количество WAL-файлов](/images/scr-dz10-03.png)

Промежуточные результаты запуска *pgbench*. Средняя производительность = 590 tps.

```bash
postgres@64294a4ab873:/usr/lib/postgresql/14/bin$ pgbench -c8 -P 30 -T 600 postgres

pgbench (14.4 (Debian 14.4-1.pgdg110+1))
starting vacuum...end.
progress: 30.0 s, 583.8 tps, lat 13.647 ms stddev 8.852
progress: 60.0 s, 600.1 tps, lat 13.323 ms stddev 8.279
progress: 90.0 s, 603.3 tps, lat 13.251 ms stddev 8.151
progress: 120.0 s, 604.7 tps, lat 13.223 ms stddev 8.023
progress: 150.0 s, 608.2 tps, lat 13.145 ms stddev 7.966
progress: 180.0 s, 576.5 tps, lat 13.870 ms stddev 9.022
progress: 210.0 s, 600.6 tps, lat 13.310 ms stddev 8.070
progress: 240.0 s, 567.4 tps, lat 14.093 ms stddev 9.559
progress: 270.0 s, 588.7 tps, lat 13.580 ms stddev 8.251
progress: 300.0 s, 600.3 tps, lat 13.320 ms stddev 8.266
progress: 330.0 s, 599.3 tps, lat 13.339 ms stddev 8.070
progress: 360.0 s, 594.3 tps, lat 13.455 ms stddev 8.103
progress: 390.0 s, 600.1 tps, lat 13.321 ms stddev 7.980
progress: 420.0 s, 584.7 tps, lat 13.675 ms stddev 8.745
progress: 450.0 s, 580.3 tps, lat 13.775 ms stddev 8.371
progress: 480.0 s, 586.4 tps, lat 13.635 ms stddev 8.428
progress: 510.0 s, 581.7 tps, lat 13.745 ms stddev 8.359
progress: 540.0 s, 570.7 tps, lat 14.011 ms stddev 9.351
progress: 570.0 s, 574.2 tps, lat 13.924 ms stddev 8.761
progress: 600.0 s, 594.9 tps, lat 13.439 ms stddev 8.346
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 600 s
number of transactions actually processed: 354011
latency average = 13.549 ms
latency stddev = 8.456 ms
initial connection time = 97.491 ms
tps = 590.096183 (without initial connection time)
```

Записи из лога, выполнение контрольных точек. По логу видим, что запуск контрольных точек сервер выполняет строго по расписанию и делает все для этого: время записи контрольной точки укладывается в интервал до 30 секунд. Вспоминаем настройку *checkpoint_completion_target*.

```bash
2022-11-26 10:42:05.043 UTC [35] LOG:  checkpoint complete: wrote 1696 buffers (10.4%); 0 WAL file(s) added, 0 removed, 0 recycled; write=26.246 s, sync=0.004 s, total=26.318 s; sync files=37, longest=0.002 s, average=0.001 s; distance=12814 kB, estimate=12814 kB
2022-11-26 10:42:08.047 UTC [35] LOG:  checkpoint starting: time
2022-11-26 10:42:35.178 UTC [35] LOG:  checkpoint complete: wrote 1981 buffers (12.1%); 0 WAL file(s) added, 0 removed, 2 recycled; write=27.014 s, sync=0.002 s, total=27.131 s; sync files=16, longest=0.001 s, average=0.001 s; distance=19362 kB, estimate=19362 kB
2022-11-26 10:42:38.180 UTC [35] LOG:  checkpoint starting: time
2022-11-26 10:43:05.170 UTC [35] LOG:  checkpoint complete: wrote 1878 buffers (11.5%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.899 s, sync=0.003 s, total=26.991 s; sync files=9, longest=0.001 s, average=0.001 s; distance=23024 kB, estimate=23024 kB
2022-11-26 10:43:08.173 UTC [35] LOG:  checkpoint starting: time
2022-11-26 10:43:35.111 UTC [35] LOG:  checkpoint complete: wrote 2053 buffers (12.5%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.866 s, sync=0.002 s, total=26.938 s; sync files=18, longest=0.002 s, average=0.001 s; distance=22823 kB, estimate=23004 kB
2022-11-26 10:43:38.113 UTC [35] LOG:  checkpoint starting: time
2022-11-26 10:44:05.161 UTC [35] LOG:  checkpoint complete: wrote 2031 buffers (12.4%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.950 s, sync=0.002 s, total=27.048 s; sync files=8, longest=0.002 s, average=0.001 s; distance=23786 kB, estimate=23786 kB
2022-11-26 10:44:08.162 UTC [35] LOG:  checkpoint starting: time
2022-11-26 10:44:35.154 UTC [35] LOG:  checkpoint complete: wrote 2113 buffers (12.9%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.871 s, sync=0.004 s, total=26.992 s; sync files=16, longest=0.004 s, average=0.001 s; distance=24041 kB, estimate=24041 kB
2022-11-26 10:44:38.155 UTC [35] LOG:  checkpoint starting: time
2022-11-26 10:45:05.151 UTC [35] LOG:  checkpoint complete: wrote 2001 buffers (12.2%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.873 s, sync=0.003 s, total=26.996 s; sync files=8, longest=0.003 s, average=0.001 s; distance=23192 kB, estimate=23956 kB
2022-11-26 10:45:08.152 UTC [35] LOG:  checkpoint starting: time
2022-11-26 10:45:35.094 UTC [35] LOG:  checkpoint complete: wrote 2088 buffers (12.7%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.856 s, sync=0.004 s, total=26.942 s; sync files=14, longest=0.004 s, average=0.001 s; distance=22863 kB, estimate=23847 kB
2022-11-26 10:45:38.098 UTC [35] LOG:  checkpoint starting: time
2022-11-26 10:46:05.121 UTC [35] LOG:  checkpoint complete: wrote 1975 buffers (12.1%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.895 s, sync=0.001 s, total=27.024 s; sync files=8, longest=0.001 s, average=0.001 s; distance=22455 kB, estimate=23707 kB
2022-11-26 10:46:08.124 UTC [35] LOG:  checkpoint starting: time
2022-11-26 10:46:35.094 UTC [35] LOG:  checkpoint complete: wrote 2108 buffers (12.9%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.869 s, sync=0.002 s, total=26.970 s; sync files=16, longest=0.002 s, average=0.001 s; distance=22749 kB, estimate=23612 kB
2022-11-26 10:46:38.097 UTC [35] LOG:  checkpoint starting: time
2022-11-26 10:47:05.129 UTC [35] LOG:  checkpoint complete: wrote 1972 buffers (12.0%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.945 s, sync=0.002 s, total=27.032 s; sync files=8, longest=0.002 s, average=0.001 s; distance=22616 kB, estimate=23512 kB
2022-11-26 10:47:08.130 UTC [35] LOG:  checkpoint starting: time
2022-11-26 10:47:35.085 UTC [35] LOG:  checkpoint complete: wrote 2114 buffers (12.9%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.848 s, sync=0.002 s, total=26.956 s; sync files=15, longest=0.002 s, average=0.001 s; distance=22785 kB, estimate=23439 kB
2022-11-26 10:47:38.088 UTC [35] LOG:  checkpoint starting: time
2022-11-26 10:48:05.133 UTC [35] LOG:  checkpoint complete: wrote 1987 buffers (12.1%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.956 s, sync=0.001 s, total=27.045 s; sync files=8, longest=0.001 s, average=0.001 s; distance=22666 kB, estimate=23362 kB
2022-11-26 10:48:08.136 UTC [35] LOG:  checkpoint starting: time
2022-11-26 10:48:35.160 UTC [35] LOG:  checkpoint complete: wrote 2085 buffers (12.7%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.951 s, sync=0.001 s, total=27.024 s; sync files=13, longest=0.001 s, average=0.001 s; distance=22541 kB, estimate=23280 kB
2022-11-26 10:48:38.163 UTC [35] LOG:  checkpoint starting: time
2022-11-26 10:49:05.133 UTC [35] LOG:  checkpoint complete: wrote 1984 buffers (12.1%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.872 s, sync=0.003 s, total=26.971 s; sync files=7, longest=0.003 s, average=0.001 s; distance=22538 kB, estimate=23206 kB
2022-11-26 10:49:08.135 UTC [35] LOG:  checkpoint starting: time
2022-11-26 10:49:35.153 UTC [35] LOG:  checkpoint complete: wrote 2301 buffers (14.0%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.944 s, sync=0.001 s, total=27.018 s; sync files=17, longest=0.001 s, average=0.001 s; distance=22207 kB, estimate=23106 kB
2022-11-26 10:49:38.153 UTC [35] LOG:  checkpoint starting: time
2022-11-26 10:50:05.165 UTC [35] LOG:  checkpoint complete: wrote 1978 buffers (12.1%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.895 s, sync=0.001 s, total=27.013 s; sync files=9, longest=0.001 s, average=0.001 s; distance=22260 kB, estimate=23021 kB
2022-11-26 10:50:08.169 UTC [35] LOG:  checkpoint starting: time
2022-11-26 10:50:35.132 UTC [35] LOG:  checkpoint complete: wrote 2066 buffers (12.6%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.870 s, sync=0.001 s, total=26.964 s; sync files=12, longest=0.001 s, average=0.001 s; distance=22136 kB, estimate=22933 kB
2022-11-26 10:50:38.135 UTC [35] LOG:  checkpoint starting: time
2022-11-26 10:51:05.123 UTC [35] LOG:  checkpoint complete: wrote 1953 buffers (11.9%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.904 s, sync=0.003 s, total=26.989 s; sync files=10, longest=0.003 s, average=0.001 s; distance=22038 kB, estimate=22843 kB
2022-11-26 10:51:08.127 UTC [35] LOG:  checkpoint starting: time
2022-11-26 10:51:35.133 UTC [35] LOG:  checkpoint complete: wrote 2270 buffers (13.9%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.902 s, sync=0.002 s, total=27.007 s; sync files=13, longest=0.002 s, average=0.001 s; distance=22207 kB, estimate=22780 kB
2022-11-26 10:51:38.136 UTC [35] LOG:  checkpoint starting: time
2022-11-26 10:52:05.139 UTC [35] LOG:  checkpoint complete: wrote 1959 buffers (12.0%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.899 s, sync=0.003 s, total=27.004 s; sync files=12, longest=0.003 s, average=0.001 s; distance=22059 kB, estimate=22708 kB
2022-11-26 10:53:08.176 UTC [35] LOG:  checkpoint starting: time
2022-11-26 10:53:35.122 UTC [35] LOG:  checkpoint complete: wrote 1128 buffers (6.9%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.848 s, sync=0.003 s, total=26.947 s; sync files=13, longest=0.003 s, average=0.001 s; distance=17750 kB, estimate=22212 kB

```


### Шаг 4. Сравните tps в синхронном/асинхронном режиме утилитой pgbench. Объясните полученный результат.

> Переведем сервер Postgre в асинхронный режим записи журнальных файлов. Проверим производительность.

### Результат

Изменение настройки выполняется стандартным способом. Повторим нагрузку в виде теста *pgbench*.

```sql
-- Включаем асинхронный режим
alter system set synchronous_commit = off;
-- Перезагрузка настроек
select pg_reload_conf();

```

Промежуточные результаты запуска *pgbench*. Средняя производительность = 3602 tps (в 6 раз). Учитываем SSD диск и упрощение процесса записи журнальны буферов, строго через конкретные промежутки времени.

```bash
postgres@64294a4ab873:/usr/lib/postgresql/14/bin$ pgbench -c8 -P 30 -T 600 postgres

pgbench (14.4 (Debian 14.4-1.pgdg110+1))
starting vacuum...end.
progress: 30.0 s, 3350.5 tps, lat 2.371 ms stddev 3.615
progress: 60.0 s, 3151.3 tps, lat 2.529 ms stddev 2.832
progress: 90.0 s, 3430.7 tps, lat 2.324 ms stddev 2.768
progress: 120.0 s, 3692.4 tps, lat 2.159 ms stddev 2.252
progress: 150.0 s, 3499.1 tps, lat 2.278 ms stddev 2.411
progress: 180.0 s, 3613.5 tps, lat 2.205 ms stddev 2.231
progress: 210.0 s, 3553.4 tps, lat 2.244 ms stddev 2.321
progress: 240.0 s, 3587.3 tps, lat 2.222 ms stddev 2.221
progress: 270.0 s, 3528.6 tps, lat 2.259 ms stddev 2.369
progress: 300.0 s, 3632.7 tps, lat 2.194 ms stddev 2.242
progress: 330.0 s, 3635.6 tps, lat 2.192 ms stddev 2.246
progress: 360.0 s, 3747.8 tps, lat 2.128 ms stddev 2.066
progress: 390.0 s, 3611.4 tps, lat 2.207 ms stddev 2.488
progress: 420.0 s, 3787.1 tps, lat 2.105 ms stddev 2.011
progress: 450.0 s, 3690.2 tps, lat 2.160 ms stddev 2.230
progress: 480.0 s, 3806.9 tps, lat 2.094 ms stddev 2.063
progress: 510.0 s, 3628.4 tps, lat 2.197 ms stddev 2.338
progress: 540.0 s, 3778.4 tps, lat 2.110 ms stddev 2.056
progress: 570.0 s, 3651.9 tps, lat 2.183 ms stddev 2.311
progress: 600.0 s, 3657.8 tps, lat 2.179 ms stddev 2.354
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 600 s
number of transactions actually processed: 2161063
latency average = 2.213 ms
latency stddev = 2.386 ms
initial connection time = 96.835 ms
tps = 3602.262727 (without initial connection time)
```


### Шаг 5. Создайте новый кластер с включенной контрольной суммой страниц. Создайте таблицу. Вставьте несколько значений. Выключите кластер. Измените пару байт в таблице. Включите кластер и сделайте выборку из таблицы. Что и почему произошло? как проигнорировать ошибку и продолжить работу?

> Поднимаем отдельный контейнер для *Postgres* с включенной проверкой контрольных сумм. Останавливаем контейнер и пробуем поменять файл с данными. Далее запускаем контейнер и пробуем обратиться к таблице, файл с данными которой поменяли.

### Результат

Команды для поднятия контейнера из *Docker*.

```bash
# Запуск контейнера
sudo docker run --name pg-docker-checksum --network pg-net -e POSTGRES_PASSWORD=postgres -e POSTGRES_INITDB_ARGS=--data-checksums -d -p 5432:5432 -v /var/lib/postgres_checksum:/var/lib/postgresql/data postgres:14

# Создадим объекты для pgbench (под пользователем postgres)
pgbench -i postgres

```

Команды для проверки настроек и размещенщия файла данных.

```sql
-- Настройки
select * from pg_catalog.pg_settings where name in ('data_checksums','ignore_checksum_failure')
-- Файл с данными
select * from pg_relation_filepath('pgbench_accounts');
-- Обращение к таблице
select * from pgbench_accounts

```

Теперь посмотрим, к чему привели наши ручные правки файла с данными.

```bash
PostgreSQL Database directory appears to contain a database; Skipping initialization

2022-11-26 12:40:41.725 UTC [1] LOG:  starting PostgreSQL 14.4 (Debian 14.4-1.pgdg110+1) on aarch64-unknown-linux-gnu, compiled by gcc (Debian 10.2.1-6) 10.2.1 20210110, 64-bit
2022-11-26 12:40:41.726 UTC [1] LOG:  listening on IPv4 address "0.0.0.0", port 5432
2022-11-26 12:40:41.726 UTC [1] LOG:  listening on IPv6 address "::", port 5432
2022-11-26 12:40:41.730 UTC [1] LOG:  listening on Unix socket "/var/run/postgresql/.s.PGSQL.5432"
2022-11-26 12:40:41.748 UTC [27] LOG:  database system was shut down at 2022-11-26 12:39:58 UTC
2022-11-26 12:40:41.790 UTC [1] LOG:  database system is ready to accept connections
2022-11-26 12:41:41.911 UTC [31] WARNING:  could not open statistics file "pg_stat_tmp/global.stat": Operation not permitted
2022-11-26 12:41:42.751 UTC [38] WARNING:  page verification failed, calculated checksum 44137 but expected 48575
2022-11-26 12:41:42.751 UTC [38] ERROR:  invalid page in block 0 of relation base/13757/16396
2022-11-26 12:41:42.751 UTC [38] STATEMENT:     select * from pgbench_accounts pa
```

```sql
-- Настройки
alter system set ignore_checksum_failure = on;
select pg_reload_conf();
-- Обращение к таблице
select * from pgbench_accounts

```

НО теперь все равно не хочется читать данные из таблицы. Возможно, как-то не очень хорошо поправил файл с данными .. но мысль понятна.

```bash
postgres@3e8d22724c1d:/$ psql

psql (14.4 (Debian 14.4-1.pgdg110+1))
Type "help" for help.

postgres=# show ignore_checksum_failure;
 ignore_checksum_failure
-------------------------
 on
(1 row)

postgres=# select * from pgbench_accounts;
WARNING:  page verification failed, calculated checksum 21392 but expected 61232
ERROR:  invalid page in block 0 of relation base/13757/16418
postgres=#
```