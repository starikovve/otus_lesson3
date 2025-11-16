# otus_lesson3
Administrator Linux. Professional
Домашнее задание
Работа с LVM

Цель:
создавать и управлять логическими томами в LVM;
Описание/Пошаговая инструкция выполнения домашнего задания:
🎯 Задание
На виртуальной машине с Ubuntu 24.04 и LVM.

Уменьшить том под / до 8G.

Выделить том под /home.

Выделить том под /var - сделать в mirror.

/home - сделать том для снапшотов.

Прописать монтирование в fstab. Попробовать с разными опциями и разными файловыми системами (на выбор).

Работа со снапшотами:

сгенерить файлы в /home/;

снять снапшот;

удалить часть файлов;

восстановиться со снапшота.

⭐️Задание со звездочкой 

На дисках поставить btrfs/zfs — с кэшем, снапшотами и разметить там каталог /opt.

Логировать работу можно с помощью утилиты script.


1 . Уменьшить том под / до 8G:

Попытка сделать без использования lveCD:

План: создадим временный корневой раздел на другом диске, перенесём туда систему, загрузимся с него, изменим исходный корневой раздел, перенесём систему обратно и снова перезагрузимся.

подключаю диск к виртуальной машине 16gb можно меньше, но лучше перестраховаться

 <img width="927" height="523" alt="image" src="https://github.com/user-attachments/assets/db2c6669-f89b-4c9a-a588-523910dca876" />
 
Используем диск /dev/sdi для создания временной Volume Group и Logical Volume.
Инициализируем диск 

pvcreate /dev/sdi

Создаем группу томов 

vgcreate vg_root /dev/sdi

Создаем логический том

lvcreate -n lv_root -l +100%FREE vg_root

<img width="895" height="584" alt="image" src="https://github.com/user-attachments/assets/6cd97e28-f8a3-4c20-b1bf-bff934c59d9b" />

Перенос системы на временный том:

Форматируем новый LV в файловую систему ext4:

mkfs.ext4 /dev/vg_root/lv_root

 <img width="974" height="356" alt="image" src="https://github.com/user-attachments/assets/3ca98cf8-8f46-4b4c-b2d8-9530a6e1408e" />
 
Монтируем новый LV в каталог /mnt_root


mkdir /mnt_root
mount /dev/vg_root/lv_root /mnt_root/

<img width="891" height="89" alt="image" src="https://github.com/user-attachments/assets/860476c8-405d-424d-b407-521e982e9a36" />

Копируем все данные с текущего корня в /mnt_root с сохранением всех атрибутов

rsync -avxHAX --progress / /mnt_root/ 

 <img width="974" height="312" alt="image" src="https://github.com/user-attachments/assets/757be80b-7e93-485d-9e29-9d86075a8892" />

Конфигурация GRUB для загрузки с временного тома:
Чтобы система могла загрузиться с нового места, нужно обновить конфигурацию загрузчика GRUB, находясь в окружении новой системы с помощью chroot.
Биндим системные директории в /mnt_root, чтобы chroot-окружение работало корректно

for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt_root/$i; done

Переходим в chroot-окружение в /mnt_root
chroot /mnt_root/
Обновляем конфигурацию загрузчика GRUB, чтобы он нашел новый корневой раздел
grub-mkconfig -o /boot/grub/grub.cfg
Обновляем образ initramfs
Update-initramfs -u

 <img width="974" height="344" alt="image" src="https://github.com/user-attachments/assets/b8484f7d-d239-4ae3-af92-fc257629d464" />

Выходим из chroot и расчитываем что система загрузится с директории /mnt_root)

<img width="974" height="261" alt="image" src="https://github.com/user-attachments/assets/ce04d81e-178f-4ecf-8360-0d8e63dcd287" />

 
Видим что корневым разделом у нас стал наш lv_root
Теперь мы можем работать с размером старого корневого раздела, мы можем безопасно его удалить и пересоздать с нужным размером 8GB.

Удаляем старый логический том

lvremove /dev/ubuntu-vg/ubuntu-lv

<img width="974" height="331" alt="image" src="https://github.com/user-attachments/assets/4607a28f-0ea1-49a7-bd53-b0d388abb754" />
 
Создаем новый логический том ubuntu-lv размером 8G в группе томов ubuntu-vg
lvcreate -n ubuntu-lv -L 8G /dev/ubuntu-vg
Форматируем новый 8-гигабайтный том
mkfs.ext4 /dev/ubuntu-vg/ubuntu-lv

<img width="974" height="366" alt="image" src="https://github.com/user-attachments/assets/158c66b1-a9b6-4dc6-9103-dc231d77bdc5" />
 
Монтируем его в /mnt

mount /dev/ubuntu-vg/ubuntu-lv /mnt

Копируем систему обратно с временного тома (/dev/sdi) на наш новый 8G том

rsync -avxHAX --progress / /mnt/

Повторяем процедуру с chroot для того, чтобы следующая загрузка произошла уже с нашего постоянного, уменьшенного корневого раздела.

Биндим системные директории и переходим в chroot

for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done

chroot /mnt/

Обновляем конфигурацию GRUB и initramfs

grub-mkconfig -o /boot/grub/grub.cfg

update-initramfs -u
 
<img width="974" height="309" alt="image" src="https://github.com/user-attachments/assets/f4263763-7f43-4dab-889e-f058e1317847" />


На этом этапе не перезагружаемся, чтобы сразу выполнить следующую задачу из chroot-окружения ( Выделить том под /var в mirror (RAID-1) )

Проверяем сколь места занимает каталог /var

du /var -hs

<img width="447" height="92" alt="image" src="https://github.com/user-attachments/assets/9ed1c85b-86b2-4a91-bc87-004ea098a876" />
 
Мы все еще находимся в chroot-окружении из предыдущего шага.

Для этого понадобятся диски /dev/sdc и /dev/sdd

<img width="895" height="573" alt="image" src="https://github.com/user-attachments/assets/b104883a-1f51-48ac-a61d-4e37bdd7e273" />
 

Инициализируем диски:

pvcreate /dev/sdc /dev/sdd

Создаем группу томов vg_var на этих двух дисках

vgcreate vg_var /dev/sdc /dev/sdd
 
<img width="734" height="184" alt="image" src="https://github.com/user-attachments/assets/d569d2c6-de25-47eb-898b-d84179aa01ed" />


Создаем зеркальный (-m1) логический том lv_var размером 900M
Ключ -m1 означает "1 mirror copy", т.е. RAID-1

lvcreate -L 900M -m1 -n lv_var vg_var

Форматируем новый зеркальный том

mkfs.ext4 /dev/vg_var/lv_var

<img width="972" height="444" alt="image" src="https://github.com/user-attachments/assets/bc8a9180-24ba-4d0d-a4b2-77497e2d740e" />
 

Монтируем новый том во временную точку /mnt (внутри chroot /mnt это /mnt/mnt)
Чтобы не путаться, выйдем из chroot и выполним снаружи

umount /mnt/boot /mnt/run /mnt/dev /mnt/sys /mnt/proc

Переносим данные из /var
монтируем новый том для var

mount /dev/vg_var/lv_var /mnt/mnt

копируем данные

cp -aR /mnt/var/* /mnt/mnt/

очищаем старый /var

rm -rf /mnt/var/*

отмонтируем

umount /mnt/mnt

<img width="974" height="217" alt="image" src="https://github.com/user-attachments/assets/a149f0e3-4379-4ae4-a277-1af56d62ca46" />

 
Добавляем запись в fstab для автоматического монтирования при загрузke

`blkid` находит UUID тома, что надежнее, чем /dev/mapper/vg_var-lv_var

VAR_UUID=$(blkid -s UUID -o value /dev/vg_var/lv_var) echo "UUID=$VAR_UUID /var ext4 defaults 0 2" >> /mnt/etc/fstab
 
<img width="974" height="398" alt="image" src="https://github.com/user-attachments/assets/e144019b-48a8-4ab2-bd07-ea6715152e92" />

Перезагружаем систему
После перезагрузки мы видим в  основной системе с уменьшенным до 8GB корнем и /var на отдельном зеркальном томе. Теперь можно удалить временные ресурсы.

 <img width="958" height="858" alt="image" src="https://github.com/user-attachments/assets/8e064f26-e2c6-4826-ab7f-e5e926f155ce" />
 
<img width="974" height="260" alt="image" src="https://github.com/user-attachments/assets/18723b56-3a0d-41fa-b958-f805c335cf35" />

 
Удаляем временный LV, VG и PV

lvremove /dev/vg_root/lv_root 

vgremove /dev/vg_root

pvremove /dev/sdi

<img width="974" height="335" alt="image" src="https://github.com/user-attachments/assets/83cae4e8-8d00-4f2a-ba3e-4b7eb367f9d2" />
 
Таким образом мы выполнили два шага:
Уменьшили корневой раздел до 8Гиг и сделали зеркалирование раздела /var

Выделить том под /home:
Создаем LV LogVol_Home размером 2G в группе ubuntu-vg

lvcreate -n LogVol_Home -L 2G /dev/ubuntu-vg

Форматируем его в файловую систему XFS

mkfs.xfs /dev/ubuntu-vg/LogVol_Home

<img width="974" height="353" alt="image" src="https://github.com/user-attachments/assets/ac7f825f-f164-4b77-b1af-9560ca3c56de" />
 
Монтируем новый том для переноса данных

mount /dev/ubuntu-vg/LogVol_Home /mnt/

Копируем существующие данные из /home и очищаем старый каталог

cp -aR /home/* /mnt/ 
rm -rf /home/*

<img width="936" height="122" alt="image" src="https://github.com/user-attachments/assets/007cfcfa-f8c3-41b2-bc61-91075097f8bf" />
 

Отмонтируем /mnt и монтируем новый том в /home

umount /mnt

 mount /dev/ubuntu-vg/LogVol_Home /home/
 
 <img width="974" height="325" alt="image" src="https://github.com/user-attachments/assets/94e8b2a1-78bc-40cc-a455-f49c3e59e691" />

Добавляем запись в /etc/fstab для автоматического монтирования
HOME_UUID=$(blkid -s UUID -o value /dev/ubuntu-vg/LogVol_Home) echo "UUID=$HOME_UUID /home xfs defaults 0 2" >> /etc/fstab

Работа со снапшотами:

Создаем 24 пустых файлов в /home

touch /home/file{1..24} ls /home

<img width="974" height="125" alt="image" src="https://github.com/user-attachments/assets/43823bbe-685b-4ab2-a341-9def30d61cae" />
 
Создание снапшота:
Создаем снапшот тома /dev/ubuntu-vg/LogVol_Home. Для снапшота нужно выделить место для хранения изменений (Copy-on-Write).
Создаем снапшот (-s) с именем home_snap и размером 100MB

lvcreate -L 100M -s -n home_snap /dev/ubuntu-vg/LogVol_Home

<img width="974" height="71" alt="image" src="https://github.com/user-attachments/assets/01046e06-c893-4bb5-a56a-7d9c9a640fd6" />
 
Имитация потери данных:
Удалим часть созданных файлов.
Удаляем файлы с 11 по 20

rm -f /home/file{11..20}

 <img width="974" height="72" alt="image" src="https://github.com/user-attachments/assets/97bca6f7-e152-4c05-bdf5-5405ce418ee0" />

Восстановление из снапшота:
Для восстановления (слияния снапшота с основным томом) том /home должен быть отмонтирован.
Отмонтируем /home
umount /home

lvconvert --merge /dev/ubuntu-vg/home_snap

<img width="974" height="522" alt="image" src="https://github.com/user-attachments/assets/847aafe5-078c-44dd-9df4-5bc7602434b8" />
 
Монтируем /home обратно
mount /home
Проверяем результат
ls  /home

 <img width="974" height="90" alt="image" src="https://github.com/user-attachments/assets/491c6770-4377-46c5-8cb2-fd8c1195d8af" />


