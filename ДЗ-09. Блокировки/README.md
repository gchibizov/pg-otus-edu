# Механизм блокировок

## Цели:
1. Понимать как работает механизм блокировок объектов и строк.

## Описание выполнения домашнего задания

### Шаг 1. Настроить сервер для отображения блокировок длительностью более 200мс. Влспроизвести одну блокировку и посмотреть журнал

> В качестве платформы с облачными сервисами будем использовать Yandex Cloud. Подготовим инстанс, .

### Результат

Развернута виртуальная машина (**gchibizov-pg-20220814**), подготовлен инстанс PostgreSQL:

```bash
# Установка PostgreSQL
sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-14

# Настройка сервера в части блокировок
show deadlock_timeout ;
show log_lock_waits ;
```
![Настройки инстанса PostgreSQL на ВМ](/images/scr-dz09-01.png)

Берем пример из занятия, создаем блокировку типа ShareLock:


```sql
-- Что произойдет, если выполнить команду CREATE INDEX?
-- Находим в документации, что эта команда устанавливает блокировку в режиме Share. По матрице определяем, что команда совместима сама с собой (то есть можно одновременно создавать несколько индексов) и с читающими командами. Таким образом, команды SELECT продолжат работу, а вот команды UPDATE, DELETE, INSERT будут заблокированы.
-- И наоборот — незавершенные транзакции, изменяющие данные в таблице, будут блокировать работу команды CREATE INDEX. Поэтому и существует вариант команды — CREATE INDEX CONCURRENTLY. Он работает дольше (и может даже упасть с ошибкой), зато допускает одновременное изменение данных.

-- Session #1
sudo -u postgres psql
\c locks
CREATE TABLE accounts(
  acc_no integer PRIMARY KEY,
  amount numeric
);
INSERT INTO accounts VALUES (1,1000.00), (2,2000.00), (3,3000.00);


-- Session #2
sudo -u postgres psql
\c locks
-- Во втором сеансе начнем транзакцию. Нам понадобится номер обслуживающего процесса.
BEGIN;
SELECT pg_backend_pid();

-- Какие блокировки удерживает только что начавшаяся транзакция? Смотрим в pg_locks:
SELECT locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted
FROM pg_locks WHERE pid = 522;
-- Транзакция всегда удерживает исключительную (ExclusiveLock) блокировку собственного номера, а данном случае — виртуального. Других блокировок у этого процесса нет.

-- Теперь обновим строку таблицы. Как изменится ситуация?
UPDATE accounts SET amount = amount + 100 WHERE acc_no = 1;

SELECT locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted
FROM pg_locks WHERE pid = 522;
-- Теперь появились блокировки изменяемой таблицы и индекса (созданного для первичного ключа), который используется командой UPDATE. Обе блокировки взяты в режиме RowExclusiveLock. Кроме того, добавилась исключительная блокировка настоящего номера транзакции (который появился, как только транзакция начала изменять данные).


-- Session #3
sudo -u postgres psql
\c locks
-- Теперь попробуем в еще одном сеансе создать индекс по таблице.
BEGIN;
SELECT pg_backend_pid();

CREATE INDEX ON accounts(acc_no);
-- Команда «подвисает» в ожидании освобождения ресурса. Какую именно блокировку она пытается захватить? Проверим:


-- Session #1
-- SELECT pg_backend_pid();
SELECT locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted FROM pg_locks WHERE pid = 523;
-- Видим, что транзакция пытается получить блокировку таблицы в режиме ShareLock, но не может (granted = f).

-- Находить номер блокирующего процесса, а в общем случае — несколько номеров, удобно с помощью функции, которая появилась в версии 9.6 (до того приходилось делать выводы, вдумчиво разглядывая все содержимое pg_locks):
SELECT pg_blocking_pids(523);

-- И затем, чтобы разобраться в ситуации, можно получить информацию о сеансах, к которым относятся найденные номера:
SELECT * FROM pg_stat_activity WHERE pid = ANY(pg_blocking_pids(523)) \gx


-- Session #2
-- После завершения транзакции блокировки снимаются и индекс создается.
COMMIT;


-- Session #3
COMMIT;
```
![Блокировка в журнале](/images/scr-dz09-02.png)


### Шаг 2. Анализ блокировок, возникающих при использовании одной и той же команды UPDATE конкретной строки из разных сессий

> Посмотрим на механизм блокировок, как он работает при изменении одной и той же строки из 3 разных сессий.

### Результат

Команда, которая будет выполняться в каждой из трех сессий:

```sql
BEGIN;

select txid_current(); --759
SELECT pg_backend_pid(); --1011

UPDATE accounts SET amount = amount + 1 WHERE acc_no = 1;

SELECT locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted
FROM pg_locks WHERE pid = 1011;

COMMIT;

END;
```

### Блокировки для сессии 1

|   locktype    |    relation   | virtxid |  xid  |      node        | granted |
|---------------|---------------|---------|-------|------------------|---------|
| relation      | accounts      |         |       | RowExclusiveLock |   true  |
| relation      | accounts_pkey |         |       | RowExclusiveLock |   true  |
| virtualxid    |               |  19/40  |       | ExclusiveLock    |   true  |
| transactionid |               |         |  759  | ExclusiveLock    |   true  |

Так мы меняем строку, накладывается исключительная блокировка на строку (ключ). Ключ ранее был создан в примере, используем тот же пример. Так же исключительные блокировки накладываются на транзакции (в том числе виртуальную).

### Блокировки для сессии 2

|   locktype    |    relation   | virtxid |  xid  |      node        | granted |
|---------------|---------------|---------|-------|------------------|---------|
| relation      | accounts      |         |       | RowExclusiveLock |   true  |
| relation      | accounts_pkey |         |       | RowExclusiveLock |   true  |
| virtualxid    |               |  20/24  |       | ExclusiveLock    |   true  |
| transactionid |               |         |  760  | ExclusiveLock    |   true  |
| tuple         |               |         |       | ExclusiveLock    |   true  |
| transactionid |               |         |  759  | ShareLock        |  false  |

Добавляется новый режим разделяемой блокировки на транзакцию, потому что пришла новая транзакция и запрашивает ту же строку. Транзакция из сессии 2 генерирует новую версию строки, и накладывает на нее исключительную блокировку.

### Блокировки для сессии 3

|   locktype    |    relation   | virtxid |  xid  |      node        | granted |
|---------------|---------------|---------|-------|------------------|---------|
| relation      | accounts      |         |       | RowExclusiveLock |   true  |
| relation      | accounts_pkey |         |       | RowExclusiveLock |   true  |
| virtualxid    |               | 11/190  |       | ExclusiveLock    |   true  |
| transactionid |               |         |  761  | ExclusiveLock    |   true  |
| tuple         | accounts      |         |       | ExclusiveLock    |  false  |

Транзакция из сессии 3 добавляет блокировку на ожидание новой версии строки из сессии 2. Пока она не станет доступна - выполнение транзакции будет невозможно.

Когда выполняем коммит в 1 сессии, 3 сессия получает исключительные доступ к новой версии строки, вторая сессия окончательно понимает каким образом она будет менять данные в строке, 3 сессия ожидает завершения 2 сесиии, поэтому появяется блокировка ShareLock на транзакцию из 2 сессии (по аналогии с сессией 2).

Завершаем транзакцию в 2 и 3 сессии.


### Шаг 3. Взаимблокировки трех транзакций

> Воспроизведем ситуацию взаимоблокировок на примере трех транзакций. Логика простая: последовательно в каждой сессии будем выполнять изменение строки первой строчкой кода, а потом обращение к тому номеру строки, которую меняет следующая сессия.

### Результат

Команды, которые помогут нам воспроизвести ситуацию:

```sql
-- СЕССИЯ 1
BEGIN;

select txid_current(); --774
SELECT pg_backend_pid(); --1524

UPDATE accounts SET amount = amount + 1 WHERE acc_no = 1;
UPDATE accounts SET amount = amount + 1 WHERE acc_no = 2;

SELECT locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted
FROM pg_locks WHERE pid = 1524;

COMMIT;
END;


-- СЕССИЯ 2
BEGIN;

select txid_current(); -- 775
SELECT pg_backend_pid(); -- 1525

UPDATE accounts SET amount = amount + 10 WHERE acc_no = 2;
UPDATE accounts SET amount = amount + 10 WHERE acc_no = 3;

SELECT locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted
FROM pg_locks WHERE pid = 1525;

COMMIT;
END;


-- СЕССИЯ 3
BEGIN;

select txid_current(); -- 776
SELECT pg_backend_pid(); -- 1526

UPDATE accounts SET amount = amount + 100 WHERE acc_no = 3;
UPDATE accounts SET amount = amount + 100 WHERE acc_no = 1;

SELECT locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted
FROM pg_locks WHERE pid = 1526;

COMMIT;
END;
```
![Взаимоблокировка](/images/scr-dz09-03.png)

При выполнении кода в 3 сессии ловим ошибку (попытка изменения строки 1):

```
SQL Error [40P01]: ERROR: deadlock detected
  Detail: Process 1526 waits for ShareLock on transaction 774; blocked by process 1524.
Process 1524 waits for ShareLock on transaction 775; blocked by process 1525.
Process 1525 waits for ShareLock on transaction 776; blocked by process 1526.
  Hint: See server log for query details.
  Where: while updating tuple (0,18) in relation "accounts"
```

Через какое-то время сервер принимает решение октатить транзакцию в третьей сессии и оставляет работать первые 2 сессии, при этом 1 сессия остается в ожидании подтверждения транзаеции во 2 сессии.

**Как и говорили на занятии**: решение проблемы в данном случае - выбрать жертву из трех транзакций, как правило, последнюю из пришедших с запросом.


### Шаг 4. Взаимоблокировка транзакций, выполняющих комнду UPDATE одной и той же таблицы без where

> В теории, такая ситуация возможна. В интернете есть описание примеров как это можно попробовать сделать, но на практике повторить не получилось.