Домашнее задание по теме: Резервное копирование.

Задание:

	Настроить стенд Vagrant с двумя виртуальными машинами: backup_server и client. (Студент самостоятельно настраивает Vagrant)
	Настроить удаленный бэкап каталога /etc c сервера client при помощи borgbackup. Резервные копии должны соответствовать следующим критериям:
директория для резервных копий /var/backup. Это должна быть отдельная точка монтирования. В данном случае для демонстрации размер не принципиален, достаточно будет и 2GB; (Студент самостоятельно настраивает)
репозиторий для резервных копий должен быть зашифрован ключом или паролем - на ваше усмотрение;
имя бэкапа должно содержать информацию о времени снятия бекапа;
глубина бекапа должна быть год, хранить можно по последней копии на конец месяца, кроме последних трех. Последние три месяца должны содержать копии на каждый день. Т.е. должна быть правильно настроена политика удаления старых бэкапов;
резервная копия снимается каждые 5 минут. Такой частый запуск в целях демонстрации;
написан скрипт для снятия резервных копий. Скрипт запускается из соответствующей Cron джобы, либо systemd timer-а - на ваше усмотрение;
настроено логирование процесса бекапа. Для упрощения можно весь вывод перенаправлять в logger с соответствующим тегом. Если настроите не в syslog, то обязательна ротация логов.

1. Пишем Vagrantfile, в котором создаются 2 ВМ на Centos 7: backup и client, в ВМ backup дополнительно добавляется диск на 2 Гб для хранения резервных копий.
На обе ВМ устанавливаем Borg Backup и создаём пользователя borg, делаем синхронизацию времени по протоколу ntp. Всё это делается через Vagrantfile  и запускаем vagrant up.


```
neva@Uneva:~$ vagrant status
Current machine states:

backup                    running (virtualbox)
client                    running (virtualbox)
```

Подключаемся к серверу бэкапов, создаем раздел и файловую систему ext4 на дополнительном диске в 2 Гб, создаем точку монтирования /var/backup и монтируем к ней диск и и назначаем на него права пользователя borg:

```
neva@Uneva:~$ vagrant ssh backup
[vagrant@backup-server ~]$ lsblk
NAME MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
sda    8:0    0  40G  0 disk
sda1   8:1    0  40G  0 part /
sdb    8:16   0   2G  0 disk

[root@backup-server ~]# fdisk /dev/sdb
Welcome to fdisk (util-linux 2.23.2).

Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table
Building a new DOS disklabel with disk identifier 0x25c0882a.

Command (m for help): m
Command action
   a   toggle a bootable flag
   b   edit bsd disklabel
   c   toggle the dos compatibility flag
   d   delete a partition
   g   create a new empty GPT partition table
   G   create an IRIX (SGI) partition table
   l   list known partition types
   m   print this menu
   n   add a new partition
   o   create a new empty DOS partition table
   p   print the partition table
   q   quit without saving changes
   s   create a new empty Sun disklabel
   t   change a partition's system id
   u   change display/entry units
   v   verify the partition table
   w   write table to disk and exit
   x   extra functionality (experts only)

Command (m for help): n
Partition type:
   p   primary (0 primary, 0 extended, 4 free)
   e   extended
Select (default p): p
Partition number (1-4, default 1): 1
First sector (2048-4194303, default 2048):
Using default value 2048
Last sector, +sectors or +size{K,M,G} (2048-4194303, default 4194303):
Using default value 4194303
Partition 1 of type Linux and of size 2 GiB is set

Command (m for help): p

Disk /dev/sdb: 2147 MB, 2147483648 bytes, 4194304 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x25c0882a

   Device Boot      Start         End      Blocks   Id  System
/dev/sdb1            2048     4194303     2096128   83  Linux

Command (m for help): w
The partition table has been altered!

Calling ioctl() to re-read partition table.
Syncing disks.

[root@backup-server ~]# lsblk
NAME   MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
sda      8:0    0  40G  0 disk
`-sda1   8:1    0  40G  0 part /
sdb      8:16   0   2G  0 disk
`-sdb1   8:17   0   2G  0 part

[root@backup-server ~]# mkfs.ext4 /dev/sdb1
mke2fs 1.42.9 (28-Dec-2013)
Filesystem label=
OS type: Linux
Block size=4096 (log=2)
Fragment size=4096 (log=2)
Stride=0 blocks, Stripe width=0 blocks
131072 inodes, 524032 blocks
26201 blocks (5.00%) reserved for the super user
First data block=0
Maximum filesystem blocks=536870912
16 block groups
32768 blocks per group, 32768 fragments per group
8192 inodes per group
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912

Allocating group tables: done
Writing inode tables: done
Creating journal (8192 blocks): done
Writing superblocks and filesystem accounting information: done

[root@backup-server ~]# mount /dev/sdb1 /var/backup/
[root@backup-server ~]# df -h
Filesystem      Size  Used Avail Use% Mounted on
devtmpfs        489M     0  489M   0% /dev
tmpfs           496M     0  496M   0% /dev/shm
tmpfs           496M  6.7M  489M   2% /run
tmpfs           496M     0  496M   0% /sys/fs/cgroup
/dev/sda1        40G  7.7G   33G  20% /
tmpfs           100M     0  100M   0% /run/user/1000
/dev/sdb1       2.0G  6.0M  1.9G   1% /var/backup

[root@backup-server ~]# chown borg:borg /var/backup/
```

Далее необходимо включить авторизацию по ssh ключам. Заходим на клиент и создаём ключи:

```
[borg@client ~]$ ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/home/borg/.ssh/id_rsa):
Created directory '/home/borg/.ssh'.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /home/borg/.ssh/id_rsa.
Your public key has been saved in /home/borg/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:/DtkJucmySh8tOWfMWVwZ/sN4e6iqb99Z1NlZeC83HY borg@client
The key's randomart image is:
+---[RSA 2048]----+
|              .. |
|             o  o|
|          . . *..|
|       .   o = =o|
|        S   o *.E|
|      . o.=o . ++|
|   . . * O+   . +|
|    o + = +B....o|
|     o  .*O+.o.o.|
+----[SHA256]-----+

```
 На сервере бэкапов создаем директорию .ssh и в ней файл authorized_keys, предоставляем права доступа:

```
[borg@backup-server ~]$ mkdir .ssh
[borg@backup-server ~]$ touch .ssh/authorized_keys
[borg@backup-server ~]$ chmod 700 .ssh
[borg@backup-server ~]$ chmod 600 .ssh/authorized_keys
```
Далее копируем в этот файл публичный ключ для подключения (предварительно прописав на клиенте в файле /etc/hosts адрес сервера бэкапов 192.168.56.20 backup):

```
[borg@client ~]$ ssh-copy-id borg@backup
/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/home/borg/.ssh/id_rsa.pub"
The authenticity of host 'backup (192.168.56.20)' can't be established.
ECDSA key fingerprint is SHA256:SzAMhuNjGjyG1Kxi5riBnuFHVmkoA0djBjixivv/fr8.
ECDSA key fingerprint is MD5:79:4b:66:9b:90:a8:21:29:80:25:c6:97:13:5c:03:14.
Are you sure you want to continue connecting (yes/no)? y
Please type 'yes' or 'no': yes
/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
borg@backup's password:

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'borg@backup'"
and check to make sure that only the key(s) you wanted were added.
```

Пробуем подключиться:

```
[borg@client ~]$ ssh borg@backup
Last login: Wed Aug 16 13:03:25 2023
[borg@backup-server ~]$
```

Инициализируем репозиторий borg, в котором будут храниться бэкапы, на backup сервере с client сервера:

```
[borg@client ~]$ borg init --encryption=repokey borg@192.168.56.20:/var/backup/
Enter new passphrase:
Enter same passphrase again:
```

Запустим проверку создания бэкапа:

```
[borg@client ~]$ borg create --stats --list borg@192.168.56.20:/var/backup/::"etc-{now:%Y-%m-%d_%H:%M:%S}" /etc

------------------------------------------------------------------------------
Archive name: etc-2023-08-17_09:56:57
Archive fingerprint: cf4a80c6b1cf261417012dabf35b7a724f2325ede279d858697e01a98f239ed8
Time (start): Thu, 2023-08-17 09:57:05
Time (end):   Thu, 2023-08-17 09:57:07
Duration: 1.97 seconds
Number of files: 417
Utilization of max. archive size: 0%
------------------------------------------------------------------------------
                       Original size      Compressed size    Deduplicated size
This archive:               17.45 MB              5.86 MB              5.85 MB
All archives:               17.45 MB              5.86 MB              5.85 MB

                       Unique chunks         Total chunks
Chunk index:                     410                  416
------------------------------------------------------------------------------
```

Смотрим, что у нас получилось

```
[borg@client ~]$ borg list borg@192.168.56.20:/var/backup/
Enter passphrase for key ssh://borg@192.168.56.20/var/backup:
etc-2023-08-17_09:56:57              Thu, 2023-08-17 09:57:05 [cf4a80c6b1cf261417012dabf35b7a724f2325ede279d858697e01a98f239ed8]
```

Смотрим список файлов:

```
[borg@client ~]$ borg list borg@192.168.56.20:/var/backup/::etc-2023-08-17_09:56:57
Enter passphrase for key ssh://borg@192.168.56.20/var/backup:
Enter passphrase for key ssh://borg@192.168.56.20/var/backup:
drwxr-xr-x root   root          0 Wed, 2023-08-16 16:31:10 etc
lrwxrwxrwx root   root         35 Tue, 2023-08-15 17:03:32 etc/localtime -> ../usr/share/zoneinfo/Europe/Moscow
lrwxrwxrwx root   root         17 Fri, 2020-05-01 01:04:55 etc/mtab -> /proc/self/mounts
-rw-r--r-- root   root      12288 Tue, 2023-08-15 17:03:12 etc/aliases.db
-rw-r--r-- root   root       2388 Fri, 2020-05-01 01:08:36 etc/libuser.conf
-rw-r--r-- root   root       2043 Fri, 2020-05-01 01:08:36 etc/login.defs
-rw-r--r-- root   root         37 Fri, 2020-05-01 01:08:36 etc/vconsole.conf
-rw-r--r-- root   root         19 Fri, 2020-05-01 01:08:36 etc/locale.conf
-rw-r--r-- root   root        450 Tue, 2023-08-15 17:03:31 etc/fstab
-rw-r--r-- root   root        566 Tue, 2023-08-15 17:04:14 etc/group
drwxr-xr-x root   root          0 Tue, 2023-08-15 17:04:00 etc/ntp
-rw-r--r-- root   root         74 Wed, 2019-11-27 19:47:41 etc/ntp/step-tickers
drwxr-x--- root   ntp           0 Tue, 2023-08-15 17:04:00 etc/ntp/crypto
-rw-r--r-- root   root        163 Fri, 2020-05-01 01:05:05 etc/.updated
-rw-r--r-- root   root          0 Wed, 2020-04-01 07:29:33 etc/subuid-
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:26 etc/X11
drwxr-xr-x root   root          0 Tue, 2020-04-07 17:38:10 etc/X11/xorg.conf.d
drwxr-xr-x root   root          0 Wed, 2018-04-11 07:59:55 etc/X11/applnk
drwxr-xr-x root   root          0 Wed, 2018-04-11 07:59:55 etc/X11/fontpath.d
drwxr-xr-x root   root          0 Wed, 2020-04-01 07:21:38 etc/rpm
-rw-r--r-- root   root         66 Wed, 2020-04-08 01:01:12 etc/rpm/macros.dist
-rw-r--r-- root   root         37 Wed, 2020-04-08 01:01:12 etc/centos-release
-rw-r--r-- root   root         51 Wed, 2020-04-08 01:01:12 etc/centos-release-upstream
-rw-r--r-- root   root         23 Wed, 2020-04-08 01:01:12 etc/issue
-rw-r--r-- root   root         22 Wed, 2020-04-08 01:01:12 etc/issue.net
lrwxrwxrwx root   root         21 Fri, 2020-05-01 01:05:05 etc/os-release -> ../usr/lib/os-release
lrwxrwxrwx root   root         14 Fri, 2020-05-01 01:05:05 etc/redhat-release -> centos-release
lrwxrwxrwx root   root         14 Fri, 2020-05-01 01:05:05 etc/system-release -> centos-release
-rw-r--r-- root   root         23 Wed, 2020-04-08 01:01:12 etc/system-release-cpe
-rw-r--r-- root   root       1529 Wed, 2020-04-01 07:29:32 etc/aliases
-rw-r--r-- root   root       2853 Wed, 2020-04-01 07:29:31 etc/bashrc
-rw-r--r-- root   root       1620 Wed, 2020-04-01 07:29:31 etc/csh.cshrc
-rw-r--r-- root   root       1103 Wed, 2020-04-01 07:29:32 etc/csh.login
-rw-r--r-- root   root          0 Wed, 2020-04-01 07:29:33 etc/environment
-rw-r--r-- root   root          0 Fri, 2013-06-07 18:31:32 etc/exports
-rw-r--r-- root   root         70 Wed, 2020-04-01 07:29:31 etc/filesystems
-rw-r--r-- root   root          9 Fri, 2013-06-07 18:31:32 etc/host.conf
-rw-r--r-- root   root        370 Fri, 2013-06-07 18:31:32 etc/hosts.allow
-rw-r--r-- root   root        460 Fri, 2013-06-07 18:31:32 etc/hosts.deny
-rw-r--r-- root   root        942 Fri, 2013-06-07 18:31:32 etc/inputrc
-rw-r--r-- root   root          0 Fri, 2013-06-07 18:31:32 etc/motd
-rw-r--r-- root   root        233 Fri, 2013-06-07 18:31:32 etc/printcap
-rw-r--r-- root   root       1819 Wed, 2020-04-01 07:29:31 etc/profile
-rw-r--r-- root   root       6545 Wed, 2020-04-01 07:29:32 etc/protocols
-rw-r--r-- root   root     670293 Fri, 2013-06-07 18:31:32 etc/services
lrwxrwxrwx root   root         13 Fri, 2020-05-01 01:06:26 etc/rc.local -> rc.d/rc.local
-rw-r--r-- root   root         44 Wed, 2020-04-01 07:29:32 etc/shells
drwxr-xr-x root   root          0 Wed, 2018-04-11 07:59:55 etc/opt
-rw-r--r-- root   root         28 Thu, 2013-02-28 00:29:02 etc/ld.so.conf
-rw-r--r-- root   root       1938 Wed, 2020-04-01 01:09:52 etc/nsswitch.conf.bak
-rw-r--r-- root   root       1634 Tue, 2012-12-25 07:02:13 etc/rpc
drwxr-xr-x root   root          0 Tue, 2014-06-10 08:03:22 etc/popt.d
lrwxrwxrwx root   root         11 Fri, 2020-05-01 01:05:17 etc/init.d -> rc.d/init.d
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:26 etc/rc.d
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:33 etc/rc.d/rc2.d
lrwxrwxrwx root   root         17 Fri, 2020-05-01 01:06:33 etc/rc.d/rc2.d/S10network -> ../init.d/network
lrwxrwxrwx root   root         20 Fri, 2020-05-01 01:06:33 etc/rc.d/rc2.d/K50netconsole -> ../init.d/netconsole
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:33 etc/rc.d/rc6.d
lrwxrwxrwx root   root         17 Fri, 2020-05-01 01:06:33 etc/rc.d/rc6.d/K90network -> ../init.d/network
lrwxrwxrwx root   root         20 Fri, 2020-05-01 01:06:33 etc/rc.d/rc6.d/K50netconsole -> ../init.d/netconsole
-rw-r--r-- root   root        473 Tue, 2020-04-07 17:38:10 etc/rc.d/rc.local
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:33 etc/rc.d/init.d
-rw-r--r-- root   root       1160 Tue, 2020-04-07 17:38:10 etc/rc.d/init.d/README
-rw-r--r-- root   root      18281 Mon, 2019-08-19 14:34:03 etc/rc.d/init.d/functions
-rwxr-xr-x root   root       4569 Mon, 2019-08-19 14:34:03 etc/rc.d/init.d/netconsole
-rwxr-xr-x root   root       7928 Mon, 2019-08-19 14:34:03 etc/rc.d/init.d/network
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:33 etc/rc.d/rc3.d
lrwxrwxrwx root   root         17 Fri, 2020-05-01 01:06:33 etc/rc.d/rc3.d/S10network -> ../init.d/network
lrwxrwxrwx root   root         20 Fri, 2020-05-01 01:06:33 etc/rc.d/rc3.d/K50netconsole -> ../init.d/netconsole
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:33 etc/rc.d/rc0.d
lrwxrwxrwx root   root         17 Fri, 2020-05-01 01:06:33 etc/rc.d/rc0.d/K90network -> ../init.d/network
lrwxrwxrwx root   root         20 Fri, 2020-05-01 01:06:33 etc/rc.d/rc0.d/K50netconsole -> ../init.d/netconsole
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:33 etc/rc.d/rc4.d
lrwxrwxrwx root   root         17 Fri, 2020-05-01 01:06:33 etc/rc.d/rc4.d/S10network -> ../init.d/network
lrwxrwxrwx root   root         20 Fri, 2020-05-01 01:06:33 etc/rc.d/rc4.d/K50netconsole -> ../init.d/netconsole
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:33 etc/rc.d/rc1.d
lrwxrwxrwx root   root         17 Fri, 2020-05-01 01:06:33 etc/rc.d/rc1.d/K90network -> ../init.d/network
lrwxrwxrwx root   root         20 Fri, 2020-05-01 01:06:33 etc/rc.d/rc1.d/K50netconsole -> ../init.d/netconsole
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:33 etc/rc.d/rc5.d
lrwxrwxrwx root   root         17 Fri, 2020-05-01 01:06:33 etc/rc.d/rc5.d/S10network -> ../init.d/network
lrwxrwxrwx root   root         20 Fri, 2020-05-01 01:06:33 etc/rc.d/rc5.d/K50netconsole -> ../init.d/netconsole
lrwxrwxrwx root   root         10 Fri, 2020-05-01 01:05:17 etc/rc0.d -> rc.d/rc0.d
lrwxrwxrwx root   root         10 Fri, 2020-05-01 01:05:17 etc/rc1.d -> rc.d/rc1.d
lrwxrwxrwx root   root         10 Fri, 2020-05-01 01:05:17 etc/rc2.d -> rc.d/rc2.d
lrwxrwxrwx root   root         10 Fri, 2020-05-01 01:05:17 etc/rc3.d -> rc.d/rc3.d
lrwxrwxrwx root   root         10 Fri, 2020-05-01 01:05:17 etc/rc4.d -> rc.d/rc4.d
lrwxrwxrwx root   root         10 Fri, 2020-05-01 01:05:17 etc/rc5.d -> rc.d/rc5.d
lrwxrwxrwx root   root         10 Fri, 2020-05-01 01:05:17 etc/rc6.d -> rc.d/rc6.d
-rw-r--r-- root   root         94 Fri, 2017-03-24 19:39:09 etc/GREP_COLORS
-rw-r--r-- root   root         50 Tue, 2023-08-15 17:03:30 etc/resolv.conf
-rw-r--r-- root   root          7 Tue, 2023-08-15 17:03:26 etc/hostname
-rw-r--r-- root   root        203 Wed, 2023-08-16 16:31:10 etc/hosts
-rw-r--r-- root   root       2000 Wed, 2019-11-27 19:47:41 etc/ntp.conf
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:05:22 etc/libnl
-rw-r--r-- root   root       1130 Thu, 2017-08-03 22:48:41 etc/libnl/classid
-rw-r--r-- root   root       1532 Thu, 2017-08-03 22:48:41 etc/libnl/pktloc
-rw-r--r-- root   root        111 Wed, 2020-04-01 05:34:27 etc/magic
-rw-r--r-- root   root       1787 Tue, 2014-06-10 06:17:54 etc/request-key.conf
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:53 etc/request-key.d
-rw-r--r-- root   root        354 Wed, 2020-04-01 06:55:43 etc/request-key.d/id_resolver.conf
-rw-r--r-- root   root         50 Thu, 2017-08-03 19:58:31 etc/request-key.d/cifs.idmap.conf
-rw-r--r-- root   root         52 Thu, 2017-08-03 19:58:31 etc/request-key.d/cifs.spnego.conf
-rw-r--r-- root   root         38 Tue, 2018-10-30 17:59:46 etc/fuse.conf
-rw-r--r-- root   root       1982 Fri, 2019-08-09 06:17:16 etc/virc
-rw-r--r-- root   root       4669 Tue, 2019-08-06 16:44:33 etc/DIR_COLORS.lightbgcolor
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:05:52 etc/python
-rw-r--r-- root   root        381 Thu, 2020-04-02 16:17:15 etc/python/cert-verification.cfg
-rw-r--r-- root   root        767 Fri, 2019-08-09 03:35:22 etc/netconfig
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:07:03 etc/cron.daily
-rwxr-xr-x root   root        618 Tue, 2018-10-30 17:55:19 etc/cron.daily/man-db.cron
-rw-r--r-- root   root        662 Wed, 2013-07-31 15:46:23 etc/logrotate.conf
-rw-r--r-- root   root       4849 Wed, 2018-04-11 07:07:10 etc/idmapd.conf
-rw-r--r-- root   root        570 Wed, 2019-11-27 19:24:56 etc/my.cnf
-rw-r--r-- root   root        553 Tue, 2023-08-15 17:04:00 etc/group-
-rw-r--r-- root   root        970 Thu, 2020-04-02 18:56:34 etc/yum.conf
-rw-r--r-- root   root       1285 Wed, 2020-04-01 05:28:10 etc/dracut.conf
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:09:26 etc/modprobe.d
-rw-r--r-- root   root         17 Fri, 2020-05-01 01:09:26 etc/modprobe.d/nofloppy.conf
-rw-r--r-- root   root        215 Wed, 2020-04-01 02:46:17 etc/modprobe.d/dccp-blacklist.conf
-rw-r--r-- root   root        674 Fri, 2019-03-22 01:10:46 etc/modprobe.d/tuned.conf
-rw-r--r-- root   root        166 Tue, 2020-04-07 17:37:15 etc/modprobe.d/firewalld-sysctls.conf
-rw-r--r-- root   root        746 Wed, 2020-04-01 06:55:43 etc/modprobe.d/lockd.conf
drwxr-xr-x root   root          0 Wed, 2020-04-01 07:27:05 etc/rsyslog.d
-rw-r--r-- root   root         49 Tue, 2020-04-07 17:38:10 etc/rsyslog.d/listen.conf
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:26 etc/systemd
-rw-r--r-- root   root        720 Tue, 2020-04-07 17:38:09 etc/systemd/bootchart.conf
-rw-r--r-- root   root        615 Tue, 2020-04-07 17:38:09 etc/systemd/coredump.conf
-rw-r--r-- root   root        983 Tue, 2020-04-07 17:38:09 etc/systemd/journald.conf
-rw-r--r-- root   root        957 Tue, 2020-04-07 17:38:09 etc/systemd/logind.conf
-rw-r--r-- root   root       1552 Tue, 2020-04-07 17:38:09 etc/systemd/system.conf
-rw-r--r-- root   root       1127 Tue, 2020-04-07 17:38:09 etc/systemd/user.conf
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:08:36 etc/systemd/system
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:28 etc/systemd/system/system-update.target.wants
lrwxrwxrwx root   root         54 Fri, 2020-05-01 01:06:28 etc/systemd/system/system-update.target.wants/systemd-readahead-drop.service -> /usr/lib/systemd/system/systemd-readahead-drop.service
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:32 etc/systemd/system/sockets.target.wants
lrwxrwxrwx root   root         38 Fri, 2020-05-01 01:06:32 etc/systemd/system/sockets.target.wants/rpcbind.socket -> /usr/lib/systemd/system/rpcbind.socket
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:33 etc/systemd/system/basic.target.wants
lrwxrwxrwx root   root         42 Fri, 2020-05-01 01:06:33 etc/systemd/system/basic.target.wants/rhel-dmesg.service -> /usr/lib/systemd/system/rhel-dmesg.service
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:44 etc/systemd/system/network-online.target.wants
lrwxrwxrwx root   root         58 Fri, 2020-05-01 01:06:44 etc/systemd/system/network-online.target.wants/NetworkManager-wait-online.service -> /usr/lib/systemd/system/NetworkManager-wait-online.service
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:53 etc/systemd/system/remote-fs.target.wants
lrwxrwxrwx root   root         41 Fri, 2020-05-01 01:06:53 etc/systemd/system/remote-fs.target.wants/nfs-client.target -> /usr/lib/systemd/system/nfs-client.target
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:54 etc/systemd/system/vmtoolsd.service.requires
lrwxrwxrwx root   root         45 Fri, 2020-05-01 01:06:54 etc/systemd/system/vmtoolsd.service.requires/vmtoolsd-init.service -> /usr/lib/systemd/system/vmtoolsd-init.service
lrwxrwxrwx root   root         39 Fri, 2020-05-01 01:06:54 etc/systemd/system/vmtoolsd.service.requires/vgauthd.service -> /usr/lib/systemd/system/vgauthd.service
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:07:01 etc/systemd/system/dev-virtio\x2dports-org.qemu.guest_agent.0.device.wants
lrwxrwxrwx root   root         48 Fri, 2020-05-01 01:07:01 etc/systemd/system/dev-virtio\x2dports-org.qemu.guest_agent.0.device.wants/qemu-guest-agent.service -> /usr/lib/systemd/system/qemu-guest-agent.service
lrwxrwxrwx root   root         37 Fri, 2020-05-01 01:08:36 etc/systemd/system/default.target -> /lib/systemd/system/multi-user.target
drwxr-xr-x root   root          0 Tue, 2023-08-15 17:04:01 etc/systemd/system/multi-user.target.wants
lrwxrwxrwx root   root         36 Tue, 2023-08-15 17:04:01 etc/systemd/system/multi-user.target.wants/ntpd.service -> /usr/lib/systemd/system/ntpd.service
lrwxrwxrwx root   root         40 Fri, 2020-05-01 01:06:28 etc/systemd/system/multi-user.target.wants/remote-fs.target -> /usr/lib/systemd/system/remote-fs.target
lrwxrwxrwx root   root         39 Fri, 2020-05-01 01:06:32 etc/systemd/system/multi-user.target.wants/rpcbind.service -> /usr/lib/systemd/system/rpcbind.service
lrwxrwxrwx root   root         46 Fri, 2020-05-01 01:06:33 etc/systemd/system/multi-user.target.wants/rhel-configure.service -> /usr/lib/systemd/system/rhel-configure.service
lrwxrwxrwx root   root         37 Fri, 2020-05-01 01:06:39 etc/systemd/system/multi-user.target.wants/crond.service -> /usr/lib/systemd/system/crond.service
lrwxrwxrwx root   root         46 Fri, 2020-05-01 01:06:44 etc/systemd/system/multi-user.target.wants/NetworkManager.service -> /usr/lib/systemd/system/NetworkManager.service
lrwxrwxrwx root   root         37 Fri, 2020-05-01 01:06:51 etc/systemd/system/multi-user.target.wants/tuned.service -> /usr/lib/systemd/system/tuned.service
lrwxrwxrwx root   root         41 Fri, 2020-05-01 01:06:53 etc/systemd/system/multi-user.target.wants/nfs-client.target -> /usr/lib/systemd/system/nfs-client.target
lrwxrwxrwx root   root         40 Fri, 2020-05-01 01:06:54 etc/systemd/system/multi-user.target.wants/vmtoolsd.service -> /usr/lib/systemd/system/vmtoolsd.service
lrwxrwxrwx root   root         36 Fri, 2020-05-01 01:06:57 etc/systemd/system/multi-user.target.wants/sshd.service -> /usr/lib/systemd/system/sshd.service
lrwxrwxrwx root   root         39 Fri, 2020-05-01 01:07:00 etc/systemd/system/multi-user.target.wants/postfix.service -> /usr/lib/systemd/system/postfix.service
lrwxrwxrwx root   root         38 Fri, 2020-05-01 01:07:01 etc/systemd/system/multi-user.target.wants/auditd.service -> /usr/lib/systemd/system/auditd.service
lrwxrwxrwx root   root         39 Fri, 2020-05-01 01:07:01 etc/systemd/system/multi-user.target.wants/chronyd.service -> /usr/lib/systemd/system/chronyd.service
lrwxrwxrwx root   root         39 Fri, 2020-05-01 01:07:02 etc/systemd/system/multi-user.target.wants/rsyslog.service -> /usr/lib/systemd/system/rsyslog.service
lrwxrwxrwx root   root         42 Fri, 2020-05-01 01:07:02 etc/systemd/system/multi-user.target.wants/irqbalance.service -> /usr/lib/systemd/system/irqbalance.service
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:33 etc/systemd/system/local-fs.target.wants
lrwxrwxrwx root   root         45 Fri, 2020-05-01 01:06:33 etc/systemd/system/local-fs.target.wants/rhel-readonly.service -> /usr/lib/systemd/system/rhel-readonly.service
lrwxrwxrwx root   root         57 Fri, 2020-05-01 01:06:44 etc/systemd/system/dbus-org.freedesktop.nm-dispatcher.service -> /usr/lib/systemd/system/NetworkManager-dispatcher.service
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:28 etc/systemd/system/getty.target.wants
lrwxrwxrwx root   root         38 Fri, 2020-05-01 01:06:28 etc/systemd/system/getty.target.wants/getty@tty1.service -> /usr/lib/systemd/system/getty@.service
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:28 etc/systemd/system/default.target.wants
lrwxrwxrwx root   root         56 Fri, 2020-05-01 01:06:28 etc/systemd/system/default.target.wants/systemd-readahead-replay.service -> /usr/lib/systemd/system/systemd-readahead-replay.service
lrwxrwxrwx root   root         57 Fri, 2020-05-01 01:06:28 etc/systemd/system/default.target.wants/systemd-readahead-collect.service -> /usr/lib/systemd/system/systemd-readahead-collect.service
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:33 etc/systemd/system/sysinit.target.wants
lrwxrwxrwx root   root         48 Fri, 2020-05-01 01:06:33 etc/systemd/system/sysinit.target.wants/rhel-autorelabel.service -> /usr/lib/systemd/system/rhel-autorelabel.service
lrwxrwxrwx root   root         53 Fri, 2020-05-01 01:06:33 etc/systemd/system/sysinit.target.wants/rhel-autorelabel-mark.service -> /usr/lib/systemd/system/rhel-autorelabel-mark.service
lrwxrwxrwx root   root         47 Fri, 2020-05-01 01:06:33 etc/systemd/system/sysinit.target.wants/rhel-domainname.service -> /usr/lib/systemd/system/rhel-domainname.service
lrwxrwxrwx root   root         49 Fri, 2020-05-01 01:06:33 etc/systemd/system/sysinit.target.wants/rhel-import-state.service -> /usr/lib/systemd/system/rhel-import-state.service
lrwxrwxrwx root   root         48 Fri, 2020-05-01 01:06:33 etc/systemd/system/sysinit.target.wants/rhel-loadmodules.service -> /usr/lib/systemd/system/rhel-loadmodules.service
drwxr-xr-x root   root          0 Tue, 2020-04-07 17:38:09 etc/systemd/user
-rw-r--r-- root   root       1222 Tue, 2023-08-15 17:04:00 etc/passwd-
drwxr-xr-x root   root          0 Tue, 2023-08-15 17:03:06 etc/udev
-r--r--r-- root   root    8384358 Tue, 2023-08-15 17:03:06 etc/udev/hwdb.bin
-rw-r--r-- root   root         49 Tue, 2020-04-07 17:38:09 etc/udev/udev.conf
drwxr-xr-x root   root          0 Tue, 2020-04-07 17:38:09 etc/udev/rules.d
-r--r--r-- root   root         33 Tue, 2023-08-15 17:03:06 etc/machine-id
-rw-r--r-- root   root       1949 Fri, 2020-05-01 01:06:28 etc/nsswitch.conf
-rw-r--r-- root   root        216 Wed, 2020-04-01 07:04:49 etc/sestatus.conf
-rw-r--r-- root   root         16 Fri, 2020-05-01 01:08:36 etc/adjtime
-rw-r--r-- root   root        511 Wed, 2020-04-01 05:50:02 etc/inittab
-rw-r--r-- root   root         58 Wed, 2020-04-01 05:50:02 etc/networks
-rw-r--r-- root   root        966 Wed, 2020-04-01 05:50:02 etc/rwtab
-rw-r--r-- root   root        212 Wed, 2020-04-01 05:50:02 etc/statetab
-rw-r--r-- root   root        449 Wed, 2020-04-01 05:50:02 etc/sysctl.conf
drwxr-xr-x root   root          0 Tue, 2014-06-10 02:14:31 etc/cron.hourly
-rwxr-xr-x root   root        392 Fri, 2019-08-09 02:07:24 etc/cron.hourly/0anacron
-rw-r--r-- root   root        451 Tue, 2014-06-10 02:14:31 etc/crontab
lrwxrwxrwx root   root         22 Fri, 2020-05-01 01:06:42 etc/grub2.cfg -> ../boot/grub2/grub.cfg
-rw-r--r-- root   root       1317 Wed, 2018-04-11 05:44:54 etc/ethertypes
-rw-r--r-- root   root       5090 Tue, 2019-08-06 16:44:33 etc/DIR_COLORS
-rw-r--r-- root   root       5725 Tue, 2019-08-06 16:44:33 etc/DIR_COLORS.256color
-rw-r--r-- root   root        646 Tue, 2020-03-31 15:15:49 etc/krb5.conf
drwxr-xr-x root   root          0 Wed, 2020-04-01 06:06:19 etc/krb5.conf.d
drwxr-xr-x root   root          0 Wed, 2020-04-01 06:55:44 etc/exports.d
-rw-r--r-- root   root       1023 Wed, 2020-04-01 06:55:43 etc/nfs.conf
-rw-r--r-- root   root       3391 Wed, 2020-04-01 06:55:43 etc/nfsmount.conf
-rw-r--r-- root   root       1108 Thu, 2019-08-08 14:40:15 etc/chrony.conf
-rw-r--r-- root   root        458 Wed, 2020-04-01 07:22:32 etc/rsyncd.conf
-rw-r--r-- root   root       3232 Wed, 2019-11-27 21:31:22 etc/rsyslog.conf
-rw-r--r-- root   root       5171 Tue, 2018-10-30 23:26:28 etc/man_db.conf
-rw-r--r-- root   root      23316 Tue, 2023-08-15 17:04:13 etc/ld.so.cache
-rw-r--r-- root   root       1261 Tue, 2023-08-15 17:04:14 etc/passwd
-rw-r--r-- root   root         18 Tue, 2023-08-15 17:04:14 etc/subuid
-rw-r--r-- root   root          0 Wed, 2020-04-01 07:29:33 etc/subgid-
-rw-r--r-- root   root         18 Tue, 2023-08-15 17:04:14 etc/subgid
-rw-r--r-- root   root        112 Wed, 2019-11-27 18:45:31 etc/e2fsck.conf
-rw-r--r-- root   root        936 Wed, 2020-04-01 05:27:20 etc/mke2fs.conf
drwxr-xr-x root   root          0 Tue, 2023-08-15 17:03:48 etc/yum.repos.d
-rw-r--r-- root   root       1050 Mon, 2017-10-02 20:44:08 etc/yum.repos.d/epel-testing.repo
-rw-r--r-- root   root        951 Mon, 2017-10-02 20:44:08 etc/yum.repos.d/epel.repo
-rw-r--r-- root   root       1664 Wed, 2020-04-08 01:01:12 etc/yum.repos.d/CentOS-Base.repo
-rw-r--r-- root   root       1309 Wed, 2020-04-08 01:01:12 etc/yum.repos.d/CentOS-CR.repo
-rw-r--r-- root   root        649 Wed, 2020-04-08 01:01:12 etc/yum.repos.d/CentOS-Debuginfo.repo
-rw-r--r-- root   root        630 Wed, 2020-04-08 01:01:12 etc/yum.repos.d/CentOS-Media.repo
-rw-r--r-- root   root       1331 Wed, 2020-04-08 01:01:12 etc/yum.repos.d/CentOS-Sources.repo
-rw-r--r-- root   root       7577 Wed, 2020-04-08 01:01:12 etc/yum.repos.d/CentOS-Vault.repo
-rw-r--r-- root   root        314 Wed, 2020-04-08 01:01:12 etc/yum.repos.d/CentOS-fasttrack.repo
-rw-r--r-- root   root        616 Wed, 2020-04-08 01:01:12 etc/yum.repos.d/CentOS-x86_64-kernel.repo
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:05:05 etc/pm
drwxr-xr-x root   root          0 Wed, 2018-04-11 07:59:55 etc/pm/sleep.d
drwxr-xr-x root   root          0 Wed, 2018-04-11 07:59:55 etc/pm/config.d
drwxr-xr-x root   root          0 Wed, 2018-04-11 07:59:55 etc/pm/power.d
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:05:14 etc/skel
-rw-r--r-- root   root         18 Wed, 2020-04-01 05:17:30 etc/skel/.bash_logout
-rw-r--r-- root   root        193 Wed, 2020-04-01 05:17:30 etc/skel/.bash_profile
-rw-r--r-- root   root        231 Wed, 2020-04-01 05:17:30 etc/skel/.bashrc
drwxr-xr-x root   root          0 Wed, 2018-04-11 07:59:55 etc/xinetd.d
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:35 etc/ld.so.conf.d
-rw-r--r-- root   root         26 Tue, 2020-04-07 17:41:17 etc/ld.so.conf.d/bind-export-x86_64.conf
-rw-r--r-- root   root         17 Thu, 2020-04-02 20:52:07 etc/ld.so.conf.d/mariadb-x86_64.conf
-r--r--r-- root   root         63 Wed, 2020-04-01 02:40:56 etc/ld.so.conf.d/kernel-3.10.0-1127.el7.x86_64.conf
drwxr-xr-x root   root          0 Wed, 2017-08-02 18:54:38 etc/gcrypt
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:05:25 etc/groff
drwxr-xr-x root   root          0 Tue, 2014-06-10 00:17:16 etc/groff/site-font
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:05:25 etc/groff/site-tmac
-rw-r--r-- root   root         96 Tue, 2014-06-10 00:17:15 etc/groff/site-tmac/man.local
-rw-r--r-- root   root         90 Tue, 2014-06-10 00:17:15 etc/groff/site-tmac/mdoc.local
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:05:30 etc/iproute2
-rw-r--r-- root   root         85 Mon, 2019-12-02 20:45:23 etc/iproute2/bpf_pinning
-rw-r--r-- root   root         75 Mon, 2019-12-02 20:45:23 etc/iproute2/ematch_map
-rw-r--r-- root   root         31 Mon, 2019-12-02 20:45:23 etc/iproute2/group
-rw-r--r-- root   root        262 Mon, 2019-12-02 20:45:23 etc/iproute2/nl_protos
-rw-r--r-- root   root        735 Mon, 2019-12-02 20:45:23 etc/iproute2/rt_dsfield
-rw-r--r-- root   root        326 Mon, 2019-12-02 20:45:23 etc/iproute2/rt_protos
-rw-r--r-- root   root        112 Mon, 2019-12-02 20:45:23 etc/iproute2/rt_realms
-rw-r--r-- root   root         92 Mon, 2019-12-02 20:45:23 etc/iproute2/rt_scopes
-rw-r--r-- root   root         87 Mon, 2019-12-02 20:45:23 etc/iproute2/rt_tables
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:09:26 etc/pam.d
-rw-r--r-- root   root        232 Wed, 2020-04-01 06:59:56 etc/pam.d/config-util
lrwxrwxrwx root   root         19 Fri, 2020-05-01 01:08:36 etc/pam.d/fingerprint-auth -> fingerprint-auth-ac
-rw-r--r-- root   root        154 Wed, 2020-04-01 06:59:56 etc/pam.d/other
lrwxrwxrwx root   root         16 Fri, 2020-05-01 01:08:36 etc/pam.d/password-auth -> password-auth-ac
lrwxrwxrwx root   root         12 Fri, 2020-05-01 01:08:36 etc/pam.d/postlogin -> postlogin-ac
lrwxrwxrwx root   root         17 Fri, 2020-05-01 01:08:36 etc/pam.d/smartcard-auth -> smartcard-auth-ac
lrwxrwxrwx root   root         14 Fri, 2020-05-01 01:08:36 etc/pam.d/system-auth -> system-auth-ac
-rw-r--r-- root   root        192 Wed, 2020-04-01 07:51:38 etc/pam.d/chfn
-rw-r--r-- root   root        192 Wed, 2020-04-01 07:51:38 etc/pam.d/chsh
-rw-r--r-- root   root        796 Wed, 2020-04-01 07:51:38 etc/pam.d/login
-rw-r--r-- root   root        681 Wed, 2020-04-01 07:51:38 etc/pam.d/remote
-rw-r--r-- root   root        143 Wed, 2020-04-01 07:51:38 etc/pam.d/runuser
-rw-r--r-- root   root        138 Wed, 2020-04-01 07:51:38 etc/pam.d/runuser-l
-rw-r--r-- root   root        689 Fri, 2020-05-01 01:09:26 etc/pam.d/su
-rw-r--r-- root   root        137 Wed, 2020-04-01 07:51:38 etc/pam.d/su-l
-rw-r--r-- root   root        129 Tue, 2020-04-07 17:38:09 etc/pam.d/systemd-user
-rw-r--r-- root   root        155 Wed, 2020-04-01 07:07:09 etc/pam.d/polkit-1
-rw-r--r-- root   root        287 Fri, 2019-08-09 02:07:24 etc/pam.d/crond
-rw-r--r-- root   root         84 Wed, 2018-10-31 01:38:31 etc/pam.d/vlock
-rw-r--r-- root   root        278 Thu, 2020-04-02 18:58:22 etc/pam.d/vmtoolsd
-rw-r--r-- root   root        904 Fri, 2019-08-09 04:40:39 etc/pam.d/sshd
-rw-r--r-- root   root         76 Wed, 2020-04-01 07:08:18 etc/pam.d/smtp.postfix
lrwxrwxrwx root   root         25 Fri, 2020-05-01 01:07:00 etc/pam.d/smtp -> /etc/alternatives/mta-pam
-rw-r--r-- root   root        188 Wed, 2020-04-01 06:57:16 etc/pam.d/passwd
-rw-r--r-- root   root        200 Wed, 2020-04-01 07:37:09 etc/pam.d/sudo
-rw-r--r-- root   root        178 Wed, 2020-04-01 07:37:09 etc/pam.d/sudo-i
-rw-r--r-- root   root       1028 Fri, 2020-05-01 01:08:36 etc/pam.d/system-auth-ac
-rw-r--r-- root   root        330 Fri, 2020-05-01 01:08:36 etc/pam.d/postlogin-ac
-rw-r--r-- root   root       1030 Fri, 2020-05-01 01:08:36 etc/pam.d/password-auth-ac
-rw-r--r-- root   root        702 Fri, 2020-05-01 01:08:36 etc/pam.d/fingerprint-auth-ac
-rw-r--r-- root   root        752 Fri, 2020-05-01 01:08:36 etc/pam.d/smartcard-auth-ac
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:47 etc/rwtab.d
-rw-r--r-- root   root         24 Wed, 2020-04-01 06:26:08 etc/rwtab.d/logrotate
-rw-r--r-- root   root         23 Wed, 2020-04-01 05:45:26 etc/rwtab.d/gssproxy
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:13 etc/my.cnf.d
-rw-r--r-- root   root        232 Thu, 2019-07-25 17:41:33 etc/my.cnf.d/mysql-clients.cnf
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:42 etc/selinux
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:55 etc/selinux/targeted
-rw-r--r-- root   root        607 Wed, 2020-04-01 07:29:41 etc/selinux/targeted/setrans.conf
-rw-r--r-- root   root        129 Wed, 2020-04-01 07:29:41 etc/selinux/targeted/.policy.sha512
-rw-r--r-- root   root       2623 Wed, 2020-04-01 07:29:41 etc/selinux/targeted/booleans.subs_dist
drwxr-xr-x root   root          0 Wed, 2020-04-01 07:29:41 etc/selinux/targeted/logins
-rw-r--r-- root   root        106 Wed, 2020-04-01 07:29:53 etc/selinux/targeted/seusers
drwx------ root   root          0 Fri, 2020-05-01 01:06:55 etc/selinux/targeted/active
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:54 etc/selinux/targeted/modules
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:54 etc/selinux/targeted/modules/active
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:54 etc/selinux/targeted/modules/active/modules
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:55 etc/selinux/targeted/contexts
-rw-r--r-- root   root        262 Wed, 2020-04-01 07:29:41 etc/selinux/targeted/contexts/customizable_types
-rw-r--r-- root   root        195 Wed, 2020-04-01 07:29:06 etc/selinux/targeted/contexts/dbus_contexts
-rw-r--r-- root   root        254 Wed, 2020-04-01 07:29:06 etc/selinux/targeted/contexts/default_contexts
-rw-r--r-- root   root        148 Wed, 2020-04-01 07:29:06 etc/selinux/targeted/contexts/default_type
-rw-r--r-- root   root         29 Wed, 2020-04-01 07:29:06 etc/selinux/targeted/contexts/failsafe_context
-rw-r--r-- root   root         30 Wed, 2020-04-01 07:29:06 etc/selinux/targeted/contexts/initrc_context
-rw-r--r-- root   root        333 Wed, 2020-04-01 07:29:06 etc/selinux/targeted/contexts/lxc_contexts
-rw-r--r-- root   root         33 Wed, 2020-04-01 07:29:06 etc/selinux/targeted/contexts/removable_context
-rw-r--r-- root   root         74 Wed, 2020-04-01 07:29:41 etc/selinux/targeted/contexts/securetty_types
-rw-r--r-- root   root       1170 Wed, 2020-04-01 07:29:06 etc/selinux/targeted/contexts/sepgsql_contexts
-rw-r--r-- root   root         53 Wed, 2020-04-01 07:29:06 etc/selinux/targeted/contexts/snapperd_contexts
-rw-r--r-- root   root         57 Wed, 2020-04-01 07:29:06 etc/selinux/targeted/contexts/systemd_contexts
-rw-r--r-- root   root         35 Wed, 2020-04-01 07:29:06 etc/selinux/targeted/contexts/userhelper_context
-rw-r--r-- root   root         62 Wed, 2020-04-01 07:29:06 etc/selinux/targeted/contexts/virtual_domain_context
-rw-r--r-- root   root         71 Wed, 2020-04-01 07:29:06 etc/selinux/targeted/contexts/virtual_image_context
-rw-r--r-- root   root       2920 Wed, 2020-04-01 07:29:06 etc/selinux/targeted/contexts/x_contexts
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:55 etc/selinux/targeted/contexts/files
-rw-r--r-- root   root     384579 Wed, 2020-04-01 07:29:53 etc/selinux/targeted/contexts/files/file_contexts
-rw-r--r-- root   root    1416154 Wed, 2020-04-01 07:35:42 etc/selinux/targeted/contexts/files/file_contexts.bin
-rw-r--r-- root   root      13406 Wed, 2020-04-01 07:29:53 etc/selinux/targeted/contexts/files/file_contexts.homedirs
-rw-r--r-- root   root      45577 Wed, 2020-04-01 07:29:53 etc/selinux/targeted/contexts/files/file_contexts.homedirs.bin
-rw-r--r-- root   root          0 Wed, 2020-04-01 07:29:41 etc/selinux/targeted/contexts/files/file_contexts.subs
-rw-r--r-- root   root        514 Wed, 2020-04-01 07:29:41 etc/selinux/targeted/contexts/files/file_contexts.subs_dist
-rw-r--r-- root   root        139 Wed, 2020-04-01 07:29:06 etc/selinux/targeted/contexts/files/media
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:55 etc/selinux/targeted/contexts/users
-rw-r--r-- root   root        253 Wed, 2020-04-01 07:29:06 etc/selinux/targeted/contexts/users/guest_u
-rw-r--r-- root   root        389 Wed, 2020-04-01 07:29:06 etc/selinux/targeted/contexts/users/root
-rw-r--r-- root   root        514 Wed, 2020-04-01 07:29:06 etc/selinux/targeted/contexts/users/staff_u
-rw-r--r-- root   root        496 Wed, 2020-04-01 07:29:06 etc/selinux/targeted/contexts/users/sysadm_u
-rw-r--r-- root   root        578 Wed, 2020-04-01 07:29:06 etc/selinux/targeted/contexts/users/unconfined_u
-rw-r--r-- root   root        353 Wed, 2020-04-01 07:29:06 etc/selinux/targeted/contexts/users/user_u
-rw-r--r-- root   root        307 Wed, 2020-04-01 07:29:06 etc/selinux/targeted/contexts/users/xguest_u
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:55 etc/selinux/targeted/policy
-rw-r--r-- root   root    3905267 Wed, 2020-04-01 07:29:53 etc/selinux/targeted/policy/policy.31
-rw-r--r-- root   root       2321 Wed, 2018-10-31 02:44:32 etc/selinux/semanage.conf
-rw-r--r-- root   root        542 Fri, 2020-05-01 01:08:36 etc/selinux/config
drwx------ root   root          0 Fri, 2020-05-01 01:06:42 etc/selinux/final
drwxr-xr-x root   root          0 Wed, 2018-10-31 02:44:32 etc/selinux/tmp
drwxr-xr-x root   root          0 Fri, 2018-07-13 16:05:22 etc/gnupg
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:09:26 etc/dracut.conf.d
-rw-r--r-- root   root         28 Fri, 2020-05-01 01:09:26 etc/dracut.conf.d/vmware-fusion-drivers.conf
-rw-r--r-- root   root         67 Fri, 2020-05-01 01:09:26 etc/dracut.conf.d/hyperv-drivers.conf
-rw-r--r-- root   root         25 Fri, 2020-05-01 01:09:26 etc/dracut.conf.d/nofloppy.conf
drwxr-xr-x root   root          0 Tue, 2020-04-07 17:38:09 etc/binfmt.d
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:29 etc/polkit-1
drwx------ polkitd root          0 Fri, 2020-05-01 01:06:29 etc/polkit-1/rules.d
drwxr-x--- root   polkitd        0 Fri, 2020-05-01 01:06:29 etc/polkit-1/localauthority
drwxr-xr-x root   root          0 Tue, 2014-06-10 02:08:33 etc/polkit-1/localauthority.conf.d
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:33 etc/ppp
-rwxr-xr-x root   root        386 Mon, 2019-08-19 14:34:03 etc/ppp/ip-down
-rwxr-xr-x root   root       3214 Mon, 2019-08-19 14:34:03 etc/ppp/ip-down.ipv6to4
-rwxr-xr-x root   root        430 Mon, 2019-08-19 14:34:03 etc/ppp/ip-up
-rwxr-xr-x root   root       6426 Mon, 2019-08-19 14:34:03 etc/ppp/ip-up.ipv6to4
-rwxr-xr-x root   root       1687 Mon, 2019-08-19 14:34:03 etc/ppp/ipv6-down
-rwxr-xr-x root   root       3182 Mon, 2019-08-19 14:34:03 etc/ppp/ipv6-up
drwxr-xr-x root   root          0 Wed, 2020-04-01 05:50:02 etc/ppp/peers
drwxr-xr-x root   root          0 Tue, 2014-06-10 02:14:31 etc/cron.monthly
drwxr-xr-x root   root          0 Tue, 2023-08-15 17:03:32 etc/ssh
-rw-r--r-- root   root        382 Tue, 2023-08-15 17:03:08 etc/ssh/ssh_host_rsa_key.pub
-rw-r--r-- root   root        162 Tue, 2023-08-15 17:03:08 etc/ssh/ssh_host_ecdsa_key.pub
-rw-r--r-- root   root         82 Tue, 2023-08-15 17:03:08 etc/ssh/ssh_host_ed25519_key.pub
-rw-r--r-- root   root     581843 Fri, 2019-08-09 04:40:39 etc/ssh/moduli
-rw-r--r-- root   root       2276 Fri, 2019-08-09 04:40:39 etc/ssh/ssh_config
drwxr-x--- root   root          0 Fri, 2020-05-01 01:06:56 etc/dhcp
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:53 etc/cifs-utils
lrwxrwxrwx root   root         35 Fri, 2020-05-01 01:06:53 etc/cifs-utils/idmap-plugin -> /etc/alternatives/cifs-idmap-plugin
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:59 etc/postfix
-rw-r--r-- root   root      20876 Thu, 2010-11-04 17:57:12 etc/postfix/access
-rw-r--r-- root   root      11883 Wed, 2020-04-01 07:07:50 etc/postfix/canonical
-rw-r--r-- root   root      10106 Wed, 2020-04-01 07:07:50 etc/postfix/generic
-rw-r--r-- root   root      21545 Mon, 2012-09-03 20:25:07 etc/postfix/header_checks
-rw-r--r-- root   root      27176 Wed, 2020-04-01 07:08:17 etc/postfix/main.cf
-rw-r--r-- root   root       6105 Wed, 2020-04-01 07:07:50 etc/postfix/master.cf
-rw-r--r-- root   root       6816 Tue, 2007-03-27 16:40:58 etc/postfix/relocated
-rw-r--r-- root   root      12549 Sun, 2009-11-22 04:22:24 etc/postfix/transport
-rw-r--r-- root   root      12696 Wed, 2020-04-01 07:07:50 etc/postfix/virtual
drwxr-x--- root   root          0 Fri, 2020-05-01 01:07:00 etc/audisp
drwx------ root   root          0 Fri, 2020-05-01 01:08:05 etc/grub.d
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:20 etc/yum
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:08:35 etc/yum/pluginconf.d
-rw-r--r-- root   root        279 Wed, 2020-04-01 08:03:27 etc/yum/pluginconf.d/fastestmirror.conf
-rw-r--r-- root   root        372 Fri, 2020-05-01 01:04:55 etc/yum/pluginconf.d/langpacks.conf
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:26 etc/yum/protected.d
-rw-r--r-- root   root          8 Tue, 2020-04-07 17:38:10 etc/yum/protected.d/systemd.conf
-rw-r--r-- root   root        444 Thu, 2020-04-02 18:56:34 etc/yum/version-groups.conf
drwxr-xr-x root   root          0 Thu, 2020-04-02 18:56:34 etc/yum/vars
-rw-r--r-- root   root          7 Wed, 2020-04-08 01:01:12 etc/yum/vars/contentdir
-rw-r--r-- root   root          4 Fri, 2020-05-01 01:09:26 etc/yum/vars/infra
drwxr-xr-x root   root          0 Thu, 2020-04-02 18:56:34 etc/yum/fssnap.d
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:07:06 etc/profile.d
-rw-r--r-- root   root         80 Wed, 2020-04-01 07:29:33 etc/profile.d/csh.local
-rw-r--r-- root   root         81 Wed, 2020-04-01 07:29:33 etc/profile.d/sh.local
-rw-r--r-- root   root        196 Fri, 2017-03-24 19:39:09 etc/profile.d/colorgrep.csh
-rw-r--r-- root   root        201 Fri, 2017-03-24 19:39:09 etc/profile.d/colorgrep.sh
-rw-r--r-- root   root        164 Tue, 2014-01-28 00:47:50 etc/profile.d/which2.csh
-rw-r--r-- root   root        169 Tue, 2014-01-28 00:47:50 etc/profile.d/which2.sh
-rw-r--r-- root   root        123 Fri, 2015-07-31 02:47:02 etc/profile.d/less.csh
-rw-r--r-- root   root        121 Fri, 2015-07-31 02:47:02 etc/profile.d/less.sh
-rw-r--r-- root   root       1741 Tue, 2019-08-06 16:44:33 etc/profile.d/colorls.csh
-rw-r--r-- root   root       1606 Tue, 2019-08-06 16:44:33 etc/profile.d/colorls.sh
-rw-r--r-- root   root        771 Wed, 2020-04-01 05:50:02 etc/profile.d/256term.csh
-rw-r--r-- root   root        841 Wed, 2020-04-01 05:50:02 etc/profile.d/256term.sh
-rw-r--r-- root   root       1706 Wed, 2020-04-01 05:50:02 etc/profile.d/lang.csh
-rw-r--r-- root   root       2703 Wed, 2020-04-01 05:50:02 etc/profile.d/lang.sh
-rw-r--r-- root   root        660 Wed, 2020-04-01 05:15:39 etc/profile.d/bash_completion.sh
drwxr-xr-x root   root          0 Tue, 2023-08-15 17:04:00 etc/sysconfig
drwxr-xr-x root   root          0 Wed, 2020-04-01 05:50:02 etc/sysconfig/console
drwxr-xr-x root   root          0 Wed, 2020-04-01 05:50:02 etc/sysconfig/modules
-rw-r--r-- root   root        180 Fri, 2020-05-01 01:08:29 etc/sysconfig/kernel
-rw-r--r-- root   root         21 Fri, 2020-05-01 01:10:37 etc/sysconfig/cloud-info
-rw-r--r-- root   root         22 Tue, 2023-08-15 17:03:25 etc/sysconfig/network
-rw-r--r-- root   root        111 Wed, 2019-11-27 19:47:41 etc/sysconfig/ntpdate
-rw-r--r-- root   root         45 Wed, 2019-11-27 19:47:41 etc/sysconfig/ntpd
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:05:30 etc/sysconfig/cbq
-rw-r--r-- root   root         11 Mon, 2019-12-02 20:45:23 etc/sysconfig/cbq/avpkt
-rw-r--r-- root   root         79 Mon, 2019-12-02 20:45:23 etc/sysconfig/cbq/cbq-0000.example
-rw-r--r-- root   root         73 Wed, 2020-04-01 07:19:32 etc/sysconfig/rpcbind
-rw-r--r-- root   root         15 Fri, 2017-08-04 11:01:03 etc/sysconfig/rdisc
-rw-r--r-- root   root        798 Wed, 2020-04-01 05:50:02 etc/sysconfig/init
-rw-r--r-- root   root        634 Wed, 2020-04-01 05:50:02 etc/sysconfig/netconsole
drwxr-xr-x root   root          0 Tue, 2023-08-15 17:03:29 etc/sysconfig/network-scripts
-rw-r--r-- root   root        254 Tue, 2023-08-15 17:03:25 etc/sysconfig/network-scripts/ifcfg-lo
-rw-r--r-- root   root         86 Tue, 2023-08-15 17:03:25 etc/sysconfig/network-scripts/ifcfg-eth0
lrwxrwxrwx root   root         24 Fri, 2020-05-01 01:06:33 etc/sysconfig/network-scripts/ifdown -> ../../../usr/sbin/ifdown
-rwxr-xr-x root   root        654 Mon, 2019-08-19 14:34:03 etc/sysconfig/network-scripts/ifdown-bnep
-rwxr-xr-x root   root       6532 Mon, 2019-08-19 14:34:03 etc/sysconfig/network-scripts/ifdown-eth
-rwxr-xr-x root   root        781 Mon, 2019-08-19 14:34:03 etc/sysconfig/network-scripts/ifdown-ippp
-rwxr-xr-x root   root       4540 Mon, 2019-08-19 14:34:03 etc/sysconfig/network-scripts/ifdown-ipv6
lrwxrwxrwx root   root         11 Fri, 2020-05-01 01:06:33 etc/sysconfig/network-scripts/ifdown-isdn -> ifdown-ippp
-rwxr-xr-x root   root       2130 Mon, 2019-08-19 14:34:03 etc/sysconfig/network-scripts/ifdown-post
-rwxr-xr-x root   root       1068 Mon, 2019-08-19 14:34:03 etc/sysconfig/network-scripts/ifdown-ppp
-rwxr-xr-x root   root        870 Mon, 2019-08-19 14:34:03 etc/sysconfig/network-scripts/ifdown-routes
-rwxr-xr-x root   root       1456 Mon, 2019-08-19 14:34:03 etc/sysconfig/network-scripts/ifdown-sit
-rwxr-xr-x root   root       1462 Mon, 2019-08-19 14:34:03 etc/sysconfig/network-scripts/ifdown-tunnel
lrwxrwxrwx root   root         22 Fri, 2020-05-01 01:06:33 etc/sysconfig/network-scripts/ifup -> ../../../usr/sbin/ifup
-rwxr-xr-x root   root      12415 Mon, 2019-08-19 14:34:03 etc/sysconfig/network-scripts/ifup-aliases
-rwxr-xr-x root   root        910 Mon, 2019-08-19 14:34:03 etc/sysconfig/network-scripts/ifup-bnep
-rwxr-xr-x root   root      13574 Mon, 2019-08-19 14:34:03 etc/sysconfig/network-scripts/ifup-eth
-rwxr-xr-x root   root      12075 Mon, 2019-08-19 14:34:03 etc/sysconfig/network-scripts/ifup-ippp
-rwxr-xr-x root   root      11893 Mon, 2019-08-19 14:34:03 etc/sysconfig/network-scripts/ifup-ipv6
lrwxrwxrwx root   root          9 Fri, 2020-05-01 01:06:33 etc/sysconfig/network-scripts/ifup-isdn -> ifup-ippp
-rwxr-xr-x root   root        650 Mon, 2019-08-19 14:34:03 etc/sysconfig/network-scripts/ifup-plip
-rwxr-xr-x root   root       1064 Mon, 2019-08-19 14:34:03 etc/sysconfig/network-scripts/ifup-plusb
-rwxr-xr-x root   root       4997 Mon, 2019-08-19 14:34:03 etc/sysconfig/network-scripts/ifup-post
-rwxr-xr-x root   root       4154 Mon, 2019-08-19 14:34:03 etc/sysconfig/network-scripts/ifup-ppp
-rwxr-xr-x root   root       2001 Mon, 2019-08-19 14:34:03 etc/sysconfig/network-scripts/ifup-routes
-rwxr-xr-x root   root       3303 Mon, 2019-08-19 14:34:03 etc/sysconfig/network-scripts/ifup-sit
-rwxr-xr-x root   root       2780 Mon, 2019-08-19 14:34:03 etc/sysconfig/network-scripts/ifup-tunnel
-rwxr-xr-x root   root       1836 Mon, 2019-08-19 14:34:03 etc/sysconfig/network-scripts/ifup-wireless
-rwxr-xr-x root   root       5419 Mon, 2019-08-19 14:34:03 etc/sysconfig/network-scripts/init.ipv6-global
-rw-r--r-- root   root      20678 Mon, 2019-08-19 14:34:03 etc/sysconfig/network-scripts/network-functions
-rw-r--r-- root   root      31027 Mon, 2019-08-19 14:34:03 etc/sysconfig/network-scripts/network-functions-ipv6
-rwxr-xr-x root   root       1621 Sun, 2018-12-09 11:57:18 etc/sysconfig/network-scripts/ifdown-Team
-rwxr-xr-x root   root       1556 Sun, 2018-12-09 11:57:18 etc/sysconfig/network-scripts/ifdown-TeamPort
-rwxr-xr-x root   root       1755 Sun, 2018-12-09 11:57:18 etc/sysconfig/network-scripts/ifup-Team
-rwxr-xr-x root   root       1876 Sun, 2018-12-09 11:57:18 etc/sysconfig/network-scripts/ifup-TeamPort
-rw-r--r-- root   root        905 Wed, 2020-04-01 05:50:02 etc/sysconfig/readonly-root
-rw-r--r-- root   root          0 Tue, 2014-06-10 02:14:31 etc/sysconfig/run-parts
-rw-r--r-- root   root        428 Thu, 2020-04-02 16:24:09 etc/sysconfig/samba
lrwxrwxrwx root   root         15 Fri, 2020-05-01 01:06:41 etc/sysconfig/grub -> ../default/grub
-rw-r--r-- root   root        395 Tue, 2019-08-06 16:44:45 etc/sysconfig/rpc-rquotad
lrwxrwxrwx root   root         17 Fri, 2020-05-01 01:06:42 etc/sysconfig/selinux -> ../selinux/config
-rw-r--r-- root   root        610 Wed, 2018-10-31 02:03:42 etc/sysconfig/wpa_supplicant
-rw-r--r-- root   root         73 Tue, 2020-04-07 17:37:15 etc/sysconfig/firewalld
-rw-r--r-- root   root       1679 Wed, 2020-04-01 06:55:43 etc/sysconfig/nfs
-rw-r--r-- root   root        480 Fri, 2020-05-01 01:08:36 etc/sysconfig/authconfig
-rw-r--r-- root   root        911 Tue, 2019-08-06 16:44:40 etc/sysconfig/qemu-ga
-rw-r--r-- root   root         46 Thu, 2019-08-08 14:40:18 etc/sysconfig/chronyd
-rw-r--r-- root   root         12 Wed, 2020-04-01 07:22:32 etc/sysconfig/rsyncd
-rw-r--r-- root   root        196 Wed, 2019-11-27 21:31:22 etc/sysconfig/rsyslog
-rw-r--r-- root   root        903 Tue, 2019-08-06 16:44:37 etc/sysconfig/irqbalance
-rw-r--r-- root   root        200 Tue, 2018-10-30 17:55:19 etc/sysconfig/man-db
-rw-r--r-- root   root        150 Wed, 2020-04-01 02:46:48 etc/sysconfig/cpupower
-rw-r--r-- root   root        102 Fri, 2020-05-01 01:09:51 etc/sysconfig/anaconda
drwxr-xr-x root   root          0 Thu, 2017-09-07 01:08:12 etc/terminfo
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:08:34 etc/default
-rw-r--r-- root   root       1756 Wed, 2020-04-01 01:09:48 etc/default/nss
-rw-r--r-- root   root        119 Tue, 2019-08-06 16:44:46 etc/default/useradd
-rw-r--r-- root   root        317 Fri, 2020-05-01 01:08:34 etc/default/grub
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:08:05 etc/alternatives
lrwxrwxrwx root   root         34 Fri, 2020-05-01 01:06:09 etc/alternatives/libnssckbi.so.x86_64 -> /usr/lib64/pkcs11/p11-kit-trust.so
lrwxrwxrwx root   root         15 Fri, 2020-05-01 01:06:10 etc/alternatives/ld -> /usr/bin/ld.bfd
lrwxrwxrwx root   root         32 Fri, 2020-05-01 01:06:53 etc/alternatives/cifs-idmap-plugin -> /usr/lib64/cifs-utils/idmapwb.so
lrwxrwxrwx root   root         26 Fri, 2020-05-01 01:07:00 etc/alternatives/mta -> /usr/sbin/sendmail.postfix
lrwxrwxrwx root   root         22 Fri, 2020-05-01 01:07:00 etc/alternatives/mta-mailq -> /usr/bin/mailq.postfix
lrwxrwxrwx root   root         27 Fri, 2020-05-01 01:07:00 etc/alternatives/mta-newaliases -> /usr/bin/newaliases.postfix
lrwxrwxrwx root   root         23 Fri, 2020-05-01 01:07:00 etc/alternatives/mta-pam -> /etc/pam.d/smtp.postfix
lrwxrwxrwx root   root         22 Fri, 2020-05-01 01:07:00 etc/alternatives/mta-rmail -> /usr/bin/rmail.postfix
lrwxrwxrwx root   root         25 Fri, 2020-05-01 01:07:00 etc/alternatives/mta-sendmail -> /usr/lib/sendmail.postfix
lrwxrwxrwx root   root         38 Fri, 2020-05-01 01:07:00 etc/alternatives/mta-mailqman -> /usr/share/man/man1/mailq.postfix.1.gz
lrwxrwxrwx root   root         43 Fri, 2020-05-01 01:07:00 etc/alternatives/mta-newaliasesman -> /usr/share/man/man1/newaliases.postfix.1.gz
lrwxrwxrwx root   root         41 Fri, 2020-05-01 01:07:00 etc/alternatives/mta-sendmailman -> /usr/share/man/man1/sendmail.postfix.1.gz
lrwxrwxrwx root   root         40 Fri, 2020-05-01 01:07:00 etc/alternatives/mta-aliasesman -> /usr/share/man/man5/aliases.postfix.5.gz
lrwxrwxrwx root   root         45 Fri, 2020-05-01 01:08:05 etc/alternatives/libwbclient.so.0.15-64 -> /usr/lib64/samba/wbclient/libwbclient.so.0.15
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:05:29 etc/ssl
lrwxrwxrwx root   root         16 Fri, 2020-05-01 01:05:29 etc/ssl/certs -> ../pki/tls/certs
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:05:52 etc/gss
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:47 etc/gss/mech.d
-rw-r--r-- root   root        189 Wed, 2020-04-01 05:45:26 etc/gss/mech.d/gssproxy.conf
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:07:02 etc/logrotate.d
-rw-r--r-- root   root        103 Thu, 2020-04-02 18:56:34 etc/logrotate.d/yum
-rw-r--r-- root   root        115 Thu, 2020-04-02 16:24:09 etc/logrotate.d/samba
-rw-r--r-- root   root        100 Wed, 2018-10-31 02:03:42 etc/logrotate.d/wpa_supplicant
-rw-r--r-- root   root        160 Wed, 2018-09-19 17:38:15 etc/logrotate.d/chrony
-rw-r--r-- root   root        224 Wed, 2019-11-27 21:31:22 etc/logrotate.d/syslog
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:28 etc/dbus-1
drwxr-xr-x root   root          0 Thu, 2019-03-14 13:18:01 etc/dbus-1/session.d
-rw-r--r-- root   root        838 Thu, 2019-03-14 13:17:47 etc/dbus-1/session.conf
-rw-r--r-- root   root        833 Thu, 2019-03-14 13:17:47 etc/dbus-1/system.conf
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:51 etc/dbus-1/system.d
-rw-r--r-- root   root        947 Tue, 2020-04-07 17:38:09 etc/dbus-1/system.d/org.freedesktop.hostname1.conf
-rw-r--r-- root   root       2527 Tue, 2020-04-07 17:38:09 etc/dbus-1/system.d/org.freedesktop.import1.conf
-rw-r--r-- root   root        937 Tue, 2020-04-07 17:38:09 etc/dbus-1/system.d/org.freedesktop.locale1.conf
-rw-r--r-- root   root       8534 Tue, 2020-04-07 17:38:09 etc/dbus-1/system.d/org.freedesktop.login1.conf
-rw-r--r-- root   root       3726 Tue, 2020-04-07 17:38:09 etc/dbus-1/system.d/org.freedesktop.machine1.conf
-rw-r--r-- root   root      10118 Tue, 2020-04-07 17:38:09 etc/dbus-1/system.d/org.freedesktop.systemd1.conf
-rw-r--r-- root   root        947 Tue, 2020-04-07 17:38:09 etc/dbus-1/system.d/org.freedesktop.timedate1.conf
-rw-r--r-- root   root        638 Wed, 2020-04-01 07:07:20 etc/dbus-1/system.d/org.freedesktop.PolicyKit1.conf
-rw-r--r-- root   root       1096 Wed, 2018-10-31 02:03:42 etc/dbus-1/system.d/wpa_supplicant.conf
-rw-r--r-- root   root        465 Wed, 2020-04-01 06:56:04 etc/dbus-1/system.d/nm-dispatcher.conf
-rw-r--r-- root   root        387 Wed, 2020-04-01 06:56:04 etc/dbus-1/system.d/nm-ifcfg-rh.conf
-rw-r--r-- root   root       9740 Wed, 2020-04-01 06:56:04 etc/dbus-1/system.d/org.freedesktop.NetworkManager.conf
-rw-r--r-- root   root        409 Sun, 2018-12-09 11:57:18 etc/dbus-1/system.d/teamd.conf
-rw-r--r-- root   root        587 Fri, 2019-03-22 01:10:46 etc/dbus-1/system.d/com.redhat.tuned.conf
-rw-r--r-- root   root       1084 Tue, 2020-04-07 17:37:15 etc/dbus-1/system.d/FirewallD.conf
drwxr-xr-x root   root          0 Tue, 2020-04-07 17:38:09 etc/modules-load.d
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:39 etc/cron.d
-rw-r--r-- root   root        128 Fri, 2019-08-09 02:07:24 etc/cron.d/0hourly
drwxr-xr-x root   root          0 Tue, 2014-06-10 02:14:31 etc/cron.weekly
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:43 etc/wpa_supplicant
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:51 etc/tuned
-rw-r--r-- root   root       1111 Fri, 2019-03-22 01:10:46 etc/tuned/bootcmdline
-rw-r--r-- root   root          5 Tue, 2023-08-15 17:03:13 etc/tuned/profile_mode
-rw-r--r-- root   root       1279 Thu, 2020-04-02 16:19:11 etc/tuned/tuned-main.conf
-rw-r--r-- root   root         14 Tue, 2023-08-15 17:03:13 etc/tuned/active_profile
drwxr-xr-x root   root          0 Thu, 2020-04-02 16:19:11 etc/tuned/recommend.d
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:07:01 etc/qemu-ga
-rwxr-xr-x root   root       1176 Tue, 2018-04-24 19:30:47 etc/qemu-ga/fsfreeze-hook
drwxr-xr-x root   root          0 Thu, 2019-08-08 14:49:53 etc/qemu-ga/fsfreeze-hook.d
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:07:02 etc/pki
drwxr-xr-x root   root          0 Tue, 2023-08-15 17:03:48 etc/pki/rpm-gpg
-rw-r--r-- root   root       1690 Wed, 2020-04-08 01:01:12 etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
-rw-r--r-- root   root       1004 Wed, 2020-04-08 01:01:12 etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-Debug-7
-rw-r--r-- root   root       1690 Wed, 2020-04-08 01:01:12 etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-Testing-7
-rw-r--r-- root   root       1662 Mon, 2017-10-02 20:44:08 etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-7
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:05:51 etc/pki/tls
lrwxrwxrwx root   root         49 Fri, 2020-05-01 01:05:29 etc/pki/tls/cert.pem -> /etc/pki/ca-trust/extracted/pem/tls-ca-bundle.pem
-rw-r--r-- root   root      10923 Tue, 2019-08-06 16:44:39 etc/pki/tls/openssl.cnf
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:06 etc/pki/tls/certs
lrwxrwxrwx root   root         49 Fri, 2020-05-01 01:05:29 etc/pki/tls/certs/ca-bundle.crt -> /etc/pki/ca-trust/extracted/pem/tls-ca-bundle.pem
lrwxrwxrwx root   root         55 Fri, 2020-05-01 01:05:29 etc/pki/tls/certs/ca-bundle.trust.crt -> /etc/pki/ca-trust/extracted/openssl/ca-bundle.trust.crt
-rw-r--r-- root   root       2516 Fri, 2019-08-09 04:38:13 etc/pki/tls/certs/Makefile
-rwxr-xr-x root   root        610 Fri, 2019-08-09 04:38:13 etc/pki/tls/certs/make-dummy-cert
-rwxr-xr-x root   root        829 Fri, 2019-08-09 04:38:13 etc/pki/tls/certs/renew-dummy-cert
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:06 etc/pki/tls/misc
-rwxr-xr-x root   root       5178 Fri, 2019-08-09 04:37:41 etc/pki/tls/misc/CA
-rwxr-xr-x root   root        119 Fri, 2019-08-09 04:37:41 etc/pki/tls/misc/c_hash
-rwxr-xr-x root   root        152 Fri, 2019-08-09 04:37:41 etc/pki/tls/misc/c_info
-rwxr-xr-x root   root        112 Fri, 2019-08-09 04:37:41 etc/pki/tls/misc/c_issuer
-rwxr-xr-x root   root        110 Fri, 2019-08-09 04:37:41 etc/pki/tls/misc/c_name
drwxr-xr-x root   root          0 Fri, 2019-08-09 04:37:40 etc/pki/tls/private
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:07 etc/pki/nss-legacy
-rw-r--r-- root   root        257 Wed, 2019-11-27 19:47:26 etc/pki/nss-legacy/nss-rhel7.config
drwx------ root   root          0 Wed, 2020-04-01 07:27:05 etc/pki/rsyslog
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:05:29 etc/pki/java
lrwxrwxrwx root   root         40 Fri, 2020-05-01 01:05:29 etc/pki/java/cacerts -> /etc/pki/ca-trust/extracted/java/cacerts
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:05:29 etc/pki/ca-trust
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:05:29 etc/pki/ca-trust/extracted
-rw-r--r-- root   root        560 Wed, 2019-11-27 18:33:55 etc/pki/ca-trust/extracted/README
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:05:30 etc/pki/ca-trust/extracted/java
-rw-r--r-- root   root        726 Wed, 2019-11-27 18:33:55 etc/pki/ca-trust/extracted/java/README
-r--r--r-- root   root     161905 Fri, 2020-05-01 01:05:30 etc/pki/ca-trust/extracted/java/cacerts
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:05:29 etc/pki/ca-trust/extracted/openssl
-rw-r--r-- root   root        787 Wed, 2019-11-27 18:33:55 etc/pki/ca-trust/extracted/openssl/README
-r--r--r-- root   root     261737 Fri, 2020-05-01 01:05:29 etc/pki/ca-trust/extracted/openssl/ca-bundle.trust.crt
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:05:30 etc/pki/ca-trust/extracted/pem
-rw-r--r-- root   root        898 Wed, 2019-11-27 18:33:55 etc/pki/ca-trust/extracted/pem/README
-r--r--r-- root   root     222148 Fri, 2020-05-01 01:05:29 etc/pki/ca-trust/extracted/pem/tls-ca-bundle.pem
-r--r--r-- root   root     173023 Fri, 2020-05-01 01:05:29 etc/pki/ca-trust/extracted/pem/email-ca-bundle.pem
-r--r--r-- root   root          0 Fri, 2020-05-01 01:05:30 etc/pki/ca-trust/extracted/pem/objsign-ca-bundle.pem
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:05:29 etc/pki/ca-trust/source
-rw-r--r-- root   root        932 Wed, 2019-11-27 18:33:55 etc/pki/ca-trust/source/README
lrwxrwxrwx root   root         59 Fri, 2020-05-01 01:05:29 etc/pki/ca-trust/source/ca-bundle.legacy.crt -> /usr/share/pki/ca-trust-legacy/ca-bundle.legacy.default.crt
drwxr-xr-x root   root          0 Mon, 2019-12-02 20:44:55 etc/pki/ca-trust/source/anchors
drwxr-xr-x root   root          0 Mon, 2019-12-02 20:44:55 etc/pki/ca-trust/source/blacklist
-rw-r--r-- root   root        166 Wed, 2019-11-27 18:33:55 etc/pki/ca-trust/README
-rw-r--r-- root   root        980 Wed, 2019-11-27 18:33:55 etc/pki/ca-trust/ca-legacy.conf
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:06 etc/pki/CA
drwxr-xr-x root   root          0 Fri, 2019-08-09 04:38:20 etc/pki/CA/certs
drwxr-xr-x root   root          0 Fri, 2019-08-09 04:38:20 etc/pki/CA/crl
drwxr-xr-x root   root          0 Fri, 2019-08-09 04:38:20 etc/pki/CA/newcerts
drwx------ root   root          0 Fri, 2019-08-09 04:38:20 etc/pki/CA/private
drwxr-xr-x root   root          0 Tue, 2023-08-15 17:03:49 etc/pki/nssdb
-rw-r--r-- root   root      65536 Tue, 2019-12-10 20:31:09 etc/pki/nssdb/cert8.db
-rw-r--r-- root   root       9216 Tue, 2023-08-15 17:03:49 etc/pki/nssdb/cert9.db
-rw-r--r-- root   root      16384 Tue, 2019-12-10 20:31:09 etc/pki/nssdb/key3.db
-rw-r--r-- root   root      11264 Tue, 2023-08-15 17:03:49 etc/pki/nssdb/key4.db
-rw-r--r-- root   root        451 Wed, 2019-11-27 19:47:26 etc/pki/nssdb/pkcs11.txt
-rw-r--r-- root   root      16384 Tue, 2019-12-10 20:31:10 etc/pki/nssdb/secmod.db
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:07:06 etc/bash_completion.d
-rw-r--r-- root   root      11272 Wed, 2020-04-01 08:03:26 etc/bash_completion.d/yum-utils.bash
-rw-r--r-- root   root        829 Wed, 2020-02-05 15:58:44 etc/bash_completion.d/iprutils
-rw-r--r-- root   root       1458 Wed, 2019-11-27 18:32:35 etc/bash_completion.d/redefine_filedir
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:26 etc/xdg
drwxr-xr-x root   root          0 Wed, 2018-04-11 07:59:55 etc/xdg/autostart
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:26 etc/xdg/systemd
lrwxrwxrwx root   root         18 Fri, 2020-05-01 01:06:26 etc/xdg/systemd/user -> ../../systemd/user
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:31 etc/prelink.conf.d
-rw-r--r-- root   root        184 Tue, 2019-12-10 22:51:14 etc/prelink.conf.d/nss-softokn-prelink.conf
-rw-r--r-- root   root         57 Wed, 2017-08-02 15:47:48 etc/prelink.conf.d/fipscheck.conf
-rw-r--r-- root   root        220 Fri, 2020-04-03 20:24:58 etc/prelink.conf.d/grub2.conf
drwxr-xr-x root   root          0 Fri, 2017-08-04 16:45:50 etc/chkconfig.d
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:05:22 etc/pkcs11
drwxr-xr-x root   root          0 Sat, 2017-08-05 02:36:45 etc/pkcs11/modules
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:05:58 etc/security
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:05:58 etc/security/limits.d
-rw-r--r-- root   root        191 Wed, 2020-04-01 06:59:56 etc/security/limits.d/20-nproc.conf
drwxr-xr-x root   root          0 Wed, 2020-04-01 06:59:50 etc/security/namespace.d
drwxr-xr-x root   root          0 Wed, 2020-04-01 06:59:48 etc/security/console.apps
-rw-r--r-- root   root       4564 Wed, 2020-04-01 06:59:45 etc/security/access.conf
-rw-r--r-- root   root       1718 Tue, 2011-12-06 23:16:48 etc/security/pwquality.conf
-rw-r--r-- root   root         82 Wed, 2020-04-01 06:59:46 etc/security/chroot.conf
-rw-r--r-- root   root        604 Wed, 2020-04-01 06:59:48 etc/security/console.handlers
-rw-r--r-- root   root        939 Wed, 2020-04-01 06:59:48 etc/security/console.perms
drwxr-xr-x root   root          0 Wed, 2020-04-01 06:59:48 etc/security/console.perms.d
-rw-r--r-- root   root       3635 Wed, 2020-04-01 06:59:49 etc/security/group.conf
-rw-r--r-- root   root       2422 Wed, 2020-04-01 06:59:49 etc/security/limits.conf
-rw-r--r-- root   root       1440 Wed, 2020-04-01 06:59:50 etc/security/namespace.conf
-rwxr-xr-x root   root       1019 Wed, 2020-04-01 06:59:50 etc/security/namespace.init
-rw-r--r-- root   root       2972 Wed, 2020-04-01 06:59:48 etc/security/pam_env.conf
-rw-r--r-- root   root        419 Wed, 2020-04-01 06:59:51 etc/security/sepermit.conf
-rw-r--r-- root   root       2179 Wed, 2020-04-01 06:59:52 etc/security/time.conf
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:59 etc/sasl2
-rw-r--r-- root   root         49 Wed, 2020-04-01 07:08:18 etc/sasl2/smtpd.conf
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:08:36 etc/openldap
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:12 etc/openldap/certs
-rw-r--r-- root   root      16384 Fri, 2020-05-01 01:06:12 etc/openldap/certs/secmod.db
-rw-r--r-- root   root      65536 Fri, 2020-05-01 01:06:12 etc/openldap/certs/cert8.db
-rw-r--r-- root   root      16384 Fri, 2020-05-01 01:06:12 etc/openldap/certs/key3.db
-rw-r--r-- root   root        365 Fri, 2020-05-01 01:08:36 etc/openldap/ldap.conf
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:25 etc/depmod.d
-rw-r--r-- root   root        116 Wed, 2020-04-01 05:58:15 etc/depmod.d/dist.conf
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:33 etc/sysctl.d
lrwxrwxrwx root   root         14 Fri, 2020-05-01 01:06:33 etc/sysctl.d/99-sysctl.conf -> ../sysctl.conf
drwxr-xr-x root   root          0 Tue, 2020-04-07 17:38:09 etc/tmpfiles.d
drwxr-xr-x root   root          0 Thu, 2020-04-02 16:29:42 etc/NetworkManager
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:07:01 etc/NetworkManager/dispatcher.d
-rwxr-xr-x root   root        175 Mon, 2019-08-19 14:34:03 etc/NetworkManager/dispatcher.d/00-netreport
-rwxr-xr-x root   root       1123 Wed, 2019-11-27 18:44:07 etc/NetworkManager/dispatcher.d/11-dhclient
-rwxr-xr-x root   root        428 Wed, 2018-09-19 17:38:15 etc/NetworkManager/dispatcher.d/20-chrony
drwxr-xr-x root   root          0 Wed, 2020-04-01 06:56:07 etc/NetworkManager/dispatcher.d/no-wait.d
drwxr-xr-x root   root          0 Wed, 2020-04-01 06:56:05 etc/NetworkManager/dispatcher.d/pre-down.d
drwxr-xr-x root   root          0 Wed, 2020-04-01 06:56:07 etc/NetworkManager/dispatcher.d/pre-up.d
drwxr-xr-x root   root          0 Wed, 2020-04-01 06:56:05 etc/NetworkManager/conf.d
drwxr-xr-x root   root          0 Wed, 2020-04-01 06:56:05 etc/NetworkManager/dnsmasq-shared.d
drwxr-xr-x root   root          0 Wed, 2020-04-01 06:56:05 etc/NetworkManager/dnsmasq.d
drwxr-xr-x root   root          0 Wed, 2020-04-01 06:56:05 etc/NetworkManager/system-connections
-rw-r--r-- root   root       2145 Wed, 2020-04-01 06:56:07 etc/NetworkManager/NetworkManager.conf
drwxr-xr-x root   root          0 Wed, 2020-04-01 05:50:02 etc/statetab.d
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:40 etc/samba
-rw-r--r-- root   root         20 Thu, 2020-04-02 16:24:09 etc/samba/lmhosts
-rw-r--r-- root   root        706 Thu, 2020-04-02 16:24:09 etc/samba/smb.conf
-rw-r--r-- root   root      11327 Thu, 2020-04-02 16:24:09 etc/samba/smb.conf.example
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:52 etc/gssproxy
drwxr-x--- root   root          0 Fri, 2020-05-01 01:06:51 etc/firewalld
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:54 etc/vmware-tools
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:53 etc/vmware-tools/scripts
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:53 etc/vmware-tools/scripts/vmware
-rwxr-xr-x root   root      15148 Thu, 2020-04-02 18:58:21 etc/vmware-tools/scripts/vmware/network
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:54 etc/vmware-tools/GuestProxyData
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:54 etc/vmware-tools/GuestProxyData/server
-rw-r--r-- root   root       1196 Fri, 2020-05-01 01:06:54 etc/vmware-tools/GuestProxyData/server/cert.pem
drwx------ root   root          0 Fri, 2020-05-01 01:06:54 etc/vmware-tools/GuestProxyData/trusted
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:53 etc/vmware-tools/vgauth
drwxr-xr-x root   root          0 Fri, 2020-05-01 01:06:53 etc/vmware-tools/vgauth/schemas
-rw-r--r-- root   root       5783 Thu, 2020-04-02 18:58:21 etc/vmware-tools/vgauth/schemas/XMLSchema-hasFacetAndProperty.xsd
-rw-r--r-- root   root       1215 Thu, 2020-04-02 18:58:21 etc/vmware-tools/vgauth/schemas/XMLSchema-instance.xsd
-rw-r--r-- root   root      16072 Thu, 2020-04-02 18:58:21 etc/vmware-tools/vgauth/schemas/XMLSchema.dtd
-rw-r--r-- root   root      86435 Thu, 2020-04-02 18:58:21 etc/vmware-tools/vgauth/schemas/XMLSchema.xsd
-rw-r--r-- root   root        473 Thu, 2020-04-02 18:58:21 etc/vmware-tools/vgauth/schemas/catalog.xml
-rw-r--r-- root   root       6356 Thu, 2020-04-02 18:58:21 etc/vmware-tools/vgauth/schemas/datatypes.dtd
-rw-r--r-- root   root      13160 Thu, 2020-04-02 18:58:21 etc/vmware-tools/vgauth/schemas/saml-schema-assertion-2.0.xsd
-rw-r--r-- root   root       5146 Thu, 2020-04-02 18:58:21 etc/vmware-tools/vgauth/schemas/xenc-schema.xsd
-rw-r--r-- root   root       8768 Thu, 2020-04-02 18:58:21 etc/vmware-tools/vgauth/schemas/xml.xsd
-rw-r--r-- root   root      10144 Thu, 2020-04-02 18:58:21 etc/vmware-tools/vgauth/schemas/xmldsig-core-schema.xsd
-rw-r--r-- root   root        453 Thu, 2020-04-02 18:58:21 etc/vmware-tools/guestproxy-ssl.conf
-rwxr-xr-x root   root       4364 Thu, 2020-04-02 18:58:21 etc/vmware-tools/poweroff-vm-default
-rwxr-xr-x root   root       4364 Thu, 2020-04-02 18:58:21 etc/vmware-tools/poweron-vm-default
-rwxr-xr-x root   root       4364 Thu, 2020-04-02 18:58:21 etc/vmware-tools/resume-vm-default
-rwxr-xr-x root   root       1478 Thu, 2020-04-02 18:58:21 etc/vmware-tools/statechange.subr
-rwxr-xr-x root   root       4364 Thu, 2020-04-02 18:58:21 etc/vmware-tools/suspend-vm-default
-rw-r--r-- root   root         60 Thu, 2020-04-02 18:58:21 etc/vmware-tools/vgauth.conf
drwxr-x--- root   root          0 Tue, 2023-08-15 17:03:06 etc/audit
drwxr-x--- root   root          0 Fri, 2020-05-01 01:09:26 etc/sudoers.d
```

Распакуем 1 файл из бэкапа:

```
[borg@client ~]$ borg extract borg@192.168.56.20:/var/backup/::etc-2023-08-17_09:56:57 etc/hostname
Enter passphrase for key ssh://borg@192.168.56.20/var/backup:
Enter passphrase for key ssh://borg@192.168.56.20/var/backup:
[borg@client ~]$ pwd
/home/borg
[borg@client ~]$ ls
etc
[borg@client ~]$ cd etc
[borg@client etc]$ ls
hostname
```

Автоматизируем создание бэкапов с помощью systemd
Создаем сервис и таймер в каталоге /etc/systemd/system/

```
# /etc/systemd/system/borg-backup.service
[Unit]
Description=Borg Backup

[Service]
Type=oneshot

# Парольная фраза
Environment="BORG_PASSPHRASE=Qwerty"
# Репозиторий
Environment=REPO=borg@192.168.56.20:/var/backup/
# Что бэкапим
Environment=BACKUP_TARGET=/etc

# Создание бэкапа
ExecStart=/bin/borg create --stats ${REPO}::etc-{now:%%Y-%%m-%%d_%%H:%%M:%%S} ${BACKUP_TARGET}

# Проверка бэкапа
ExecStart=/bin/borg check ${REPO}

# Очистка старых бэкапов
ExecStart=/bin/borg prune --keep-daily 90 --keep-monthly 12 --keep-yearly 1 ${REPO}




# /etc/systemd/system/borg-backup.timer
[Unit]
Description=Borg Backup

[Timer]
OnUnitActiveSec=5min

[Install]
WantedBy=timers.target
```

Включаем и запускаем службу таймера

```
[root@client ~]# systemctl enable borg-backup.timer
Created symlink from /etc/systemd/system/timers.target.wants/borg-backup.timer to /etc/systemd/system/borg-backup.timer.
[root@client ~]# systemctl start borg-backup.timer
[root@client ~]# systemctl status borg-backup.timer
● borg-backup.timer - Borg Backup
   Loaded: loaded (/etc/systemd/system/borg-backup.timer; enabled; vendor preset: disabled)
   Active: active (elapsed) since Thu 2023-08-17 10:50:38 MSK; 8min ago

Aug 17 10:50:38 client systemd[1]: Started Borg Backup.
```

Проверяем результат:

```
[borg@client ~]$ borg list borg@192.168.56.20:/var/backup/
Enter passphrase for key ssh://borg@192.168.56.20/var/backup:
etc-2023-08-17_09:56:57              Thu, 2023-08-17 09:57:05 [cf4a80c6b1cf261417012dabf35b7a724f2325ede279d858697e01a98f239ed8]
etc-2023-08-17_11:22:08              Thu, 2023-08-17 11:22:10 [c29ae761f3dad138aa3c24d70e536b534374e0b3fa933ec3676d8947b4a75a3d]
etc-2023-08-17_11:26:01              Thu, 2023-08-17 11:26:04 [6ebb5c33235bd2a6782f89d27aec89fe70fa4ce4fcbe3fff9faa5e0542815516]
etc-2023-08-17_11:31:11              Thu, 2023-08-17 11:31:13 [1a8527d31cba235d8264f613cdba698bc10b7db5c554b4e29184e90b637c1596]
etc-2023-08-17_11:36:48              Thu, 2023-08-17 11:36:50 [e01d04ad81b82e101a6a1b95ecc3f60f0c146e36c2f31c754657609471776c3b]
etc-2023-08-17_11:42:48              Thu, 2023-08-17 11:42:50 [e784c6bf88f1765d556ec10c3ae9ceb16083949737f45eb4b44340c3ccc4dea1]
etc-2023-08-17_11:48:23              Thu, 2023-08-17 11:48:25 [caabc51e8572bb134ef05317cf0481ca4312d5033a1fd7dbd7252595613f51b5]
etc-2023-08-17_11:53:48              Thu, 2023-08-17 11:53:50 [0f96d5d122cf706161358de8cb98b1c19e335ead4c95f8e0d5c05e649f36b67e]
etc-2023-08-17_11:59:48              Thu, 2023-08-17 11:59:50 [9ae485e43ee1dbbda4bda7b651b81859fe83d9f8ce05163a1bbfe39ad69b17da]
etc-2023-08-17_12:05:03              Thu, 2023-08-17 12:05:05 [79ea86d6825481512adc64a5e3f8d9201a52ce4f6d164c86f07b7c48ac78a221]
etc-2023-08-17_12:08:05              Thu, 2023-08-17 12:08:07 [8efc021793927f183c4849a3d1acc3a849a172bc3a6b41008eea5e433c215ac1]
etc-2023-08-17_12:17:32              Thu, 2023-08-17 12:17:35 [5c6498e6c559e35b3c8b1a3f950200dc991c70110c8dc1070e4e42a80e2adc9a]
etc-2023-08-17_12:23:21              Thu, 2023-08-17 12:23:23 [9edaaac030fca0ade95549452502a7f5acc22e18b5b9a7c0030fab3057796064]
etc-2023-08-17_12:49:02              Thu, 2023-08-17 12:49:04 [8e8dc3843e69613f87ae74004289c111be9718cf2e8e0cf8392585e8312a41ac]
```










