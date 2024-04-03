# otus_lp_lesson_7_zfs
OTUS Linux Professional Lesson #7 | Subject: ZFS
#### ПОРЯДОК ВЫПОЛНЕНИЯ:
Поднимаем виртуальную машину с 8 дополнительными дисками (используя Vagrantfile)
##### ЗАДАНИЕ 1. Определение алгоритма с наилучшим сжатием
1. Смотрим список всех дисков, которые есть в виртуальной машине:
   
   `lsblk`
2. Создаём 4 пула по два диска в каждом в режиме RAID 1:
   ```
   [root@zfs ~]# zpool create otus1 mirror /dev/sda /dev/sdb
   [root@zfs ~]# zpool create otus2 mirror /dev/sdc /dev/sdd
   [root@zfs ~]# zpool create otus3 mirror /dev/sde /dev/sdf
   [root@zfs ~]# zpool create otus4 mirror /dev/sdg /dev/sdh
   ```
   Команды для просмотра информацию о пулах:
   
   `zpool list` - показывает информацию о размере пула, количеству занятого и свободного места, дедупликации и т.д.
   
   `zpool status` - показывает информацию о каждом диске, состоянии сканирования и об ошибках чтения, записи и совпадения
3. Дальше можно создать датасеты, либо сразу использовать весь пул (если быть точнее, то "использовать рутовый датасет", так как при создании пула создается рутовый датасет с ограниченными возможностями). Рекомендуется создавать дополнительные датасеты по следующим причинам:
   - гибкость. каждый датасет можно настроить по своему, указав для датасета необходимые свойства, отличные от рутового (compression, encryption и т.п.) 
   - это полезно для снапшотов и zfs send/recv. Вроде как нельзя на рутовый датасет реплицировать, а значит что нельзя без лишних телодвижений восстановить рутовый датасет из резервной копии:
    > And you can't replicate to it, which means you can't restore from backup to it efficiently.
    >
    > Any attempt to zfs receive tank will fail. You can instead zfs receive tank/dataset, but if all of your applications, filesharing configs, etc all point directly to the root dataset at tank you'll have to       > reconfigure every last one of them to account for the difference.

    > By contrast, if you put your data--even if it is all in a single dataset!--in tank/dataset in the first place, when you screw it up later and have to restore from backup you can zfs receive tank/dataset and     > be done with it, no reconfiguration needed.

   Некоторые люди даже убирают монтирование рутового датасета, чтобы он не мешал при просмотре и чтобы не записать в него что-то случайно:
   
   `zfs set mountpoint=none poolname`,  или с опцией `"-m none"` при создании пула командой `zpool create` 

   Соответственно, если мы дальше создадим датасет на таком скрытом пуле, то нужно обязательно указать для него точку монтирования. Например:
   ```
   zfs set mountpoint=none otus1
   zfs set mountpoint=/mnt otus1/dataset1
   ```
   Мы пока не будем заморачиваться с отмонтированным рутовым датасетом. Просто создаем дополнительные датасеты в каждом пуле:
   ```
   [root@zfs ~]# zfs create otus1/dataset1
   [root@zfs ~]# zfs create otus2/dataset2
   [root@zfs ~]# zfs create otus3/dataset3
   [root@zfs ~]# zfs create otus4/dataset4
   ```
4. Добавим разные алгоритмы сжатия на каждый датасет:
   ```
   zfs set compression=lzjb otus1/dataset1
   zfs set compression=lz4 otus2/dataset2
   zfs set compression=gzip-9 otus3/dataset3
   zfs set compression=zle otus4/dataset4
   ```
   Проверим, что все датасеты имеют разные методы сжатия:
   ```
   zfs get all | grep compression
   ```
   *NOTE: Сжатие файлов будет работать только с файлами, которые были добавлены после включение настройки сжатия.*
5. Скачаем один и тот же текстовый файл во все пулы:
   ```
   [root@zfs ~]# for i in {1..4}; do wget -P /otus$i/dataset$i https://gutenberg.org/cache/epub/2600/pg2600.converter.log; done
   ```
   Проверим, что файл был скачан во все пулы:
   ```
   [root@zfs ~]# ls -l /otus*/dataset*
   ```
   Уже на этом этапе видно, что самый оптимальный метод сжатия у нас используется в датасете otus3/dataset3.
   
   Проверим, сколько места занимает один и тот же файл в разных пулах и проверим степень сжатия файлов:
   ```
   [root@zfs ~]# zfs list
   NAME             USED  AVAIL     REFER  MOUNTPOINT
   otus1           21.7M   330M     25.5K  /otus1
   otus1/dataset1  21.6M   330M     21.6M  /otus1/dataset1
   otus2           17.7M   334M     25.5K  /otus2
   otus2/dataset2  17.6M   334M     17.6M  /otus2/dataset2
   otus3           10.8M   341M     25.5K  /otus3
   otus3/dataset3  10.7M   341M     10.7M  /otus3/dataset3
   otus4           39.3M   313M     25.5K  /otus4
   otus4/dataset4  39.2M   313M     39.2M  /otus4/dataset4
   ```
   ```
   [root@zfs ~]# zfs get all | grep compressratio | grep -v ref
   otus1           compressratio         1.82x                  -
   otus1/dataset1  compressratio         1.82x                  -
   otus2           compressratio         2.23x                  -
   otus2/dataset2  compressratio         2.23x                  -
   otus3           compressratio         3.66x                  -
   otus3/dataset3  compressratio         3.67x                  -
   otus4           compressratio         1.00x                  -
   otus4/dataset4  compressratio         1.00x                  -

   ```
   Видим, что алгоритм gzip-9 самый эффективный по сжатию. Но скорее всего он будет и самый медленный. Говорят что оптимальный, который "почти-налету" сжимает и почти не грузит систему - **lz4**

##### ЗАДАНИЕ 2. Определение настроек пула
   
