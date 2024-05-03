`# Начну с выполненного задания. Отправлены 4 файла:`
1) README.md - описание всей проеланной работы.
2) Vagrantfile - итоговый вагрант-файл
3) Vagrantfile_old - изначальный рабочий вагрант-файл.
4) zfs.sh - скрипт для конфигурирования сервера

`# Сразу напишу, что при запуске vagrant-файла, предоставленного в методичке автоматически не устанавливалась ZFS, остальное ставилось. При этом руками всё доставлялось прекрасно.`
`# Я пробовал менять параметры, убил очень много времени.`
`# Пробовал в том числе обновлять пакеты, но ни к чему хорошему это не привело, стало только ещё хуже.`
`# И уже когда я не знал, что делать, заработал оригинальный файл с методички, непонятно почему. И всё работает, повторял несколько раз.`
`# Пишу, дабы быть честным. В чём проблема, не могу сказать. Каждый раз работа с вагрантом начинается со спецоперации.`

`# Задание 1`

lsblk                                               # просматриваем свои диски

[root@zfs vagrant]# lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0   40G  0 disk 
`-sda1   8:1    0   40G  0 part /
sdb      8:16   0  512M  0 disk 
sdc      8:32   0  512M  0 disk 
sdd      8:48   0  512M  0 disk 
sde      8:64   0  512M  0 disk 
sdf      8:80   0  512M  0 disk 
sdg      8:96   0  512M  0 disk 
sdh      8:112  0  512M  0 disk 
sdi      8:128  0  512M  0 disk

`# Создаём пул из двух дисков в режиме RAID 1`

zpool create otus1 mirror /dev/sdb /dev/sdc
zpool create otus4 mirror /dev/sdd /dev/sde
zpool create otus4 mirror /dev/sdf /dev/sdg
zpool create otus4 mirror /dev/sdh /dev/sdi

zpool list                                          # выводим информацию о пулах

[root@zfs vagrant]# zpool list
NAME    SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
otus1   480M   112K   480M        -         -     0%     0%  1.00x    ONLINE  -
otus2   480M  91.5K   480M        -         -     0%     0%  1.00x    ONLINE  -
otus3   480M  91.5K   480M        -         -     0%     0%  1.00x    ONLINE  -
otus4   480M  91.5K   480M        -         -     0%     0%  1.00x    ONLINE  -

`# пробуем разные варианты компрессии`

zfs set compression=lzjb otus1
zfs set compression=lz4 otus2
zfs set compression=gzip-9 otus3
zfs set compression=zle otus4

`# проверяем`

zfs get all | grep compression

[root@zfs vagrant]# zfs get all | grep compression
otus1  compression           lzjb                   local
otus2  compression           lz4                    local
otus3  compression           gzip-9                 local
otus4  compression           zle                    local

`# Скачаем один и тот же текстовый файл во все пулы: `
for i in {1..4}; do wget -P /otus$i https://gutenberg.org/cache/epub/2600/pg2600.converter.log; done

ls -l /otus*                                        # проверяем, что файл скачался

[root@zfs vagrant]# ls -l /otus*
/otus1:
total 22077
-rw-r--r--. 1 root root 41043469 May  2 07:54 pg2600.converter.log

/otus2:
total 17998
-rw-r--r--. 1 root root 41043469 May  2 07:54 pg2600.converter.log

/otus3:
total 10962
-rw-r--r--. 1 root root 41043469 May  2 07:54 pg2600.converter.log

/otus4:
total 40109
-rw-r--r--. 1 root root 41043469 May  2 07:54 pg2600.converter.log

zfs list                                            # Проверим, сколько места занимает один и тот же файл в разных пулах

[root@zfs vagrant]# zfs list
NAME    USED  AVAIL     REFER  MOUNTPOINT
otus1  21.7M   330M     21.6M  /otus1
otus2  17.7M   334M     17.6M  /otus2
otus3  10.8M   341M     10.7M  /otus3
otus4  39.3M   313M     39.2M  /otus4

zfs get all | grep compressratio | grep -v ref      # проверим степень сжатия файлов

[root@zfs vagrant]# zfs get all | grep compressratio | grep -v ref
otus1  compressratio         1.82x                  -
otus2  compressratio         2.23x                  -
otus3  compressratio         3.66x                  -
otus4  compressratio         1.00x                  -

`# делаем вывод, что gzip-9 наиболее эффективен для сжатия`

`# Определение настроек пула`

`# скачиваем архив в домашний каталог по рекомендациям из методички и разархивируем его.`
wget -O archive.tar.gz --no-check-certificate 'https://drive.usercontent.google.com/download?id=1MvrcEp-WgAQe57aDEzxSRalPAwbNN1Bb&export=download' 
tar -xzvf archive.tar.gz

ls -lah                                             # проверяем

[root@zfs vagrant]# ls -lah
total 7.0M
drwx------. 4 vagrant vagrant  115 May  3 09:04 .
drwxr-xr-x. 3 root    root      21 Apr 30  2020 ..
-rw-r--r--. 1 vagrant vagrant   18 Apr  1  2020 .bash_logout
-rw-r--r--. 1 vagrant vagrant  193 Apr  1  2020 .bash_profile
-rw-r--r--. 1 vagrant vagrant  231 Apr  1  2020 .bashrc
drwx------. 2 vagrant vagrant   29 May  2 21:27 .ssh
-rw-r--r--. 1 root    root    7.0M Dec  6 15:49 archive.tar.gz
drwxr-xr-x. 2 root    root      32 May 15  2020 zpoolexport

zpool import -d zpoolexport                         # Проверим, возможно ли импортировать данный каталог в пул

[root@zfs vagrant]# zpool import -d zpoolexport/
   pool: otus
     id: 6554193320433390805
  state: ONLINE
 action: The pool can be imported using its name or numeric identifier.
 config:

	otus                                 ONLINE
	  mirror-0                           ONLINE
	    /home/vagrant/zpoolexport/filea  ONLINE
	    /home/vagrant/zpoolexport/fileb  ONLINE

zpool import -d zpoolexport/ otus                   # Сделаем импорт данного пула

zpool status                                        # проверим: выведем информацию о составе импортированного пула

[root@zfs vagrant]# zpool status
  pool: otus
 state: ONLINE
  scan: none requested
config:

	NAME                                 STATE     READ WRITE CKSUM
	otus                                 ONLINE       0     0     0
	  mirror-0                           ONLINE       0     0     0
	    /home/vagrant/zpoolexport/filea  ONLINE       0     0     0
	    /home/vagrant/zpoolexport/fileb  ONLINE       0     0     0

errors: No known data errors

  pool: otus1
 state: ONLINE
  scan: none requested
config:

	NAME        STATE     READ WRITE CKSUM
	otus1       ONLINE       0     0     0
	  mirror-0  ONLINE       0     0     0
	    sdb     ONLINE       0     0     0
	    sdc     ONLINE       0     0     0

errors: No known data errors

  pool: otus2
 state: ONLINE
  scan: none requested
config:

	NAME        STATE     READ WRITE CKSUM
	otus2       ONLINE       0     0     0
	  mirror-0  ONLINE       0     0     0
	    sdd     ONLINE       0     0     0
	    sde     ONLINE       0     0     0

errors: No known data errors

  pool: otus3
 state: ONLINE
  scan: none requested
config:

	NAME        STATE     READ WRITE CKSUM
	otus3       ONLINE       0     0     0
	  mirror-0  ONLINE       0     0     0
	    sdf     ONLINE       0     0     0
	    sdg     ONLINE       0     0     0

errors: No known data errors

  pool: otus4
 state: ONLINE
  scan: none requested
config:

	NAME        STATE     READ WRITE CKSUM
	otus4       ONLINE       0     0     0
	  mirror-0  ONLINE       0     0     0
	    sdh     ONLINE       0     0     0
	    sdi     ONLINE       0     0     0

errors: No known data errors

`# Задание 2`

`# проверяем настройки пула и запрашиваем параметры файловой системы`
zpool get all otus
zfs get all otus

`# по домашнему заданию определяем:`

`# размер хранилища`

zfs get available otus

[root@zfs vagrant]# zfs get available otus
NAME  PROPERTY   VALUE  SOURCE
otus  available  350M   -

`# тип pool`

zfs get readonly otus

[root@zfs vagrant]# zfs get readonly otus
NAME  PROPERTY  VALUE   SOURCE
otus  readonly  off     default

`# значение recordsize`

zfs get recordsize otus

[root@zfs vagrant]# zfs get recordsize otus
NAME  PROPERTY    VALUE    SOURCE
otus  recordsize  128K     local

`# какое сжатие используется`

zfs get compression otus

[root@zfs vagrant]# zfs get compression otus
NAME  PROPERTY     VALUE     SOURCE
otus  compression  zle       local

`# какая контрольная сумма используется`

zfs get checksum otus

[root@zfs vagrant]# zfs get checksum otus
NAME  PROPERTY  VALUE      SOURCE
otus  checksum  sha256     local

`# Задание 3`

`# Работа со снапшотами`

wget -O otus_task2.file --no-check-certificate https://drive.usercontent.google.com/download?id=1wgxjih8YZ-cqLqaZVa0lA3h3Y029c3oI&export=download   # скачиваем файл, как указано в методичке

zfs receive otus/test@today < otus_task2.file       # восстановим файловую систему из снапшота

find /otus/test -name "secret_message"              # ищем secret_message в каталоге /otus/test

[root@zfs ~]# find /otus/test -name "secret_message"
/otus/test/task1/file_mess/secret_message

cat /otus/test/task1/file_mess/secret_message       #просмотрим содержимое

[root@zfs ~]# cat /otus/test/task1/file_mess/secret_message
https://otus.ru/lessons/linux-hl/
