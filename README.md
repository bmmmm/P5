# P5
Arch Linux encrypted 2 HDD installation

scenario:
	sda = root 100G & swap 8G  & (left space) data
	sda1 = UEFI boot
	sda2 = luks lvm
		- main-root
		- main-swap
		- main-data
	sdb = home (whole disk)
	sdb1 = luks lvm
		- main-home 

understanding lvm diagram: https://askubuntu.com/questions/219881/how-can-i-create-one-logical-volume-over-two-disks-using-lvm

in praxis we used to connect to installation pc via ssh. for this just do:

check ip with:

	ip addr OR ifconfig

	passwd
	systemctl start sshd.service
	
now change to remote pc:

	ssh root@IPADDR

1. prepare partition table as mentioned for sda:

		gdisk /dev/sda
	
2. prepare partition table as mentioned for sdb:

		gdisk /dev/sdb

3. make file systems for both disks:

		mkfs.fat -F 32 -n EFIBOOT /dev/sda1
		cryptsetup -c aes-xts-plain64 -y -s 512 luksFormat /dev/sda2
		cryptsetup -c aes-xts-plain64 -y -s 512 luksFormat /dev/sdb1
	
4. create LVM on both disks:
	
		cryptsetup luksOpen /dev/sda2 lvm
		pvcreate /dev/mapper/lvm
		vgcreate main /dev/mapper/lvm
		lvcreate -L 100GB -n root main
		lvcreate -L 8GB -n swap main
		lvcreate -l 100%FREE -n data main

		cryptsetup luksOpen /dev/sdb1 lvmB
		pvcreate /dev/mapper/lvmB
		vgextend main /dev/mapper/lvmB 
		lvcreate -l 100%FREE -n home main

make filesystems for all partitions:

	mkfs.ext4 -L root -O \^64bit /dev/mapper/main-root
	mkfs.ext4 -L data -O \^64bit /dev/mapper/main-data
	mkswap -L swap /dev/mapper/main-swap

	mkfs.ext4 -L home -O \^64bit /dev/mapper/main-home

5. mount partitions in folders:
	
		mount /dev/mapper/main-root /mnt
		mkdir /mnt/home
		mount /dev/mapper/main-home /mnt/home
		mkdir /mnt/boot
		mount /dev/sda1 /mnt/boot
		mkdir /mnt/data
		mount /dev/mapper/main-data /mnt/data
		swapon /dev/mapper/main-swap

6. install base system

	prepare mirror list:
	
		cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.bak
		grep -E -A 1 ".*Germany.*$" /etc/pacman.d/mirrorlist.bak | sed '/--/d' > /etc/pacman.d/mirrorlist
	
	base packets:
	
		pacstrap /mnt base base-devel intel-ucode wpa_supplicant grub efibootmgr dosfstools gptfdisk
	
	create fstab with UUID and labels (ULp)
	
		genfstab -ULp /mnt > /mnt/etc/fstab  

7. modify fstab for SSD drive (not home drive because of no SSD drive)
	
		LABEL=root              /               ext4            rw,defaults,noatime,discard     0 1
		LABEL=data              /data           ext4            rw,defaults,noatime,discard     0 2
		LABEL=swap              none            swap            defaults,noatime,discard        0 0

8. change root

		arch-chroot /mnt

9. configure system
	
		echo ArchComputer > /etc/hostname

		nano /etc/locale.conf

	write following lines into **/etc/locale.conf**:
	
		echo LANG=de_DE.UTF-8 > /etc/locale.conf
		echo LC_COLLATE=C >> /etc/locale.conf
		echo LANGUAGE=de_DE >> /etc/locale.conf
	
	write following lines into **/etc/vconsole.conf**:
	
		echo KEYMAP=de-latin1 > /etc/vconsole.conf
		echo FONT=lat9w-16 >> /etc/vconsole.conf

	link to local time zone:
	
		ln -sf /usr/share/zoneinfo/Europe/Berlin /etc/localtime

	uncomment following lines in **/etc/locale.gen**:	
	
		nano /etc/locale.gen

		#de_DE.UTF-8 UTF-8
		#de_DE ISO-8859-1
		#de_DE@euro ISO-8859-15

	generate locals with:
	
		locale-gen

10. configure GRUB

		grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=arch_grub --recheck --debug
		mkdir -p /boot/grub/locale
		cp /usr/share/locale/en\@quot/LC_MESSAGES/grub.mo /boot/grub/locale/en.mo

	check UUID from encrypted partition sda2:
	
		blkid

		open **/etc/default/grub** and comment existing **GRUB_CMDLINE_LINUX=""** and replace with:
		nano /etc/default/grub
		GRUB_CMDLINE_LINUX="lang=de locale=de_DE.UTF-8 cryptdevice=UUID="de2c8075-fa4d-4e08-821e-bf16051a5623":main root=/dev/mapper/main-root"

	create grub config file (don't care about warnings)
	
		grub-mkconfig -o /boot/grub/grub.cfg

11. edit /etc/mkinitcpio.conf:

		nano /etc/mkinitcpio.conf
		MODULES="ext4"
		comment existing HOOKS and paste following line:
		HOOKS="base udev autodetect modconf block keyboard keymap encrypt lvm2 filesystems fsck shutdown"


12. prepare **/etc/crypttab** for extern home sdb1:
	
		make folder to store keyfile:
		mkdir root/crypto
	
	create keyfile:
	
		dd if=/dev/urandom of=/root/crypto/home.key bs=1k count=2 

	add key do channel for open:
	
		cryptsetup luksAddKey /dev/sdb1 /root/crypto/home.key

	nano /etc/crypttab : 
	
		home	UUID=67dc0b7c-f72b-404d-a177-b9e539f85b43	/root/crypto/home.key 

13. create kernel-image

		mkinitcpio -p linux

14. enable network DHCP:
	
		systemctl enable dhcpcd.service
	

15. leave chroot, umount and reboot
	
		passwd
		exit
		umount -R /mnt
		reboot
	
