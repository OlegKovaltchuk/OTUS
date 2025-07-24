# OTUS
Administrator Linux. Professional


**Домашнее задание 4**

Заходим на сервер: kod_otus 
Дальнейшие действия выполняются от пользователя root. Переходим в
root пользователя
```
oleg@kodotus01:~$ sudo -i
```
**1. Устанавливаем ZFS **

Устанавливаем пакет zfsutils-linux
```
root@kodotusnfsserver:~# apt install zfsutils-linux
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following additional packages will be installed:
  libnvpair3linux libuutil3linux libzfs4linux libzpool5linux zfs-zed
Suggested packages:
  nfs-kernel-server samba-common-bin zfs-initramfs | zfs-dracut
The following NEW packages will be installed:
  libnvpair3linux libuutil3linux libzfs4linux libzpool5linux zfs-zed zfsutils-linux
0 upgraded, 6 newly installed, 0 to remove and 64 not upgraded.
Need to get 2355 kB of archives.
After this operation, 7399 kB of additional disk space will be used.
Do you want to continue? [Y/n] y

```
Смотрим информацию о пулах

```
root@kodotus01:~# zpool list
no pools available
```


**2. Определение алгоритма с наилучшим сжатием**

Смотрим список всех дисков, которые есть в виртуальной машине:
```

root@kodotus01:~# lsblk
NAME                      MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINTS
sda                         8:0    0   30G  0 disk
├─sda1                      8:1    0    1M  0 part
├─sda2                      8:2    0    2G  0 part  /boot
└─sda3                      8:3    0   28G  0 part
  └─ubuntu--vg-ubuntu--lv 252:0    0   14G  0 lvm   /
sdb                         8:16   0   20G  0 disk
├─sdb1                      8:17   0    1G  0 part
│ └─md127                   9:127  0 1022M  0 raid1
├─sdb2                      8:18   0    1K  0 part
├─sdb5                      8:21   0    1G  0 part
│ └─md127                   9:127  0 1022M  0 raid1
└─sdb6                      8:22   0    1G  0 part
sdc                         8:32   0    5G  0 disk
sdd                         8:48   0    1G  0 disk
sde                         8:64   0    1G  0 disk
sdf                         8:80   0    1G  0 disk
sdg                         8:96   0    1G  0 disk
sdh                         8:112  0    1G  0 disk
sdi                         8:128  0    1G  0 disk
sdj                         8:144  0    1G  0 disk
sdk                         8:160  0    1G  0 disk
sdl                         8:176  0    1G  0 disk
sdm                         8:192  0    1G  0 disk

```

Создаём четыре пула из двух дисков в режиме RAID 1:

```
root@kodotus01:~# zpool create test_otus1 mirror /dev/sdf  /dev/sdg
root@kodotus01:~# zpool create test_otus2 mirror /dev/sdh  /dev/sdi
root@kodotus01:~# zpool create test_otus3 mirror /dev/sdj  /dev/sdk 
root@kodotus01:~# zpool create test_otus4 mirror /dev/sdl  /dev/sdm
```

Смотрим информацию о пулах: zpool list

```
root@kodotus01:/# zpool list
NAME         SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
test_otus1   960M   408K   960M        -         -     0%     0%  1.00x    ONLINE  -
test_otus2   960M   492K   960M        -         -     0%     0%  1.00x    ONLINE  -
test_otus3   960M   492K   960M        -         -     0%     0%  1.00x    ONLINE  -
test_otus4   960M   372K   960M        -         -     0%     0%  1.00x    ONLINE  -
```


Добавим разные алгоритмы (lzjb, lz4, gzip-9, zle) сжатия в каждую файловую систему соответственно:

```
root@kodotus01:/# zfs set compression=lzjb test_otus1
root@kodotus01:/# zfs set compression=lz4 test_otus2
root@kodotus01:/# zfs set compression=gzip-9 test_otus3
root@kodotus01:/# zfs set compression=zle test_otus4

```

Проверим, что все файловые системы имеют разные методы сжатия:

```
root@kodotus01:/# zfs get all | grep compression
test_otus1  compression           lzjb                   local
test_otus2  compression           lz4                    local
test_otus3  compression           gzip-9                 local
test_otus4  compression           zle                    local

```

Копируем один и тот же текстовый файл во все пулы:

```
root@kodotus01:/var/log# for i in {1..4}; do cp -r /var/log/auth.log.1 /test_otus$i; done

```

Проверим, что файл был скачан во все пулы:

```

root@kodotus01:/var/log# ls -l /test_otus*
/test_otus:
total 0

/test_otus1:
total 57
-rw-r----- 1 root root 291852 Jul 15 21:05 auth.log.1

/test_otus2:
total 57
-rw-r----- 1 root root 291852 Jul 15 21:05 auth.log.1

/test_otus3:
total 37
-rw-r----- 1 root root 291852 Jul 15 21:05 auth.log.1

/test_otus4:
total 297
-rw-r----- 1 root root 291852 Jul 15 21:05 auth.log.1
```

Проверим, сколько места занимает один и тот же файл в разных пулах
и проверим степень сжатия файлов:

```
root@kodotus01:/var/log# zfs list
NAME         USED  AVAIL  REFER  MOUNTPOINT
test_otus1   584K   831M   152K  /test_otus1
test_otus2   584K   831M   152K  /test_otus2
test_otus3   552K   831M   132K  /test_otus3
test_otus4   788K   831M   392K  /test_otus4

root@kodotus01:/var/log# zfs get all | grep compressratio | grep -v ref
test_otus1  compressratio         2.47x                  -
test_otus2  compressratio         2.47x                  -
test_otus3  compressratio         2.74x                  -
test_otus4  compressratio         1.21x                  -
```

**Таким образом, у нас получается, что алгоритм gzip-9 самый
эффективный по сжатию.**

Запрос сразу всех параметром файловой системы:

```
root@kodotus01:/# zfs get all test_otus1
NAME        PROPERTY              VALUE                  SOURCE
test_otus1  type                  filesystem             -
test_otus1  creation              Mon Jul 14 13:04 2025  -
test_otus1  used                  584K                   -
test_otus1  available             831M                   -
test_otus1  referenced            152K                   -
test_otus1  compressratio         2.47x                  -
test_otus1  mounted               yes                    -
test_otus1  quota                 none                   default
test_otus1  reservation           none                   default
test_otus1  recordsize            128K                   default
test_otus1  mountpoint            /test_otus1            default
test_otus1  sharenfs              off                    default
test_otus1  checksum              on                     default
test_otus1  compression           lzjb                   local
test_otus1  atime                 on                     default
test_otus1  devices               on                     default
test_otus1  exec                  on                     default
test_otus1  setuid                on                     default
test_otus1  readonly              off                    default
test_otus1  zoned                 off                    default
test_otus1  snapdir               hidden                 default
test_otus1  aclmode               discard                default
test_otus1  aclinherit            restricted             default
test_otus1  createtxg             1                      -
test_otus1  canmount              on                     default
test_otus1  xattr                 on                     default
test_otus1  copies                1                      default
test_otus1  version               5                      -
test_otus1  utf8only              off                    -
test_otus1  normalization         none                   -
test_otus1  casesensitivity       sensitive              -
test_otus1  vscan                 off                    default
test_otus1  nbmand                off                    default
test_otus1  sharesmb              off                    default
test_otus1  refquota              none                   default
test_otus1  refreservation        none                   default
test_otus1  guid                  16775919799391520570   -
test_otus1  primarycache          all                    default
test_otus1  secondarycache        all                    default
test_otus1  usedbysnapshots       0B                     -
test_otus1  usedbydataset         152K                   -
test_otus1  usedbychildren        432K                   -
test_otus1  usedbyrefreservation  0B                     -
test_otus1  logbias               latency                default
test_otus1  objsetid              54                     -
test_otus1  dedup                 off                    default
test_otus1  mlslabel              none                   default
test_otus1  sync                  standard               default
test_otus1  dnodesize             legacy                 default
test_otus1  refcompressratio      4.57x                  -
test_otus1  written               152K                   -
test_otus1  logicalused           564K                   -
test_otus1  logicalreferenced     430K                   -
test_otus1  volmode               default                default
test_otus1  filesystem_limit      none                   default
test_otus1  snapshot_limit        none                   default
test_otus1  filesystem_count      none                   default
test_otus1  snapshot_count        none                   default
test_otus1  snapdev               hidden                 default
test_otus1  acltype               off                    default
test_otus1  context               none                   default
test_otus1  fscontext             none                   default
test_otus1  defcontext            none                   default
test_otus1  rootcontext           none                   default
test_otus1  relatime              on                     default
test_otus1  redundant_metadata    all                    default
test_otus1  overlay               on                     default
test_otus1  encryption            off                    default
test_otus1  keylocation           none                   default
test_otus1  keyformat             none                   default
test_otus1  pbkdf2iters           0                      default
test_otus1  special_small_blocks  0                      default

```
C помощью команды grep можно уточнить конкретный параметр,
например:

```
root@kodotus01:~# zfs get available test_otus1
NAME        PROPERTY   VALUE  SOURCE
test_otus1  available  831M   -

```

Тип:

```
root@kodotus01:~# zfs get readonly test_otus1
NAME        PROPERTY  VALUE   SOURCE
test_otus1  readonly  off     default
```

Значение recordsize
```
root@kodotus01:~# zfs get recordsize test_otus1
NAME        PROPERTY    VALUE    SOURCE
test_otus1  recordsize  128K     default

```

Тип сжатия (или параметр отключения):
```
root@kodotus01:~# zfs get compression test_otus1
NAME        PROPERTY     VALUE           SOURCE
test_otus1  compression  lzjb            local
```

Тип контрольной суммы:
```
root@kodotus01:~# zfs get checksum test_otus1
NAME        PROPERTY  VALUE      SOURCE
test_otus1  checksum  on         default
```

