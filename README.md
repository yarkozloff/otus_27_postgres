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

Т.к. firewalld оставляем включенным - его необходимо настроить, добавив в разрешение сервис postgresql. Порт будет использоваться по умолчанию 5432.

Далее после старта pg под пользователем postgres создаем пользователя для репликации, а также создаем базу с тестовой таблицей (т.к. в задании не указано нужно ли и откуда взять тестовую бд):
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
Аналогично мастеру устанавливаем необходимое ПО, настраиваем фаервол. Далее 
