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

Проверяем репликацию:

1. На хосте node1 создаем базу данных test_otus:

```
[root@node1 ~]# sudo -u postgres psql                                                                                                                                                    
could not change directory to "/root": Permission denied                                                                                                                                 
psql (14.7)                                                                                                                                                                              
Type "help" for help.                                                                                                                                                                    
~                                                                                                                                                                                        
postgres=# CREATE DATABASE otus_test;                                                                                                                                                    
CREATE DATABASE
```

2. На хосте node2 выводим список баз данных:

```
[vagrant@node2 ~]$ sudo -u postgres psql
could not change directory to "/home/vagrant": Permission denied
psql (14.7)
Type "help" for help.

postgres=# \l
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges   
-----------+----------+----------+-------------+-------------+-----------------------
 otus      | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 otus_test | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
(5 rows)
```

База данных otus_test появилась на хосте node2

Далее проверяем статус репликации:

1. На хосте node1:

```
postgres=# select * from pg_stat_replication;                                                                                                                                            
  pid  | usesysid |   usename   |  application_name  |  client_addr  | client_hostname | client_port |         backend_start         | backend_xmin |   state   | sent_lsn  | write_lsn | flush_lsn | repla
y_lsn |    write_lag    |    flush_lag    |   replay_lag    | sync_priority | sync_state |          reply_time                                                                           
-------+----------+-------------+--------------------+---------------+-----------------+-------------+-------------------------------+--------------+-----------+-----------+-----------+-----------+------
------+-----------------+-----------------+-----------------+---------------+------------+-------------------------------                                                                
 29293 |    16384 | replication | walreceiver        | 192.168.57.12 |                 |       56200 | 2023-05-02 16:42:19.859616-03 |          739 | streaming | 0/4000AF0 | 0/4000AF0 | 0/4000AF0 | 0/400
0AF0  |                 |                 |                 |             0 | async      | 2023-05-02 16:49:02.064909-03                                                                 
 29297 |    16385 | barman      | barman_receive_wal | 192.168.57.13 |                 |       49572 | 2023-05-02 16:42:22.376179-03 |              | streaming | 0/4000AF0 | 0/4000AF0 | 0/4000000 |      
      | 00:00:03.141536 | 00:06:31.579195 | 00:06:42.267279 |             0 | async      | 2023-05-02 16:49:04.670899-03                                                                 
(2 rows)                                                                                                                                                                                 
~          
```

2. На хосте node2:

```
postgres=# select * from pg_stat_wal_receiver;
  pid  |  status   | receive_start_lsn | receive_start_tli | written_lsn | flushed_lsn | received_tli |      last_msg_send_time       |     last_msg_receipt_time     | latest_end_lsn |        latest_end_
time        | slot_name |  sender_host  | sender_port |                                                                                                                                         conninfo   
                                                                                                                                      
-------+-----------+-------------------+-------------------+-------------+-------------+--------------+-------------------------------+-------------------------------+----------------+-------------------
------------+-----------+---------------+-------------+----------------------------------------------------------------------------------------------------------------------------------------------------
--------------------------------------------------------------------------------------------------------------------------------------
 28671 | streaming | 0/3000000         |                 1 | 0/4000AF0   | 0/4000AF0   |            1 | 2023-05-02 16:49:41.671539-03 | 2023-05-02 16:49:41.674278-03 | 0/4000AF0      | 2023-05-02 16:48:1
1.483639-03 |           | 192.168.57.11 |        5432 | user=replication password=******** channel_binding=prefer dbname=replication host=192.168.57.11 port=5432 fallback_application_name=walreceiver ssl
mode=prefer sslcompression=0 sslsni=1 ssl_min_protocol_version=TLSv1.2 gssencmode=prefer krbsrvname=postgres target_session_attrs=any
(1 row)
```


Проверяем резервное копирование.


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
