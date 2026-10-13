1.	Задача: Добавьте в виртуальную машину несколько дисков

root@clean:/home/asur# sudo lshw -short | grep disk
/0/100/1.1/0.0.0  /dev/cdrom  disk        CD-ROM
/0/100/d/0        /dev/sda    disk        42GB VBOX HARDDISK
/0/100/d/1        /dev/sdb    disk        1073MB VBOX HARDDISK
/0/100/d/2        /dev/sdc    disk        1073MB VBOX HARDDISK
/0/100/d/3        /dev/sdd    disk        1073MB VBOX HARDDISK

2.	Задача: Соберите RAID-1
#команда для создания RAID 1 состоящего из 2 дисков
root@clean:/home/asur# mdadm --create --verbose /dev/md0 -l 1 -n 2 /dev/sd{b,c}

#Проверка того, что RAID создался без ошибок
root@clean:/home/asur# cat /proc/mdstat
Personalities : [raid0] [raid1] [raid4] [raid5] [raid6] [raid10] [linear]
md0 : active raid1 sdc[1] sdb[0]
      1046528 blocks super 1.2 [2/2] [UU]

3.	Задача: Сломать и починить RAID
#специально ломаем RAID
mdadm /dev/md0 --fail /dev/sdc

#удаляем сломанный RAID
root@clean:/home/asur# mdadm /dev/md0 --remove /dev/sdc

#добавляем новый диск в RAID
root@clean:/home/asur# mdadm /dev/md0 --add /dev/sdd

#процесс восстановления
root@clean:/home/asur# cat /proc/mdstat
Personalities : [raid0] [raid1] [raid4] [raid5] [raid6] [raid10] [linear]
md0 : active raid1 sdd[2] sdb[0]
      1046528 blocks super 1.2 [2/1] [U_]
      [=============>.......]  recovery = 69.4% (727616/1046528) finish=0.1min speed=31635K/sec


4.	Задача: Создать GPT таблицу, пять разделов и смонтировать их в системе

#создаем раздел GPT
root@clean:/home/asur# parted -s /dev/md0 mklabel gpt

#создаем партиции
root@clean:/home/asur# parted /dev/md0 mkpart primary ext4 0% 20%
root@clean:/home/asur# parted /dev/md0 mkpart primary ext4 20% 40%
root@clean:/home/asur# parted /dev/md0 mkpart primary ext4 40%6%
root@clean:/home/asur# parted /dev/md0 mkpart primary ext4 60% 80%
root@clean:/home/asur# parted /dev/md0 mkpart primary ext4 80% 100%

#необходимо создать файловую систему на ранее созданных партициях (система просит обновить /etc/fstab)
root@clean:/home/asur# for i in $(seq 1 5); do sudo mkfs.ext4 /dev/md0p$i; done
mke2fs 1.47.0 (5-Feb-2023)
Creating filesystem with 51968 4k blocks and 51968 inodes
Filesystem UUID: 1b81450a-f589-4e80-900d-e4bd94975aee
Superblock backups stored on blocks:
        32768……..

#монтирование партиций по каталогам
root@clean:/home/asur# mkdir -p /raid/part{1,2,3,4,5}

root@clean:/home/asur# for i in $(seq 1 5); do mount /dev/md0p$i /raid/part$i; done

root@clean:/home/asur# lsblk
NAME      MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINTS
sda         8:0    0   40G  0 disk
├─sda1      8:1    0    1M  0 part
└─sda2      8:2    0   40G  0 part  /
sdb         8:16   0    1G  0 disk
└─md0       9:0    0 1022M  0 raid1
  ├─md0p1 259:0    0  203M  0 part  /raid/part1
  ├─md0p2 259:1    0  205M  0 part  /raid/part2
  ├─md0p3 259:2    0  204M  0 part  /raid/part3
  ├─md0p4 259:3    0  205M  0 part  /raid/part4
  └─md0p5 259:4    0  203M  0 part  /raid/part5
sdc         8:32   0    1G  0 disk
sdd         8:48   0    1G  0 disk
└─md0       9:0    0 1022M  0 raid1
  ├─md0p1 259:0    0  203M  0 part  /raid/part1
  ├─md0p2 259:1    0  205M  0 part  /raid/part2
  ├─md0p3 259:2    0  204M  0 part  /raid/part3
  ├─md0p4 259:3    0  205M  0 part  /raid/part4
  └─md0p5 259:4    0  203M  0 part  /raid/part5
sr0        11:0    1 1024M  1 rom

