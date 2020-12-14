В ДЗ не предоставлен vagrantfile. Создаем свой. Centos 7. +2 SATA диска.

Проверяем наличие дисков и заводим ZFS
```sh
sudo -s
lsblk
```
Видим диски
```sh
NAME MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
sda 8:0 0 40G 0 disk
├─sda1 8:1 0 1M 0 part
├─sda2 8:2 0 1G 0 part /boot
└─sda3 8:3 0 39G 0 part
├─VolGroup00-LogVol00 253:0 0 37.5G 0 lvm /
└─VolGroup00-LogVol01 253:1 0 1.5G 0 lvm [SWAP]
sdb 8:16 0 1G 0 disk
sdc 8:32 0 1G 0 disk
```
Заводим ZFS 
```sh
yum install yum-utils -y
yum install -y http://download.zfsonlinux.org/epel/zfs-release.el7_9.noarch.rpm
yum-config-manager --disable zfs
yum-config-manager --enable zfs-kmod
yum install zfs -y
modprobe zfs
```
Определить алгоритм с наилучшим сжатием создать 4 файловых системы на каждой применить свой алгоритм сжатия 
1.1 Создаем pool
```sh
zpool create pool0 mirror /dev/sdb /dev/sdc
```
1.2 Создаем ФС со сжатием gzip, zle lzjb, lz4
```sh
zfs create -o compression=gzip pool0/gzip
zfs create -o compression=lz4 pool0/lz4
zfs create -o compression=lzjb pool0/lzjb
zfs create -o compression=zle pool0/zle
```
скачать файл ядра распаковать и расположить на файловой системе 

ставим wget, качаем kernel и смотрим исходное состояние
```sh
yum install wget -y
wget https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.9.13.tar.xz
zfs list
```

Видим
```sh
NAME USED AVAIL REFER MOUNTPOINT
pool0 217K 3.62G 28K /pool0
pool0/gzip 24K 3.62G 24K /pool0/gzip
pool0/lz4 24K 3.62G 24K /pool0/lz4
pool0/lzjb 24K 3.62G 24K /pool0/lzjb
pool0/zle 24K 3.62G 24K /pool0/zle
```
Распаковываем ядро в каждую из ФС
```sh
tar -C /pool0/gzip -xvf linux-5.9.13.tar.xz
tar -C /pool0/lz4 -xvf linux-5.9.13.tar.xz
tar -C /pool0/lzjb -xvf linux-5.9.13.tar.xz
tar -C /pool0/zle -xvf linux-5.9.13.tar.xz
```
Смотрим, что получилось
```sh
zfs list
NAME USED AVAIL REFER MOUNTPOINT
pool0 1.98G 1.64G 28K /pool0
pool0/gzip 253M 1.64G 253M /pool0/gzip
pool0/lz4 380M 1.64G 380M /pool0/lz4
pool0/lzjb 436M 1.64G 436M /pool0/lzjb
pool0/zle 962M 1.64G 962M /pool0/zle
```
Алгоритм с наилучшим сжатием - gzip

Определить настройки pool’a
Загрузить архив с файлами локально.
```sh
wget --no-check-certificate -O zfs_task1.tar 'https://drive.google.com/u/0/uc?id=1KRBNW33QWqbvbVHa3hLJivOAt60yukkg&export=download'
```
2.2 Распаковать
```sh
tar xf zfs_task1.tar.gz
```
2.3 С помощью команды zfs import собрать pool ZFS.
```sh
zpool import -d /home/vagrant/zpoolexport/
pool: otus
 id: 6554193320433390805
state: ONLINE
action: The pool can be imported using its name or numeric identifier.
config:
    otus                                 ONLINE
          mirror-0                           ONLINE
             /home/vagrant/zpoolexport/filea  ONLINE
             /home/vagrant/zpoolexport/fileb  ONLINE
zpool list
NAME SIZE ALLOC FREE CKPOINT EXPANDSZ FRAG CAP DEDUP HEALTH ALTROOT
pool0 3.75G 1.98G 1.77G - - 0% 52% 1.00x ONLINE -
```
Не сработало. Импортируем как read only.
```sh
zpool import otus -o readonly=on -d /home/vagrant/zpoolexport/
zpool list
NAME SIZE ALLOC FREE CKPOINT EXPANDSZ FRAG CAP DEDUP HEALTH ALTROOT
otus 480M 2.11M 478M - - 0% 0% 1.00x ONLINE -
pool0 3.75G 1.98G 1.77G - - 0% 52% 1.00x ONLINE -
```
Вывод комманд определения настроек пула:
```sh
zpool list
zpool status otus
```
Остальное забираем отсюда
```sh
zfs get all otus
NAME PROPERTY VALUE SOURCE
otus recordsize 128K local
otus checksum sha256 local
otus compression zle local
```
размер хранилища - 480М
тип pool - mirror
значение recordsize - 128K
какое сжатие используется - zle
какая контрольная сумма используется - sha256

Найти сообщение от преподавателей
3.1 Скопировать файл из удаленной директории.
```sh
wget --no-check-certificate -O otus_task2.file 'https://drive.google.com/file/d/1gH8gCL9y7Nd5Ti3IRmplZPF1XjzxeRAG/view?usp=sharing'
```
3.2 Восстановить его локально.
```sh
zfs receive pool0/gzip/task2 < otus_task2.file
```
3.3 Найти зашифрованное сообщение в файле secret_message
```sh
find /pool0/gzip/task2/ -name secret_message
/pool0/gzip/task2/task1/file_mess/secret_message
cat /pool0/gzip/task2/task1/file_mess/secret_message
