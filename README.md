# pgcloud-homework03

## Домашнее задание
**Настройка дисков для Постгреса**

### Описание/Пошаговая инструкция выполнения домашнего задания:
ДЗ по прежнему выполняется на собственной ВМ / Hyper-v сервер

Создано 2 ВМ Ubuntu 20.04.4 LTS Ubuntu01 Ubuntu02 соответственно.
Установлен PostgreSQL 14

Проверяем, что кластер запущен через sudo -u postgres pg_lsclusters
```
#  sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
root@ubuntu01:/home/ujack# systemctl status postgresql
postgresql@14-main.service  postgresql.service
root@ubuntu01:/home/ujack# systemctl status postgresql@14-main.service
● postgresql@14-main.service - PostgreSQL Cluster 14-main
     Loaded: loaded (/lib/systemd/system/postgresql@.service; enabled-runtime; vendor preset: enabled)
     Active: active (running) since Wed 2022-05-25 13:14:46 UTC; 3min 58s ago
   Main PID: 3214 (postgres)
      Tasks: 7 (limit: 2195)
     Memory: 18.8M
     CGroup: /system.slice/system-postgresql.slice/postgresql@14-main.service
             ├─3214 /usr/lib/postgresql/14/bin/postgres -D /var/lib/postgresql/14/main -c config_file=/etc/postgresql/14/main/postgr>
             ├─3216 postgres: 14/main: checkpointer
             ├─3217 postgres: 14/main: background writer
             ├─3218 postgres: 14/main: walwriter
             ├─3219 postgres: 14/main: autovacuum launcher
             ├─3220 postgres: 14/main: stats collector
             └─3221 postgres: 14/main: logical replication launcher

May 25 13:14:44 ubuntu01 systemd[1]: Starting PostgreSQL Cluster 14-main...
May 25 13:14:46 ubuntu01 systemd[1]: Started PostgreSQL Cluster 14-main.
```

Зайдем из под пользователя postgres в psql и сделаем произвольную таблицу с произвольным содержимым
```
postgres=# create table test(c1 text); insert into test values('1'); \q
CREATE TABLE
INSERT 0 1
```
Остановим postgresql
```
root@ubuntu01:/home/ujack# systemctl stop postgresql@14-main.service
```
Создадим новый диск, смонтируем его в виртуалке. (sdb - 10Gb)
```
root@ubuntu01:/home/ujack# lsblk
NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
loop0                       7:0    0 61.9M  1 loop /snap/core20/1328
loop1                       7:1    0 67.2M  1 loop /snap/lxd/21835
loop2                       7:2    0 43.6M  1 loop /snap/snapd/14978
loop3                       7:3    0 44.7M  1 loop /snap/snapd/15904
loop4                       7:4    0 61.9M  1 loop /snap/core20/1494
loop5                       7:5    0 67.8M  1 loop /snap/lxd/22753
sda                         8:0    0   60G  0 disk
├─sda1                      8:1    0  1.1G  0 part /boot/efi
├─sda2                      8:2    0    2G  0 part /boot
└─sda3                      8:3    0   57G  0 part
  └─ubuntu--vg-ubuntu--lv 253:0    0   57G  0 lvm  /
sdb                         8:16   0   10G  0 disk
sr0                        11:0    1 1024M  0 rom
```
Создадим раздел, отформатируем, примонтируем
```
root@ubuntu01:/home/ujack# pvcreate /dev/sdb
  Physical volume "/dev/sdb" successfully created.
root@ubuntu01:/home/ujack# vgcreate vg_tmp_pgdata /dev/sdb
  Volume group "vg_tmp_pgdata" successfully created
root@ubuntu01:/home/ujack# lvcreate -n lv_tmp_pgdata -l +100%FREE vg_tmp_pgdata
  Logical volume "lv_tmp_pgdata" created.
root@ubuntu01:/home/ujack# mkfs.ext4 /dev/vg_tmp_pgdata/lv_tmp_pgdata
mke2fs 1.45.5 (07-Jan-2020)
Discarding device blocks: done
Creating filesystem with 2620416 4k blocks and 655360 inodes
Filesystem UUID: b65c61b7-a31f-431c-8db0-c27684104327
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632

Allocating group tables: done
Writing inode tables: done
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done

root@ubuntu01:/home/ujack# cat /etc/fstab
...
/dev/vg_tmp_pgdata/lv_tmp_pgdata /mnt/data ext4 defaults 0 1
...

root@ubuntu01:/home/ujack# df -h
Filesystem                               Size  Used Avail Use% Mounted on
udev                                     915M     0  915M   0% /dev
tmpfs                                    192M  960K  191M   1% /run
/dev/mapper/ubuntu--vg-ubuntu--lv         56G  4.7G   49G   9% /
tmpfs                                    959M     0  959M   0% /dev/shm
tmpfs                                    5.0M     0  5.0M   0% /run/lock
tmpfs                                    959M     0  959M   0% /sys/fs/cgroup
/dev/sda2                                2.0G  113M  1.7G   7% /boot
/dev/sda1                                1.1G  5.3M  1.1G   1% /boot/efi
/dev/loop2                                44M   44M     0 100% /snap/snapd/14978
/dev/loop0                                62M   62M     0 100% /snap/core20/1328
/dev/loop1                                68M   68M     0 100% /snap/lxd/21835
/dev/loop3                                45M   45M     0 100% /snap/snapd/15904
/dev/loop4                                62M   62M     0 100% /snap/core20/1494
/dev/loop5                                68M   68M     0 100% /snap/lxd/22753
tmpfs                                    192M     0  192M   0% /run/user/1000
/dev/mapper/vg_tmp_pgdata-lv_tmp_pgdata  9.8G   37M  9.3G   1% /mnt/data
```

Сделаем пользователя postgres владельцем **/mnt/data**
```
root@ubuntu01:/home/ujack# chown postgres: /mnt/data/
```
Перенесите содержимое /var/lib/postgres/14 в /mnt/data
```
root@ubuntu01:/home/ujack# mv /var/lib/postgresql/14 /mnt/data

root@ubuntu01:/home/ujack# ll /mnt/data/14/main/
total 88
drwx------ 19 postgres postgres 4096 May 26 13:01 ./
drwxr-xr-x  3 postgres postgres 4096 May 25 13:14 ../
drwx------  5 postgres postgres 4096 May 25 13:14 base/
drwx------  2 postgres postgres 4096 May 25 13:20 global/
drwx------  2 postgres postgres 4096 May 25 13:14 pg_commit_ts/
drwx------  2 postgres postgres 4096 May 25 13:14 pg_dynshmem/
drwx------  4 postgres postgres 4096 May 26 13:01 pg_logical/
drwx------  4 postgres postgres 4096 May 25 13:14 pg_multixact/
drwx------  2 postgres postgres 4096 May 25 13:14 pg_notify/
drwx------  2 postgres postgres 4096 May 25 13:14 pg_replslot/
drwx------  2 postgres postgres 4096 May 25 13:14 pg_serial/
drwx------  2 postgres postgres 4096 May 25 13:14 pg_snapshots/
drwx------  2 postgres postgres 4096 May 26 13:01 pg_stat/
drwx------  2 postgres postgres 4096 May 25 13:14 pg_stat_tmp/
drwx------  2 postgres postgres 4096 May 25 13:14 pg_subtrans/
drwx------  2 postgres postgres 4096 May 25 13:14 pg_tblspc/
drwx------  2 postgres postgres 4096 May 25 13:14 pg_twophase/
-rw-------  1 postgres postgres    3 May 25 13:14 PG_VERSION
drwx------  3 postgres postgres 4096 May 25 13:14 pg_wal/
drwx------  2 postgres postgres 4096 May 25 13:14 pg_xact/
-rw-------  1 postgres postgres   88 May 25 13:14 postgresql.auto.conf
-rw-------  1 postgres postgres  130 May 25 13:14 postmaster.opts
```
Все тут.

Попытаемся запустить кластер:
```
root@ubuntu01:/home/ujack# systemctl start postgresql@14-main.service
Job for postgresql@14-main.service failed because the service did not take the steps required by its unit configuration.
See "systemctl status postgresql@14-main.service" and "journalctl -xe" for details.
root@ubuntu01:/home/ujack# systemctl status postgresql@14-main.service
● postgresql@14-main.service - PostgreSQL Cluster 14-main
     Loaded: loaded (/lib/systemd/system/postgresql@.service; enabled-runtime; vendor preset: enabled)
     Active: failed (Result: protocol) since Thu 2022-05-26 13:28:23 UTC; 4s ago
    Process: 10144 ExecStart=/usr/bin/pg_ctlcluster --skip-systemctl-redirect 14-main start (code=exited, status=1/FAILURE)

May 26 13:28:23 ubuntu01 systemd[1]: Starting PostgreSQL Cluster 14-main...
May 26 13:28:23 ubuntu01 postgresql@14-main[10144]: Error: /var/lib/postgresql/14/main is not accessible or does not exist
May 26 13:28:23 ubuntu01 systemd[1]: postgresql@14-main.service: Can't open PID file /run/postgresql/14-main.pid (yet?) after start:>
May 26 13:28:23 ubuntu01 systemd[1]: postgresql@14-main.service: Failed with result 'protocol'.
May 26 13:28:23 ubuntu01 systemd[1]: Failed to start PostgreSQL Cluster 14-main.
```
Естественно, так как данных кластера нет - запуститься он не может.

Поменяем в файле postgresql.conf значение **data_directory** c
data_directory = '/var/lib/postgresql/14/main'
на
data_directory = '/mnt/data/14/main'
```
root@ubuntu01:/home/ujack# cat /etc/postgresql/14/main/postgresql.conf | grep data_dir
data_directory = '/mnt/data/14/main'            # use data in another directory
```
Попробуем теперь запустить наш кластер
```
root@ubuntu01:/home/ujack# systemctl start postgresql@14-main.service
root@ubuntu01:/home/ujack# systemctl status postgresql@14-main.service
● postgresql@14-main.service - PostgreSQL Cluster 14-main
     Loaded: loaded (/lib/systemd/system/postgresql@.service; enabled-runtime; vendor preset: enabled)
     Active: active (running) since Thu 2022-05-26 13:34:02 UTC; 5s ago
    Process: 10248 ExecStart=/usr/bin/pg_ctlcluster --skip-systemctl-redirect 14-main start (code=exited, status=0/SUCCESS)
   Main PID: 10262 (postgres)
      Tasks: 7 (limit: 2195)
     Memory: 22.3M
     CGroup: /system.slice/system-postgresql.slice/postgresql@14-main.service
             ├─10262 /usr/lib/postgresql/14/bin/postgres -D /mnt/data/14/main -c config_file=/etc/postgresql/14/main/postgresql.conf
             ├─10264 postgres: 14/main: checkpointer
             ├─10265 postgres: 14/main: background writer
             ├─10266 postgres: 14/main: walwriter
             ├─10267 postgres: 14/main: autovacuum launcher
             ├─10268 postgres: 14/main: stats collector
             └─10269 postgres: 14/main: logical replication launcher

May 26 13:34:00 ubuntu01 systemd[1]: Starting PostgreSQL Cluster 14-main...
May 26 13:34:02 ubuntu01 systemd[1]: Started PostgreSQL Cluster 14-main.
```
И проверим нашу табличку
```
root@ubuntu01:/home/ujack# sudo -u postgres psql
psql (14.3 (Ubuntu 14.3-1.pgdg20.04+1))
Type "help" for help.

postgres=# select * from test;
 c1
----
 1
(1 row)
```
Все на месте.
Done.
# **\***

Отмонтируем диск от текущей ВМ
```
root@ubuntu01:/home/ujack# systemctl stop postgresql@14-main.service
root@ubuntu01:/home/ujack# umount /mnt/data
root@ubuntu01:/home/ujack# df -h
Filesystem                         Size  Used Avail Use% Mounted on
udev                               915M     0  915M   0% /dev
tmpfs                              192M  960K  191M   1% /run
/dev/mapper/ubuntu--vg-ubuntu--lv   56G  4.7G   49G   9% /
tmpfs                              959M     0  959M   0% /dev/shm
tmpfs                              5.0M     0  5.0M   0% /run/lock
tmpfs                              959M     0  959M   0% /sys/fs/cgroup
/dev/sda2                          2.0G  113M  1.7G   7% /boot
/dev/sda1                          1.1G  5.3M  1.1G   1% /boot/efi
/dev/loop2                          44M   44M     0 100% /snap/snapd/14978
/dev/loop0                          62M   62M     0 100% /snap/core20/1328
/dev/loop1                          68M   68M     0 100% /snap/lxd/21835
/dev/loop3                          45M   45M     0 100% /snap/snapd/15904
/dev/loop4                          62M   62M     0 100% /snap/core20/1494
/dev/loop5                          68M   68M     0 100% /snap/lxd/22753
tmpfs                              192M     0  192M   0% /run/user/1000
```
Переклюим диск на новую ВМ. **ubuntu02**
```
root@ubuntu02:/home/ujack# lsblk
NAME                          MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
loop1                           7:1    0 67.2M  1 loop /snap/lxd/21835
loop2                           7:2    0 43.6M  1 loop /snap/snapd/14978
loop3                           7:3    0 44.7M  1 loop /snap/snapd/15904
loop4                           7:4    0 61.9M  1 loop /snap/core20/1434
loop5                           7:5    0 67.8M  1 loop /snap/lxd/22753
loop6                           7:6    0 61.9M  1 loop /snap/core20/1494
sda                             8:0    0   60G  0 disk
├─sda1                          8:1    0  1.1G  0 part /boot/efi
├─sda2                          8:2    0    2G  0 part /boot
└─sda3                          8:3    0   57G  0 part
  └─ubuntu--vg-ubuntu--lv     253:0    0   57G  0 lvm  /
sdb                             8:16   0   10G  0 disk
└─vg_tmp_pgdata-lv_tmp_pgdata 253:1    0   10G  0 lvm
sr0                            11:0    1 1024M  0 rom
```
Поставим постгрес
```
root@ubuntu02:/home/ujack# apt install postgresql-14
...
root@ubuntu02:/home/ujack# systemctl status postgresql@14-main.service
● postgresql@14-main.service - PostgreSQL Cluster 14-main
     Loaded: loaded (/lib/systemd/system/postgresql@.service; enabled-runtime; vendor preset: enabled)
     Active: active (running) since Thu 2022-05-26 13:43:22 UTC; 36s ago
   Main PID: 6510 (postgres)
      Tasks: 7 (limit: 2195)
     Memory: 18.7M
     CGroup: /system.slice/system-postgresql.slice/postgresql@14-main.service
             ├─6510 /usr/lib/postgresql/14/bin/postgres -D /var/lib/postgresql/14/main -c config_file=/etc/postgresql/14/main/postgr>
             ├─6512 postgres: 14/main: checkpointer
             ├─6513 postgres: 14/main: background writer
             ├─6514 postgres: 14/main: walwriter
             ├─6515 postgres: 14/main: autovacuum launcher
             ├─6516 postgres: 14/main: stats collector
             └─6517 postgres: 14/main: logical replication launcher

May 26 13:43:20 ubuntu02 systemd[1]: Starting PostgreSQL Cluster 14-main...
May 26 13:43:22 ubuntu02 systemd[1]: Started PostgreSQL Cluster 14-main.
```
Остановим и очистим папку с данными
```
root@ubuntu02:/home/ujack# systemctl stop postgresql@14-main.service
root@ubuntu02:/home/ujack# rm -rf /var/lib/postgresql/*
root@ubuntu02:/home/ujack# ll /var/lib/postgresql/
total 8
drwxr-xr-x  2 postgres postgres 4096 May 26 13:44 ./
drwxr-xr-x 43 root     root     4096 May 26 13:43 ../
```
Примонтируем наш диск
```
root@ubuntu02:/home/ujack# mount /dev/vg_tmp_pgdata/lv_tmp_pgdata /var/lib/postgresql/
root@ubuntu02:/home/ujack# ll /var/lib/postgresql/
total 28
drwxr-xr-x  4 postgres postgres  4096 May 26 13:22 ./
drwxr-xr-x 43 root     root      4096 May 26 13:43 ../
drwxr-xr-x  3 postgres postgres  4096 May 25 13:14 14/
drwx------  2 root     root     16384 May 26 13:11 lost+found/
```
Ну, и попробуем запустить, в конфиге ничего менять дефолтном не будем, так как путь к папке с данными мы сохранили
```
root@ubuntu02:/home/ujack# systemctl start postgresql@14-main.service
root@ubuntu02:/home/ujack# systemctl status postgresql@14-main.service
● postgresql@14-main.service - PostgreSQL Cluster 14-main
     Loaded: loaded (/lib/systemd/system/postgresql@.service; enabled-runtime; vendor preset: enabled)
     Active: active (running) since Thu 2022-05-26 13:47:44 UTC; 4s ago
    Process: 7855 ExecStart=/usr/bin/pg_ctlcluster --skip-systemctl-redirect 14-main start (code=exited, status=0/SUCCESS)
   Main PID: 7872 (postgres)
      Tasks: 7 (limit: 2195)
     Memory: 20.0M
     CGroup: /system.slice/system-postgresql.slice/postgresql@14-main.service
             ├─7872 /usr/lib/postgresql/14/bin/postgres -D /var/lib/postgresql/14/main -c config_file=/etc/postgresql/14/main/postgr>
             ├─7874 postgres: 14/main: checkpointer
             ├─7875 postgres: 14/main: background writer
             ├─7876 postgres: 14/main: walwriter
             ├─7877 postgres: 14/main: autovacuum launcher
             ├─7878 postgres: 14/main: stats collector
             └─7879 postgres: 14/main: logical replication launcher

May 26 13:47:42 ubuntu02 systemd[1]: Starting PostgreSQL Cluster 14-main...
May 26 13:47:44 ubuntu02 systemd[1]: Started PostgreSQL Cluster 14-main.
root@ubuntu02:/home/ujack# sudo -u postgres psql
psql (14.3 (Ubuntu 14.3-1.pgdg20.04+1))
Type "help" for help.

postgres=# select * from test ;
 c1
----
 1
(1 row)
```

Done **\***
