Домашняя работа № 28. Postgres: Backup + Репликация

Задание:

Настроить hot_standby репликацию с использованием слотов

Настроить правильное резервное копирование

Ход работы:

В прилагаемом Vagrantfile описано создание трех виртуальных машин:

1. node1

2. node2

3. barman

После выполнения комманды vagrant up  и создания ВМ дальнейшая настройка ВМ происходит с помощью ansible







На сервере barman проверяем работу barman:
```
[barman@barman ~]$ barman check node1
Server node1:
	PostgreSQL: OK
	superuser or standard user with backup privileges: OK
	PostgreSQL streaming: OK
	wal_level: OK
	replication slot: OK
	directories: OK
	retention policy settings: OK
	backup maximum age: FAILED (interval provided: 4 days, latest backup age: No available backups)
	backup minimum size: OK (0 B)
	wal maximum age: OK (no last_wal_maximum_age provided)
	wal size: OK (0 B)
	compression settings: OK
	failed backups: OK (there are 0 failed backups)
	minimum redundancy requirements: FAILED (have 0 backups, expected at least 1)
	pg_basebackup: OK
	pg_basebackup compatible: OK
	pg_basebackup supports tablespaces mapping: OK
	systemid coherence: OK (no system Id stored on disk)
	pg_receivexlog: OK
	pg_receivexlog compatible: OK
	receive-wal running: OK
	archiver errors: OK
```
Создаем бэкап БД:
```
[barman@barman ~]$ barman backup node1
Starting backup using postgres method for server node1 in /var/lib/barman/node1/base/20230429T210632
Backup start at LSN: 0/5000060 (000000010000000000000005, 00000060)
Starting backup copy via pg_basebackup for 20230429T210632
Copy done (time: 1 second)
Finalising the backup.
This is the first backup for server node1
WAL segments preceding the current backup have been found:
	000000010000000000000003 from server node1 has been removed
	000000010000000000000004 from server node1 has been removed
Backup size: 33.5 MiB
Backup end at LSN: 0/7000000 (000000010000000000000006, 00000000)
Backup completed (start time: 2023-04-29 21:06:32.361940, elapsed time: 1 second)
Processing xlog segments from streaming for node1
	000000010000000000000005
	000000010000000000000006
```

```
[vagrant@node1 ~]$ sudo -u postgres psql
postgres=# drop database otus;
DROP DATABASE

```
На сервере barman выводим список бэкапов:
```
[barman@barman ~]$ barman list-backup node1
node1 20230429T210632 - Sat Apr 29 18:06:33 2023 - Size: 33.6 MiB - WAL Size: 0 B
```
Восстанавливаем БД из бэкапа:
```
[barman@barman ~]$ barman recover node1 20230429T210632 /var/lib/pgsql/14/data/ --remote-ssh-comman "ssh postgres@192.168.57.11"
The authenticity of host '192.168.57.11 (192.168.57.11)' can't be established.
ECDSA key fingerprint is SHA256:DWc+OknFpzLXGf4k3+o6BJDb+L9iYcmFm3Wu5E4qMyo.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Starting remote restore for server node1 using backup 20230429T210632
Destination directory: /var/lib/pgsql/14/data/
Remote command: ssh postgres@192.168.57.11
Copying the base backup.
Copying required WAL segments.
Generating archive status files
Identify dangerous settings in destination directory.

Recovery completed (start time: 2023-04-29 21:12:29.801727+00:00, elapsed time: 7 seconds)
Your PostgreSQL server has been successfully prepared for recovery!
```

После процедуры восстановления перезапускаем postgres на node1
```
[vagrant@node1 ~]$ service postgresql-14 restart
```
И, проверяем, восстановилась ли БД otus

```
[vagrant@node1 ~]$ sudo -u postgres psql
postgres=# \l
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges   
-----------+----------+----------+-------------+-------------+-----------------------
 otus      | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
(4 rows)

```
