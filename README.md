                                    Домашнее задание

                                       Работа с LVM

 Цель работы: научиться использовать систему управления логическими томами (LVM) .

 Для выполнения домашнего задания использовать прилагаемую методичку.

 Что нужно сделать?

    На имеющемся образе (centos/7 1804.2)
 https://gitlab.com/otus_linux/stands-03-lvm

 /dev/mapper/VolGroup00-LogVol00 38G 738M 37G 2% 
 
  1. уменьшить том под / 8GB
  2. выделить том под /home
  3. выделить том под /var (/var - сделать в mirror)
  4. для /home - сделать том для снэпшотов
  5. прописать монтирование в fstab (попробовать с разными опциями и разными файловыми системами на выбор)
  6. Работа со снапшотами:
    * сгенерировать файлы в /home/
	* снять снэпшот
	* удалить часть файлов
	* восстановиться со снэпшота


  
 

					    Выполнение


	1. Уменьшить том под / 8GB

 Создаю в домашней директории Vagrantfile, в тело данного файла копирую содержимое Vagrantfile репозитория git-a:https://gitlab.com/otus_linux/stands-03-lvm.
 
 Собираю стенд командой:
[nur@test hw-4]$ vagrant up
 
 Подключаюсь к стенду: -[nur@test hw-4]$ vagrant ssh
 
 Устанавливаем пакет xfsdump: - [vagrant@lvm ~]$ yum install -y xfsdump
 
 Подготавливаем временный том для / раздела:
 
 Помечаем блочное устройство "sdb" как физическое устройство: -  [root@lvm ~]# pvcreate /dev/sdb
  Physical volume "/dev/sdb" successfully created.
  
 Создаем volume-группу "vg_root" и включаем в группу физическое устройство "sdb":
  [root@lvm ~]# vgcreate vg_root /dev/sdb
  Volume group "vg_root" successfully created
  
 Создаем логический том "lv_root" на основе volume-группы "vg_root", со всем доступным дисковом пространством:
 [root@lvm ~]# lvcreate -n lv_root -l+100%FREE /dev/vg_root
  Logical volume "lv_root" created.
  
 Создаем на томе "lv_root" файловую систему и монтируем её:
 
 [root@lvm ~]# mkfs.xfs /dev/vg_root/lv_root
  meta-data=/dev/vg_root/lv_root   isize=512    agcount=4, agsize=655104 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
  data     =                       bsize=4096   blocks=2620416, imaxpct=25
         =                       sunit=0      swidth=0 blks
  log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
  realtime =none                   extsz=4096   blocks=0, rtextents=0
  
 [root@lvm ~]# mount /dev/vg_root/lv_root /mnt
 
 Скопируем все данные с логического тома "LogVol00" на смонтированный том "lv_root":
  [root@lvm ~]# xfsdump -J - /dev/VolGroup00/LogVol00 | xfsrestore -J - /mnt
  .........
  xfsrestore: Restore Status: SUCCESS
  
 Проверим содержимое /mnt:
  [root@lvm ~]# ls /mnt
  bin  boot  dev  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  vagrant  var
  
