# Настройка autovacuum с учетом оптимальной производительности

## Цели:
1. Запустить нагрузочный тест pgbench с профилем нагрузки DWH.
1. Настроить параметры autovacuum для достижения максимального уровня устойчивой производительности.

## Описание выполнения домашнего задания

### Шаг 1. Создать инстанс Yandex Cloud, развернуть PostgreSQL, применить настройки из файла.

> В качестве платформы с облачными сервисами будем использовать Yandex Cloud. Подготовим инстанс по уже известным шагам. Опишем команды для установки PostgreSQL 14 и выполнения необходимых действий.

### Результат

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

# Настройки для кластера PostgreSQL
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

### Шаг 2. Проверим работу autovacuum на примере 4 профилей.

> Задача проверить зависимость между настройками **autovacuum** и количеством tps на определенном горизонте времени (в районе 30 мин - 1 часа). Для проверки будем использовать 4 профиля.

### Результат

Настройки **Профиль 1** = стандартные настройки **autovaccum** из коробки.

Выполним pgbench:

```bash
# Команды
pgbench -i postgres
pgbench -c8 -P 60 -T 3600 -U postgres postgres
```
![Запуск pgbench](/images/scr-dz08-03.png)


Настройки **Профиль 2** = берем часть настроек из рекомендаций

```bash
#------------------------------------------------------------------------------
# AUTOVACUUM
#------------------------------------------------------------------------------

autovacuum = on                         # Enable autovacuum subprocess?  'on'
                                        # requires track_counts to also be on.
autovacuum_max_workers = 6             # max number of autovacuum subprocesses
                                        # (change requires restart)
autovacuum_naptime = 15s                # time between autovacuum runs
autovacuum_vacuum_threshold = 25        # min number of row updates before
                                        # vacuum
#autovacuum_vacuum_insert_threshold = 1000      # min number of row inserts
                                        # before vacuum; -1 disables insert
                                        # vacuums
#autovacuum_analyze_threshold = 50      # min number of row updates before
                                        # analyze
autovacuum_vacuum_scale_factor = 0.05    # fraction of table size before vacuum
#autovacuum_vacuum_insert_scale_factor = 0.2    # fraction of inserts over table
                                        # size before insert vacuum
#autovacuum_analyze_scale_factor = 0.1  # fraction of table size before analyze
#autovacuum_freeze_max_age = 200000000  # maximum XID age before forced vacuum
                                        # (change requires restart)
#autovacuum_multixact_freeze_max_age = 400000000        # maximum multixact age
                                        # before forced vacuum
                                        # (change requires restart)
autovacuum_vacuum_cost_delay = 10       # default vacuum cost delay for
                                        # autovacuum, in milliseconds;
                                        # -1 means use vacuum_cost_delay
autovacuum_vacuum_cost_limit = 1000     # default vacuum cost limit for
                                        # autovacuum, -1 means use
                                        # vacuum_cost_limit
```

Настройки **Профиль 3** = увеличиваем количество воркеров до 10

```bash
#------------------------------------------------------------------------------
# AUTOVACUUM
#------------------------------------------------------------------------------

autovacuum = on                         # Enable autovacuum subprocess?  'on'
                                        # requires track_counts to also be on.
autovacuum_max_workers = 10             # max number of autovacuum subprocesses
                                        # (change requires restart)
autovacuum_naptime = 15s                # time between autovacuum runs
autovacuum_vacuum_threshold = 50        # min number of row updates before
                                        # vacuum
#autovacuum_vacuum_insert_threshold = 1000      # min number of row inserts
                                        # before vacuum; -1 disables insert
                                        # vacuums
#autovacuum_analyze_threshold = 50      # min number of row updates before
                                        # analyze
autovacuum_vacuum_scale_factor = 0.1    # fraction of table size before vacuum
#autovacuum_vacuum_insert_scale_factor = 0.2    # fraction of inserts over table
                                        # size before insert vacuum
#autovacuum_analyze_scale_factor = 0.1  # fraction of table size before analyze
#autovacuum_freeze_max_age = 200000000  # maximum XID age before forced vacuum
                                        # (change requires restart)
#autovacuum_multixact_freeze_max_age = 400000000        # maximum multixact age
                                        # before forced vacuum
                                        # (change requires restart)
autovacuum_vacuum_cost_delay = 10       # default vacuum cost delay for
                                        # autovacuum, in milliseconds;
                                        # -1 means use vacuum_cost_delay
autovacuum_vacuum_cost_limit = 1000     # default vacuum cost limit for
                                        # autovacuum, -1 means use
                                        # vacuum_cost_limit
```

Настройки **Профиль 4** = уменьшаем интервал для пробуждения и по сути размер пачки до стандартной

```bash
#------------------------------------------------------------------------------
# AUTOVACUUM
#------------------------------------------------------------------------------

autovacuum = on                         # Enable autovacuum subprocess?  'on'
                                        # requires track_counts to also be on.
autovacuum_max_workers = 10             # max number of autovacuum subprocesses
                                        # (change requires restart)
autovacuum_naptime = 1s                 # time between autovacuum runs
autovacuum_vacuum_threshold = 50        # min number of row updates before
                                        # vacuum
#autovacuum_vacuum_insert_threshold = 1000      # min number of row inserts
                                        # before vacuum; -1 disables insert
                                        # vacuums
autovacuum_analyze_threshold = 50       # min number of row updates before
                                        # analyze
autovacuum_vacuum_scale_factor = 0.05   # fraction of table size before vacuum
#autovacuum_vacuum_insert_scale_factor = 0.2    # fraction of inserts over table
                                        # size before insert vacuum
autovacuum_analyze_scale_factor = 0.05  # fraction of table size before analyze
#autovacuum_freeze_max_age = 200000000  # maximum XID age before forced vacuum
                                        # (change requires restart)
#autovacuum_multixact_freeze_max_age = 400000000        # maximum multixact age
                                        # before forced vacuum
                                        # (change requires restart)
autovacuum_vacuum_cost_delay = 5        # default vacuum cost delay for
                                        # autovacuum, in milliseconds;
                                        # -1 means use vacuum_cost_delay
autovacuum_vacuum_cost_limit = 200      # default vacuum cost limit for
                                        # autovacuum, -1 means use
                                        # vacuum_cost_limit

```


Опишем результаты работы профилей в одной общей табличке:

|  Профиль / Интервалы времени по 60 сек         |  T1   |  T2   |  T3   |  T4   |  T5   |  T6   |  T7   |  T8   |  T9   |  T10  |
|------------------------------------------------|-------|-------|-------|-------|-------|-------|-------|-------|-------|-------|
|  "Профиль 1", tps                              |  606  |  571  |  590  |  596  |  580  |  576  |  566  |  568  |  629  |  538  |
|  "Профиль 2. Vacuum пришел на 51 отрезке", tps |  316  |  317  |  316  |  316  |  314  |  315  |  317  |  317  |  315  |  311  |
|  "Профиль 3. Vacuum пришел на 4 отрезке", tps  |  320  |  320  |  319  |  320  |  316  |  320  |  320  |  319  |  319  |  317  |
|  "Профиль 4. Vacuum пришел на 3 отрезке", tps  |  318  |  317  |  317  |  318  |  316  |  317  |  319  |  317  |  318  |  317  |


![График с профилями настроек](/images/scr-dz08-04.png)

### Выводы

1. Когда приходит **autovacuum** мы чувствуем сразу и это отражается на **tps**. Поэтому очень важно не промахнуться с настройками.
1. Стандартный вывод = отключать **autovacuum** нельзя, выполняет очень важную работу.
1. Активно работающие воркеры **autovacuum** могут похоронить скорость выполнения операций чтения записи для файлов БД.
1. Конфигурировать **autovacuum** не просто, поэтому лучше тщательно подходить к испытаниям и проверки определенного профиля настроек.