# Репликация

## Цели:
1. Реализовать свой миникластер на 3 ВМ.

## Дополнительные материалы:
1.

## Описание выполнения домашнего задания

### Шаг 1. Настроить 3 ВМ (подготокить машины и развернуть Postgre). На ВМ1 создаем таблицы test1 для записи, test2 для запросов на чтение. Создаем публикацию таблицы test1 и подписываемся на публикацию таблицы test2 с ВМ2.

> Настраиваем 3 ВМ для работы, продолжаю использовать **Docker**. Две машниы более производительные, одна медленная. Посмотрим будут ли проблемы репликации, связанные с производительностью медленной машины.

### Результат

Разворачиваем уже знакомый нам контейнер с **Postgres**, подкручиваем настроойки сервера под лучшу производительность. Используем следующие сетевые адреса машин:

```bash
# ВМ1 = 192.168.1.89
# ВМ2 = 192.168.1.91 (медленная машина)
# ВМ3 = 192.168.1.100

psql -h 192.168.1.91 -p 5432 -U postgres
psql -h 192.168.1.100 -p 5432 -U postgres
```

Проверим подключение.

```bash
postgres@64294a4ab873:/$ psql -h 192.168.1.91 -p 5432 -U postgres
Password for user postgres:
psql (14.4 (Debian 14.4-1.pgdg110+1), server 14.6 (Debian 14.6-1.pgdg110+1))
Type "help" for help.

postgres=# exit

postgres@64294a4ab873:/$ psql -h 192.168.1.100 -p 5432 -U postgres
Password for user postgres:
psql (14.4 (Debian 14.4-1.pgdg110+1), server 14.5 (Debian 14.5-2.pgdg110+2))
Type "help" for help.

postgres=# exit
```

Для **Docker** все подключения открыты, оставим настройки фаервола для выполнения задачи такими. Будем использовать так же пользователя **postgres**. Но на практике лучше создавать отдельного пользователя для репликации и ограничивать подключения.

Подготовм запрос для создания таблиц, публикации и подписки. Выполним его на **ВМ1** и **ВМ2**. Запросы выполним через **DBeaver**.

```sql
-- Настройки
show wal_level;
alter system set wal_level='logical';
-- Создание таблиц
create table test1 (id integer primary key, value text);
create table test2 (id integer primary key, value text);
-- Создание публикации
create publication pub1_test1 for table test1;
-- Создание подписки
create subscription sub1_pub2_test2
  connection 'host=192.168.1.91 port=5432 user=postgres password=postgres dbname=postgres'
  publication pub2_test2;
-- Проверить публикации
select * from pg_publication;
-- Проверить подписки
select * from pg_subscription;

```


### Шаг 2. На ВМ2 создаем таблицы test2 для записи, test1 для запросов на чтение. Создаем публикацию таблицы test2 и подписываемся на публикацию таблицы test1 с ВМ1

> Выполним похоже действия, что и на шаге 1. Убедимся, что все запросы выполнены и написаны корректно.

### Результат

Подготовм запрос для создания таблиц, публикации и подписки. Выполним его на **ВМ1** и **ВМ2**. Запросы выполним через **DBeaver**.

```sql
-- Настройки
show wal_level;
alter system set wal_level='logical';
-- Создание таблиц
create table test1 (id integer primary key, value text);
create table test2 (id integer primary key, value text);
-- Создание публикации
create publication pub2_test2 for table test2;
-- Создание подписки
create subscription sub2_pub1_test1
  connection 'host=192.168.1.89 port=5432 user=postgres password=postgres dbname=postgres'
  publication pub1_test1;
-- Проверить публикации
select * from pg_publication;
-- Проверить подписки
select * from pg_subscription;
-- Слоты репликации
select * from pg_replication_slots
-- Удалить слот репликации (если создан по ошбке)
select pg_drop_replication_slot('sub2_pub1_test1');

```

Подписки для **ВМ1** и **ВМ2**.

![Список подписок для ВМ2](/images/scr-dz14-01.png)

![Список подписок для ВМ1](/images/scr-dz14-02.png)



### Шаг 3. ВМ3 использовать как реплику для чтения и бэкапов (подписаться на таблицы из ВМ1 и ВМ2 )

> Создаем подписки для репликации. Выполняем запросы для добавления данных и смотрим на результат.

### Результат

Подготовм **ВМ3**. После того, как все подписки будут готовы, выполним запросы.

```sql
-- Настройки
show wal_level;
alter system set wal_level='logical';
-- Создание таблиц
create table test1 (id integer primary key, value text);
create table test2 (id integer primary key, value text);
-- Создание подписки
create subscription sub3_pub2_test2
  connection 'host=192.168.1.91 port=5432 user=postgres password=postgres dbname=postgres'
  publication pub2_test2;

create subscription sub3_pub1_test1
  connection 'host=192.168.1.89 port=5432 user=postgres password=postgres dbname=postgres'
  publication pub1_test1;
-- Проверить подписки
select * from pg_subscription;
-- Слоты репликации
select * from pg_replication_slots
-- Удалить слот репликации (если создан по ошбке), обращать внимание на PID процесса, который использует слот
select pg_drop_replication_slot('sub3_pub2_test2');
select pg_drop_replication_slot('sub3_pub1_test1');

-- Запросы для добавления данных на ВМ1 и ВМ2
insert into test1(id, value) values(1, 'Value1 for Test1');
insert into test2(id, value) values(1, 'Value1 for Test2');
-- Запросы для просмотра данных
select * from test1
select * from test2

```
![Список подписок для ВМ3](/images/scr-dz14-03.png)

Проверим теперь все запрос через утилиту **psql** на **ВМ3**.

![Список подписок для ВМ3](/images/scr-dz14-04.png)

> Получили работающую репликацию для трех машин. Внимательно нужно относиться к наименованию подписок (слотов репликации). Следить за процессами, которые используют слоты. Грамотно продумывать структуру таблиц (на занятии говорили про проблемы с уникальными индексами), медленная машина немного подтомрозила с запросом .. но ошибок никаких не выводила.
