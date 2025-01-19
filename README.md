# Домашнее задание: Работа с LVM

Цель работы: научиться использовать систему управления логическими томами (LVM) .

Для выполнения домашнего задания использовать прилагаемую методичку.

Что нужно сделать?

На имеющемся образе (centos/7 1804.2)
 https://gitlab.com/otus_linux/stands-03-lvm

/dev/mapper/VolGroup00-LogVol00 38G 738M 37G 2% 
 
  1. Уменьшить том под / 8GB
  2. Выделить том под /var (/var - сделать в mirror)
  3. Выделить том под /home
  4. Работа со снапшотами:
     
     Cгенерировать файлы в /home/;
	   Cнять снэпшот;
	   Удалить часть файлов;
	   Восстановиться со снэпшота.


  # Выполнение


## 1. Уменьшить том под / 8GB

Создаю в домашней директории Vagrantfile, в тело данного файла копирую содержимое Vagrantfile репозитория git-a: https://gitlab.com/otus_linux/stands-03-lvm.
Собираю стенд командой:
``` [nur@test hw-4]$ vagrant up ```
 
Подключаюсь к стенду:

``` [nur@test hw-4]$ vagrant ssh ```

Cмотрим какие блочные устройства и тома имеем в наличие:

``` [root@lvm ~]# lsblk

  > NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT

  >   sda                       8:0    0   40G   0    disk

  >  ├─sda1                    8:1    0    1M    0    part

  >  ├─sda2                    8:2    0    1G    0    part /boot

  >  └─sda3                    8:3    0   39G    0    part

  >    ├─VolGroup00-LogVol00 253:0    0 37.5G  0 lvm  /

  >    └─VolGroup00-LogVol01 253:1    0  1.5G  0 lvm  [SWAP]

  >  sdb                       8:16   0   10G  0 disk
 
  >  sdc                       8:32   0    2G  0 disk
 
  >  sdd                       8:48   0    1G  0 disk
 
  >  sde                       8:64   0    1G  0 disk
```
Устанавливаем пакет xfsdump:

``` [vagrant@lvm ~]$ yum install -y xfsdump ```
 
Подготавливаем временный том для / раздела:

Помечаем блочное устройство "sdb" как физическое устройство:

``` [root@lvm ~]# pvcreate /dev/sdb 
 > Physical volume "/dev/sdb" successfully created.
```  
 Создаем volume-группу "vg_root" и включаем в группу физическое устройство "sdb":
 
 ``` [root@lvm ~]# vgcreate vg_root /dev/sdb 
 > Volume group "vg_root" successfully created
``` 
 Создаем логический том "lv_root" на основе volume-группы "vg_root", со всем доступным дисковым пространством:
 
 ``` [root@lvm ~]# lvcreate -n lv_root -l+100%FREE /dev/vg_root
 > Logical volume "lv_root" created.
```  
 Создаем на томе "lv_root" файловую систему и монтируем её:
 
 ``` [root@lvm ~]# mkfs.xfs /dev/vg_root/lv_root 
 
 >  meta-data=/dev/vg_root/lv_root   isize=512    agcount=4, agsize=655104 blks

 >           =                       sectsz=512   attr=2, projid32bit=1

 >           =                       crc=1        finobt=0, sparse=0

 >  data      =                       bsize=4096   blocks=2620416, imaxpct=25

 >           =                       sunit=0      swidth=0 blks

 >  log       =internal log           bsize=4096   blocks=2560, version=2

 >           =                       sectsz=512   sunit=0 blks, lazy-count=1

 >  realtime  =none                   extsz=4096   blocks=0, rtextents=0

````
  
 ``` [root@lvm ~]# mount /dev/vg_root/lv_root /mnt ```
 
Скопируем все данные с логического тома "LogVol00" на смонтированный том "lv_root"

  ``` [root@lvm ~]# xfsdump -J - /dev/VolGroup00/LogVol00 | xfsrestore -J - /mnt 
 > .........
 
 > xfsrestore: Restore Status: SUCCESS
````
  
Проверим содержимое /mnt:

 ``` [root@lvm ~]# ls /mnt ```
 > bin  boot  dev  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  vagrant  var
 
 Подмонтируем системные каталоги для chroot:
 
 ``` [root@lvm ~]# for i in /proc/ /sys/ /dev/ /run/ /boot/; \ ```
 
 ```> do mount --bind $i /mnt/$i; done ```
 
 Подключимся по chroot:
 
 ``` [root@lvm ~]# chroot /mnt/ ```
 
 Обновляем grub:
 
 ``` [root@lvm /]#  grub2-mkconfig -o /boot/grub2/grub.cfg ```
 > Generating grub configuration file ...

 > Found linux image: /boot/vmlinuz-3.10.0-1160.114.2.el7.x86_64

 > Found initrd image: /boot/initramfs-3.10.0-1

 > 160.114.2.el7.x86_64.img

 > Found linux image: /boot/vmlinuz-3.10.0-862.2.3.el7.x86_64

 > Found initrd image: /boot/initramfs-3.10.0-862.2.3.el7.x86_64.img

 > done
 
 Обновляем образ initrd:
 
 ``` [root@lvm /]# cd /boot ; for i in `ls initramfs-*img`; \ ```
 
 ``` > do dracut -v $i `echo $i|sed "s/initramfs-//g; \ ```
 
 ``` > s/.img//g"` --force; done ```
 
 > *** Creating initramfs image file '/boot/initramfs-3.10.0-862.2.3.el7.x86_64.img' done ***
 
 Редактируем фаил /boot/grub2/grub.cfg, меняем значение rd.lvm.lv=VolGroup00/LogVol00 на rd.lvm.lv=vg_root/lv_root:
 
  ``` [root@lvm boot]# vi /boot/grub2/grub.cfg ```
  
 > linux16 /vmlinuz-3.10.0-1160.114.2.el7.x86_64 root=/dev/mapper/vg_root-lv_root ro no_timer_check console=tty0

 > console=ttyS0,115200n8 net.ifnames=0 biosdevname=0

 > elevator=noop crashkernel=auto rd.lvm.lv=vg_root/lv_root rd.lvm.lv=vg_root/lv_root rhgb quiet
 
  Перезагружаем систему:
  
  ``` [root@lvm ~]# shutdown -r now ```
 
  Отобразим имеющиеся тома на системе: 
  
  ``` [root@lvm /]# lsblk ```
  
 >  NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT

 > sda                       8:0    0   40G  0 disk

 > ├─sda1                    8:1    0    1M  0 part

 > ├─sda2                    8:2    0    1G  0 part /boot

 > └─sda3                    8:3    0   39G  0 part

 >   ├─VolGroup00-LogVol01 253:1    0  1.5G  0 lvm  [SWAP]

 >   └─VolGroup00-LogVol00 253:2    0 37.5G  0 lvm
  
 > sdb                       8:16   0   10G  0 disk
 
 > └─vg_root-lv_root       253:0    0   10G  0 lvm  /

 > sdc                       8:32   0    2G  0 disk
 
 > sdd                       8:48   0    1G  0 disk
 
 > sde                       8:64   0    1G  0 disk 
 
  Изменим размер раздела "LogVol00" с 40G до 8G (удаляем и создаём том): 
  
  ``` [root@lvm ~]# lvremove /dev/VolGroup00/LogVol00 ```
  
 > Do you really want to remove active logical volume VolGroup00/LogVol00? [y/n]: y

 >  Logical volume "LogVol00" successfully removed
 
 ``` [root@lvm ~]# lvcreate -n VolGroup00/LogVol00 -L 8G /dev/VolGroup00 ```
 
 > WARNING: xfs signature detected on /dev/VolGroup00/LogVol00 at offset 0. Wipe it? [y/n]: y

 >  Wiping xfs signature on /dev/VolGroup00/LogVol00.

 >   Logical volume "LogVol00" created.
 
  Cоздадим на томе "LogVol00" файловую систему, смонтируем том на раздел /mnt,   
  cкопируем все данные из тома "lv_root" на вновь созданный "LogVol00": 
  
  ``` [root@lvm ~]# mkfs.xfs /dev/VolGroup00/LogVol00 ```
  
 > meta-data=/dev/VolGroup00/LogVol00 isize=512    agcount=4, agsize=524288 blks

 >          =                       sectsz=512   attr=2, projid32bit=1

 >          =                       crc=1        finobt=0, sparse=0

 > data     =                       bsize=4096   blocks=2097152, ima>xpct=25

 >          =                       sunit=0      swidth=0 blks

 > naming   =version 2              bsize=4096   ascii-ci=0 ftype=1

 > log      =internal log           bsize=4096   blocks=2560, version=2

 >          =                       sectsz=512   sunit=0 blks, lazy-count=1

 > realtime =none                   extsz=4096   blocks=0, rtextents=0

 ``` [root@lvm ~]# mount /dev/VolGroup00/LogVol00 /mnt ```
 
 ``` [root@lvm ~]#  xfsdump -J - /dev/vg_root/lv_root | \ ```
 
 ``` > xfsrestore -J - /mnt ```
 
 > .............

 > xfsrestore: Restore Status: SUCCESS
  
  Конфигурируем grub не редактируя grub.cfg:
  
  ``` [root@lvm ~]# for i in /proc/ /sys/ /dev/ /run/ /boot/; \ ```
  
  ``` > do mount --bind $i /mnt/$i; done ```
  
  ``` [root@lvm ~]#  chroot /mnt/ ```
  
  ``` [root@lvm /]# grub2-mkconfig -o /boot/grub2/grub.cfg ``` 
 > Generating grub configuration file ...

 > Found linux image: /boot/vmlinuz-3.10.0-1160.114.2.el7.x86_64

 > Found initrd image: /boot/initramfs-3.10.0-1160.114.2.el7.x86_64.img

 > Found linux image: /boot/vmlinuz-3.10.0-862.2.3.
 
 > Found initrd image: /boot/initramfs-3.10.0-862.2.3.el7.x86_64.img

 > done
 
 ``` [root@lvm /]# cd /boot ; for i in `ls initramfs-*img`; \ ``` 
 
 ```> do dracut -v $i `echo $i|sed "s/initramfs-//g; \```
 
 ```> s/.img//g"` --force; done ```
 
 > *** Creating initramfs image file '/boot/initramfs-3.10.0-862.2.3.el7.x86_64.img' done ***
 
 ## 2. Выделить том под /var (/var - сделать в mirror)
  
  Создадим зеркало на на блочных устройствах "sdc" и "sdd":  
  
  ``` [root@lvm boot]# pvcreate /dev/sdc /dev/sdd ```  
  >Physical volume "/dev/sdc" successfully created.

  >Physical volume "/dev/sdd" successfully created.
  
  Создадим volume-группу "vg_var" и логический том "lv_var" размером 950М: 
  
  ``` [root@lvm boot]# vgcreate vg_var /dev/sdc /dev/sdd ```
  
  > Volume group "vg_var" successfully created

  ``` [root@lvm boot]# lvcreate -L 950M -m1 -n lv_var vg_var ```
  
  > Rounding up size to full physical extent 952.00 MiB

  > Logical volume "lv_var" created.
 
 На "lv_var" создаём файловую систему и копируем на нее данные /var:
 
  ``` [root@lvm boot]# mkfs.ext4 /dev/vg_var/lv_var ```
  > ..........

  > Writing superblocks and filesystem accounting information: done
 
  ``` [root@lvm boot]# mount /dev/vg_var/lv_var /mnt ```
 
  ``` [root@lvm boot]# cp -aR /var/* /mnt/ ```
 
 Сохраняем содержимое текущего каталога /var и монтируем том "lv_var" в новый каталог "var":
 
  ``` [root@lvm boot]# mkdir /tmp/oldvar && mv /var/* /tmp/oldvar ```
 
  ``` [root@lvm boot]# umount /mnt ```

  ``` [root@lvm boot]# mount /dev/vg_var/lv_var /var ```
 
 Правим fstab для автоматического монтирования нового /var при загрузке системы:
 
  ``` [root@lvm boot]# echo "`blkid | grep var: | awk '{print $2}'` \ ```
  
  ``` > /var ext4 defaults 0 0" >> /etc/fstab ```
  
 Перезагружаем систему:
  
  ``` [root@lvm boot]# shutdown -r now ```
  
 Удаляем временный том "lv_root":
 
  ``` [root@lvm ~]# lvremove /dev/vg_root/lv_root ```
  
  > Do you really want to remove active logical volume vg_root/lv_root? [y/n]: y

  > Logical volume "lv_root" successfully removed

  ## 3. Выделить том под /home
 
  Создаем логический том "LogVol_Home" размером 2G на основе volume-группы "VolGroup00":
  
  ``` [root@lvm ~]#  lvcreate -n LogVol_Home -L 2G /dev/VolGroup00 ```
  > Logical volume "LogVol_Home" created.
  
  Создаём файловую систему на томе и примонтируем том в каталог /mnt:
  
  ``` [root@lvm ~]# mkfs.xfs /dev/VolGroup00/LogVol_Home ```
   
  ``` [root@lvm ~]# mount /dev/VolGroup00/LogVol_Home /mnt/ ```
  
  Копируем текущую директорию "home" в /mnt:
  
  ``` [root@lvm ~]# cp -aR /home/* /mnt/ ```
  
  Удаляем текущую директорию "home":
  
  ``` [root@lvm ~]# rm -rf /home/* ```
  
  Отмонтируем /mnt и монтируем том "LogVol_Home" в дирректорию /home:
  
  ``` [root@lvm ~]# umount /mnt ```
 
  ``` [root@lvm ~]# mount /dev/VolGroup00/LogVol_Home /home/ ```
 
  Редактируем fstab для автоматического монтирования /home:
  
  ``` [root@lvm ~]#  echo "`blkid | grep Home | awk '{print $2}'` \ ```
   
  ``` > /home xfs defaults 0 0" >> /etc/fstab ```
   
   ## 4. Работа со снапшотами 
   
   Сгенерировать файлы в /home:
   
   ``` [root@lvm ~]# touch /home/file{1..20} ```
   
   Снимаем снапшот:
   
   ``` [root@lvm ~]# lvcreate -L 100MB -s -n home_snap /dev/VolGroup00/LogVol_Home ```
   
   > Rounding up size to full physical extent 128.00 MiB

   > Logical volume "home_snap" created.
   
   Удаляем файлы и отмонтируем каталог /home:
   
   ``` [root@lvm ~]# rm -f /home/file{11..20} ```
   
   ``` [root@lvm ~]# umount /home ```
   
   Восстанавливаем данные из снапшота:
   
   ``` [root@lvm ~]# lvconvert --merge /dev/VolGroup00/home_snap ```
   
  > Merging of volume VolGroup00/home_snap started.

  > VolGroup00/LogVol_Home: Merged: 100.00%
   
   ``` [root@lvm ~]# mount /home ```
   
   ``` [root@lvm ~]# ls -al /home ```
   
 > total 0

 > drwxr-xr-x.  3 root    root    292 May  3 09:14 .

 > drwxr-xr-x. 18 root    root    239 May  3 09:05 ..

 > -rw-r--r--.  1 root    root      0 May  3 09:14 file1

 > -rw-r--r--.  1 root    root      0 May  3 09:14 file10

 > -rw-r--r--.  1 root    root      0 May  3 09:14 file11

 > -rw-r--r--.  1 root    root      0 May  3 09:14 file12

 > -rw-r--r--.  1 root    root      0 May  3 09:14 file13

 > -rw-r--r--.  1 root    root      0 May  3 09:14 file14

 > -rw-r--r--.  1 root    root      0 May  3 09:14 file15

 > -rw-r--r--.  1 root    root      0 May  3 09:14 file16

 > -rw-r--r--.  1 root    root      0 May  3 09:14 file17

 > -rw-r--r--.  1 root    root      0 May  3 09:14 file18

 > -rw-r--r--.  1 root    root      0 May  3 09:14 file19

 > -rw-r--r--.  1 root    root      0 May  3 09:14 file2

 > -rw-r--r--.  1 root    root      0 May  3 09:14 file20

 > -rw-r--r--.  1 root    root      0 May  3 09:14 file3

 > -rw-r--r--.  1 root    root      0 May  3 09:14 file4

 > -rw-r--r--.  1 root    root      0 May  3 09:14 file5

 > -rw-r--r--.  1 root    root      0 May  3 09:14 file6

 > -rw-r--r--.  1 root    root      0 May  3 09:14 file7

 > -rw-r--r--.  1 root    root      0 May  3 09:14 file8

 > -rw-r--r--.  1 root    root      0 May  3 09:14 file9

 > drwx------.  3 vagrant vagrant  74 May 12  2018 vagrant
