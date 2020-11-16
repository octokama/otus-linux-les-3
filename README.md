# otus-linux-les-3
## Работа с LVM
Для поднятия машины из Vagrantfile `vagrant up` и подключение `vagrant ssh`

### 1. Уменьшить том под / до 8G 
подготовка временного тома для /
`pvcreate /dev/sdb` 
`vgcreate vg_root /dev/sdb` 
`lvcreate -n lv_root -l +100%FREE /dev/vg_root` 
 
создние ФС
`mkfs.xfs /dev/vg_root/lv_root` 
`mount /dev/vg_root/lv_root /mnt` 
 
Копирование данных из / в /mnt
`xfsdump -J - /dev/VolGroup00/LogVol00 | xfsrestore -J - /mnt` 
 
Имитация текущего root и обновление grub:
`for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done` 
`chroot /mnt/` 
`grub2-mkconfig -o /boot/grub2/grub.cfg` 

Обновление образа initrd 
	cd /boot ; for i in \`ls initramfs-\*img\`; do dracut -v $i \`echo $i|sed "s/initramfs-//g; s/.img//g"\` --force; done 
 
Изменение размера старой VG 
`lvremove /dev/VolGroup00/LogVol00` 
`lvcreate -n VolGroup00/LogVol00 -L 8G /dev/VolGroup00` 

Создание ФС и копирование данных 
`mkfs.xfs /dev/VolGroup00/LogVol00` 
`mount /dev/VolGroup00/LogVol00 /mnt` 
`xfsdump -J - /dev/vg_root/lv_root | xfsrestore -J - /mnt` 

Переконфигурация grub 
`for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done` 
`chroot /mnt/` 
`grub2-mkconfig -o /boot/grub2/grub.cfg` 
	cd /boot ; for i in \`ls initramfs-\*img\`; do dracut -v $i \`echo $i|sed "s/initramfs-//g; s/.img//g"\` --force; done 

### Выделить том под /var
Создание зеркала
	pvcreate /dev/sdc /dev/sdd 
	vgcreate vg_var /dev/sdc /dev/sdd 
	lvcreate -L 950M -m1 -n lv_var vg_var 
 
Создае ФС и перемещаем /var
	mkfs.ext4 /dev/vg_var/lv_var 
	mount /dev/vg_var/lv_var /mnt 
	cp -aR /var/* /mnt/ # rsync -avHPSAX /var/ /mnt/ 
	mkdir /tmp/oldvar && mv /var/* /tmp/oldvar 
 
Монтирование нового var в каталог /var
	umount /mnt 
	mount /dev/vg_var/lv_var /var 
 
Правка fstab для автоматического монтирования /var
	echo "\`blkid | grep var: | awk '{print $2}'\` /var ext4 defaults 0 0" >> /etc/fstab 
 
Удаление временного Volume Group 
	lvremove /dev/vg_root/lv_root 
	vgremove /dev/vg_root 
 	pvremove /dev/sdb 
 
### Выделить том под /home 
Создание LV 
	lvcreate -n LogVol_Home -L 2G /dev/VolGroup00 
 
Создае ФС и копирование /home
	mkfs.xfs /dev/VolGroup00/LogVol_Home 
	mount /dev/VolGroup00/LogVol_Home /mnt/ 
	cp -aR /home/* /mnt/ 
	rm -rf /home/* 
 
Монтирование нового home в каталог /home
	umount /mnt 
	mount /dev/VolGroup00/LogVol_Home /home/ 
 
Правка fstab для автоматического монтирования /home 
	echo "\`blkid | grep Home | awk '{print $2}'\` /home xfs defaults 0 0" >> /etc/fstab 
 
Генерация файлов в /home/
	touch /home/file{1..20} 

Снепшот 
	lvcreate -L 100MB -s -n home_snap /dev/VolGroup00/LogVol_Home
 
Удаление части файлов 
	rm -f /home/file{11..20} 
 
Восстановление со снепшота: 
	umount /home 
	lvconvert --merge /dev/VolGroup00/home_snap 
	mount /home 
 
 