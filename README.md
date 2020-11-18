lvm
#скачиваем Vagrantfile

   https://gitlab.com/otus_linux/stands-03-lvm

#захожу в папку с Vagrantfile,

   vagrant up

   vagrant ssh

#затем работаю под sudo

sudo -i

#устанавливаю пакет xfsdump

yum install xfsdump

#готовлю временый том для раздела

pvcreate /dev/sdb

vgcreate vg_root /dev/sdb

lvcreate -n lv_root -l +100%FREE /dev/vg_root

#создаю на нем файловую систему

mkfs.xfs /dev/vg_root/lv_root

#и смонтирую его, чтобы перенести туда данные

mount /dev/vg_root/lv_root /mnt

#копирую все данные с раздела в /mnt

xfsdump -J - /dev/VolGroup00/LogVol00 | xfsrestore -J - /mnt

#проверяю скопировались данные

ls /mnt/

#после переконфигурирую grub для того, чтобы при старте перейти в новый
#Сымитируем текущий root и сделаем в него chroot, обновим grub

for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done

chroot /mnt/

grub2-mkconfig -o /boot/grub2/grub.cfg

#Обновим образ initrd.

cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g;
s/.img//g"` --force; done

#для того, чтобы при загрузке был смонтирован нужный root нужно в файле
#/boot/grub2/grub.cfg заменить rd.lvm.lv=VolGroup00/LogVol00 на rd.lvm.lv=vg_root/lv_root

#я скачал nano

yum install nano

nano /boot/grub2/grub.cfg

#меняю rd.lvm.lv=VolGroup00/LogVol00 на rd.lvm.lv=vg_root/lv_root и сохраняю
#перезагружаюсь новым рутом

reboot

#проверяем что система загрузилась с новым root томом

lsblk

#Теперь нужно изменить размер старой VG и вернуть на него root. Для этого удаляем
#старый LV размеров в 40G и создаем новый на 8G

lvremove /dev/VolGroup00/LogVol00 [y]

lvcreate -n VolGroup00/LogVol00 -L 8G /dev/VolGroup00 [y]

#Проделываем на нем те же операции, что и в первый раз

mkfs.xfs /dev/VolGroup00/LogVol00

mount /dev/VolGroup00/LogVol00 /mnt

xfsdump -J - /dev/vg_root/lv_root | xfsrestore -J - /mnt

#Так же как в первый раз переконфигурирую grub, за исключением правки /etc/grub2/grub.cfg

for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done

chroot /mnt/

grub2-mkconfig -o /boot/grub2/grub.cfg

cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g;
s/.img//g"` --force; done

#Пока не перезагружаюсь и не выхожу из под chroot - потому как можно заодно перенести /var
#На свободных дисках создаю зеркало

pvcreate /dev/sdc /dev/sdd

vgcreate vg_var /dev/sdc /dev/sdd

lvcreate -L 950M -m1 -n lv_var vg_var

#Создю на нем ФС и перемещаю туда /var

mkfs.ext4 /dev/vg_var/lv_var

mount /dev/vg_var/lv_var /mnt

cp -aR /var/* /mnt/

rsync -avHPSAX /var/ /mnt/

#На всякий случай сохраняю содержимое старого var (или же можно его просто удалить)

mkdir /tmp/oldvar && mv /var/* /tmp/oldvar

#Ну и монтирую новый var в каталог /var

umount /mnt

mount /dev/vg_var/lv_var /var

#Правлю fstab для автоматического монтирования /var

echo "`blkid | grep var: | awk '{print $2}'` /var ext4 defaults 0 0" >> /etc/fstab

#После чего можно успешно перезагружаться в новый (уменьшенный root) и удалять
#временную Volume Group

lvremove /dev/vg_root/lv_root

vgremove /dev/vg_root

pvremove /dev/sdb

#Выделяю том под /home по тому же принципу что делал для /var

lvcreate -n LogVol_Home -L 2G /dev/VolGroup00

mkfs.xfs /dev/VolGroup00/LogVol_Home

mount /dev/VolGroup00/LogVol_Home /mnt/

cp -aR /home/* /mnt/

rm -rf /home/*

umount /mnt

mount /dev/VolGroup00/LogVol_Home /home/

#Правлю fstab для автоматического монтирования /home

echo "`blkid | grep Home | awk '{print $2}'` /home xfs defaults 0 0" >> /etc/fstab

/home - сделать том для снапшотов

#Сгенерирую файлы в /home/

touch /home/file{1..20}

#Снять снапшот

lvcreate -L 100MB -s -n home_snap /dev/VolGroup00/LogVol_Home

#Удалить часть файлов

rm -f /home/file{11..20}

#Процесс восстановления со снапшота

umount /home

lvconvert --merge /dev/VolGroup00/home_snap

mount /home

#проверяю восстановились ли файлы

ls /home/


