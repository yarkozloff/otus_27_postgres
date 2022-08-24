# Репликация postgres

## Описание/Пошаговая инструкция выполнения домашнего задания:
- настроить hot_standby репликацию с использованием слотов
- настроить правильное резервное копирование

Для сдачи работы присылаем ссылку на репозиторий, в котором должны обязательно быть
- Vagranfile (2 машины)
- плейбук Ansible
- конфигурационные файлы postgresql.conf, pg_hba.conf и recovery.conf,
- конфиг barman, либо скрипт резервного копирования.
- Команда "vagrant up" должна поднимать машины с настроенной репликацией и резервным копированием.
- Рекомендуется в README.md файл вложить результаты (текст или скриншоты) проверки работы репликации и резервного копирования.

## Среда окружения
Основная машина:
```
root@yarkozloff:/otus/pg# hostnamectl | grep Operating System:
grep: System:: No such file or directory
root@yarkozloff:/otus/pg# hostnamectl | grep "Operating System"
  Operating System: Ubuntu 20.04.3 LTS
root@yarkozloff:/otus/pg# vagrant --version
Vagrant 2.2.19
root@yarkozloff:/otus/pg# ansible --version
ansible [core 2.12.4]
```
+ Локально загруженный vagrant box centos7

## Настройка мастера
Действия выполняются с использованием ansible. После поднятия машины обновляем yum cache, устанавливаем необходимое ПО: postgresql11-server, vim, nmap. Далее необходимо инициализировать бд. После настраиваются конфиги. В pg_hba.conf будет дополнительно прописан сервер-реплика и пользователь:
```
# TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
local   all             all                                     trust
# IPv4 local connections:
host    all             all             127.0.0.1/32            trust
# IPv6 local connections:
host    all             all             ::1/128                 trust
# Allow replication connections from localhost, by a user with the
# replication privilege.
local   replication     all                                     trust
host    replication     all             127.0.0.1/32            trust
host    replication     all             ::1/128                 trust
host    replication     repl_user       192.168.50.11/24        trust
```
В конфиге postgresql.conf прописываем параметр listen_addresses = '*'. Конфиг был взят по умолчанию, при необходимости индивидуальной настройки под имеющиеся ресурсы и специфики приложения можно обратиться к этому калькулятору: https://pgtune.leopard.in.ua/#/

Т.к. firewalld оставляем включенным - его необходимо настроить, добавив в разрешение сервис postgresql. Порт будет использоваться по умолчанию 5432. А также перезапустить демон.

Далее после старта pg под пользователем postgres создаем пользователя для репликации, а также создаем базу с тестовой таблицей (т.к. в задании нет указания на использование другой бд):
```
Create user repl_user with replication encrypted password 'Yarpg082022otus';
CREATE DATABASE yar99;
CREATE TABLE IF NOT EXISTS person (Id 	INT PRIMARY KEY, first_name VARCHAR(50) NOT NULL, last_name VARCHAR(50) NOT NULL, gender CHAR(1), email_id VARCHAR(100) UNIQUE, country_of_birth VARCHAR(50));
```
Теперь подключимя к машине (мастер) и добавим строку в новую таблицу, убедимся что всё работает:
```
root@yarkozloff:/otus/pg# vagrant ssh pgmaster
Last login: Wed Aug 17 19:05:47 2022 from 10.0.2.2
[vagrant@pgmaster ~]$ sudo -i
[root@pgmaster ~]# su postgres
bash-4.2$ psql
could not change directory to "/root": Permission denied
psql (11.17)
Type "help" for help.

postgres=# \l
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges
-----------+----------+----------+-------------+-------------+-----------------------
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 yar99     | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
(4 rows)

postgres=# \c yar99;
You are now connected to database "yar99" as user "postgres".
yar99=# insert into person (id, first_name, last_name, gender, email_id, country_of_birth) values (2,'andrew', 'morrow', 0, 'andrrr@gmail.com', 'usa');
INSERT 0 1
yar99=# select * from person;
 id | first_name | last_name | gender |     email_id     | country_of_birth
----+------------+-----------+--------+------------------+------------------
  2 | andrew     | morrow    | 0      | andrrr@gmail.com | usa
(1 row)
```
## Настройка слэйва
Аналогично мастеру устанавливаем необходимое ПО, настраиваем фаервол. Далее проверяем что сервис стартует, останавливаем его, удаляем всё из каталога /var/lib/pgsql/11/data/ чтобы восстановить туда резервную копию с мастера:
```
/usr/pgsql-11/bin/pg_basebackup -h 192.168.50.10 -U repl_user -p 5432 -D /var/lib/pgsql/11/data -Fp -Xs -P -R -C -S repl_slot
```
Если машины провижить не в первый раз то можно получить ошибку при создании слота репликации: replication slot already exists. Это связано с тем, что используются одинаковое имя слота. Для того чтобы создать с таким же именем слот можно удалить командой в мастере: SELECT pg_drop_replication_slot('repl_slot');

Далее каталогу /var/lib/pgsql/11/data необходимо выдать необходимые права и назначить владельца - postgres, иначе postgresql-11 не стартанет.

После старта сервиса можно подключаться к реплике и проверить тестовую таблицу:
```
root@yarkozloff:/otus/pg# vagrant ssh pgslave
Last login: Wed Aug 17 20:06:59 2022 from 10.0.2.2
[vagrant@pgslave ~]$ sudo -i
[root@pgslave ~]# su postgres
bash-4.2$ psql
could not change directory to "/root": Permission denied
psql (11.17)
Type "help" for help.

postgres=# \c yar99;
You are now connected to database "yar99" as user "postgres".
yar99=# select * from person;
 id | first_name | last_name | gender |     email_id     | country_of_birth
----+------------+-----------+--------+------------------+------------------
  2 | andrew     | morrow    | 0      | andrrr@gmail.com | usa
(1 row)
```

## Тестирвоание
Проверим репликацию с мастера:
```
yar99=# select * from pg_stat_replication;
-[ RECORD 1 ]----+-----------------------------
pid              | 22466
usesysid         | 16384
usename          | repl_user
application_name | walreceiver
client_addr      | 192.168.50.11
client_hostname  |
client_port      | 47364
backend_start    | 2022-08-17 20:07:01.60756+00
backend_xmin     |
state            | streaming
sent_lsn         | 0/6000910
write_lsn        | 0/6000910
flush_lsn        | 0/6000910
replay_lsn       | 0/6000910
write_lag        | 00:00:00.001656
flush_lag        | 00:00:00.003859
replay_lag       | 00:00:00.004293
sync_priority    | 0
sync_state       | async
```
Также Получим текущую позицию записи в журнале предзаписи
```
yar99=# select * from pg_current_wal_lsn();
-[ RECORD 1 ]------+----------
pg_current_wal_lsn | 0/6000A28
```
На pgslave проверим что она такая же:
```
yar99=# select * from pg_last_wal_receive_lsn();;
-[ RECORD 1 ]-----------+----------
pg_last_wal_receive_lsn | 0/6000A28
```
pgslave находится в режиме восстановления, 
Пробуем с мастера добавить еще строку в таблицу и понаблюдать в реплике:
![image](https://user-images.githubusercontent.com/69105791/185235181-834f62e1-3f58-4e33-a326-5763114cb9d5.png)
Если попробовать выполнить insert с сервера pgslave то получим ошибку: ERROR:  cannot execute INSERT in a read-only transaction. 

В текущем примере была рассмотрена асинхронная репликация. В случае с синхронной реплкацией в postgresql.conf необходимо задать synchronous_standby_names = '*'

это может потребовать перезапуска postgresql-11.service

## Barman
В связи со сложностью развертывания barman через ansible (стокнулся с проблемой прокидывания ssh ключей), сам менеджер бэкапов будет располагаться на сервере pgmaster, а каталог с бэкапом на удаленном сервере pgbackup; не советуют не промышлять таким. Таким образом, на отдельном сервере был поднят nfs-server с сетевым каталогом, который монтируется на pgmaster.

Для самого barman на сервере pgmaster дополнительно потребовалось:

- Добавить питон библиотеки в репозиторий:
```
- http://download-ib01.fedoraproject.org/pub/epel/7/x86_64/Packages/p/python2-argcomplete-1.7.0-4.el7.noarch.rpm
- http://download-ib01.fedoraproject.org/pub/epel/7/aarch64/Packages/p/python2-argh-0.26.1-5.el7.noarch.rpm
```
- Установить сам barman и nfs-utils

