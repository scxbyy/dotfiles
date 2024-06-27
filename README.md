## Installation

	iwctl
	station wlan0 connect SSID
	<password prompt>
	exit
	/dev/nvme0n1p1 - EFI - 512MB;				partition code EF00
	/dev/nvme0n1p2 - encrypted LUKS - remaining space;	partition code 8309
	mkfs.fat -F32 /dev/nvme0n1p1
	cryptsetup luksFormat --type luks2 /dev/nvme0n1p2
	pvcreate /dev/mapper/cryptlvm
	vgcreate vg /dev/mapper/cryptlvm
	lvcreate -l 100%FREE vg -n root
	mkfs.ext4 /dev/vg/root
	mount /dev/vg/root /mnt
	mkdir -p /mnt/boot/efi
	mount /dev/nvme0n1p1 /mnt/boot/efi
	pacstrap /mnt base linux linux-firmware intel-ucode sudo vim lvm2 dracut sbsigntools iwd git efibootmgr binutils dhcpcd
	genfstab -U /mnt >> /mnt/etc/fstab
	arch-chroot /mnt
	passwd
	pacman -Syu man-db
	ln -sf /usr/share/zoneinfo/America/New_York /etc/localtime
	hwclock --systohc
	vim /etc/locale.gen # uncomment locales you want
	locale-gen

	vim /etc/locale.conf
		LANG=en_US.UTF-8
  
	vim /etc/vconsole.conf
		KEYMAP=us
		FONT=Lat2-Terminus16

	vim /etc/hostname
 
	useradd -m aura
	passwd aura
 
	visudo
		%wheel	ALL=(ALL) ALL # Uncomment this line

	usermod -aG wheel,audio,power,video,storage,floppy aura
 
 	systemctl enable dhcpcd
    systemctl enable iwd

## Creating Unified Kernel Image and configuring boot entry

Create dracut scripts that will hook into pacman:

	vim /usr/local/bin/dracut-install.sh

		#!/usr/bin/env bash

		mkdir -p /boot/efi/EFI/Linux
  
		while read -r line; do
			if [[ "$line" == 'usr/lib/modules/'+([^/])'/pkgbase' ]]; then
				kver="${line#'usr/lib/modules/'}"
				kver="${kver%'/pkgbase'}"
		
				dracut --force --uefi --kver "$kver" /boot/efi/EFI/Linux/arch-linux.efi
			fi
		done

And the removal script:

	vim /usr/local/bin/dracut-remove.sh

		#!/usr/bin/env bash
	 	rm -f /boot/efi/EFI/Linux/arch-linux.efi

Make those scripts executable and create pacman's hook directory:

	chmod +x /usr/local/bin/dracut-*
 	mkdir /etc/pacman.d/hooks

 Now the actual hooks, first for the install and upgrade:

	 vim /etc/pacman.d/hooks/90-dracut-install.hook
  
		[Trigger]
		Type = Path
		Operation = Install
		Operation = Upgrade
		Target = usr/lib/modules/*/pkgbase
		
		[Action]
		Description = Updating linux EFI image
		When = PostTransaction
		Exec = /usr/local/bin/dracut-install.sh
		Depends = dracut
		NeedsTargets

And for removal:

	vim /etc/pacman.d/hooks/60-dracut-remove.hook
 
		[Trigger]
		Type = Path
		Operation = Remove
		Target = usr/lib/modules/*/pkgbase
		
		[Action]
		Description = Removing linux EFI image
		When = PreTransaction
		Exec = /usr/local/bin/dracut-remove.sh
		NeedsTargets

Check UUID of your encrypted volume and write it to file you will edit next:

	blkid -s UUID -o value /dev/nvme0n1p2 >> /etc/dracut.conf.d/cmdline.conf

Edit the file and fill with with kernel arguments:

	vim /etc/dracut.conf.d/cmdline.conf
		kernel_cmdline="rd.luks.uuid=luks-YOUR_UUID rd.lvm.lv=vg/root root=/dev/mapper/vg-root rootfstype=ext4 rootflags=rw,relatime"

Create file with flags:

	vim /etc/dracut.conf.d/flags.conf
		compress="zstd"
		hostonly="no"

Generate your image by re-installing `linux` package and making sure the hooks work properly:

	pacman -S linux

 You should have `arch-linux.efi` within your `/efi/EFI/Linux/`

 Now you only have to add UEFI boot entry and create an order of booting:

	efibootmgr --create --disk /dev/nvme0n1 --part 1 --label "Arch Linux" --loader 'EFI\Linux\arch-linux.efi' --unicode
	efibootmgr 		# Check if you have left over UEFI entries, remove them with efibootmgr -b INDEX -B and note down Arch index
	efibootmgr -o ARCH_INDEX_FROM_PREVIOUS_COMMAND # 0 or whatever number your Arch entry shows as

Now you can reboot and log into your system.

:exclamation: :exclamation: :exclamation: **Compatilibity thing I noticed** :exclamation: :exclamation: :exclamation:

Some (older?) platforms can ignore entries by efibootmgr all together and just look for `EFI\BOOT\bootx64.efi`, in that case you may generate your UKI directly to that directory and under that name. It's very important that the name is also `bootx64.efi`.

## SecureBoot

At this point you should enable Setup Mode for SecureBoot in your BIOS, and erase your existing keys (it may spare you setting attributes for efi vars in OS). If your system does not offer reverting to default keys (useful if you want to install windows later), you should backup them, though this will not be described here.

Configuring SecureBoot is easy with sbctl:

	pacman -S sbctl

Check your status, setup mode should be enabled (You can do that in BIOS):

	sbctl status
	  Installed:      ✘ Sbctl is not installed
	  Setup Mode:     ✘ Enabled
	  Secure Boot:    ✘ Disabled

Create keys and sign binaries:

	sbctl create-keys
	sbctl sign -s /boot/efi/EFI/Linux/arch-linux.efi #it should be single file with name verying from kernel version

Configure dracut to know where are signing keys:

	vim /etc/dracut.conf.d/secureboot.conf
		uefi_secureboot_cert="/usr/share/secureboot/keys/db/db.pem"
		uefi_secureboot_key="/usr/share/secureboot/keys/db/db.key"

We also need to fix sbctl's pacman hook. Creating the following file will overshadow the real one:

	vim /etc/pacman.d/hooks/zz-sbctl.hook
		[Trigger]
		Type = Path
		Operation = Install
		Operation = Upgrade
		Operation = Remove
		Target = boot/*
		Target = efi/*
		Target = usr/lib/modules/*/vmlinuz
		Target = usr/lib/initcpio/*
		Target = usr/lib/**/efi/*.efi*

		[Action]
		Description = Signing EFI binaries...
		When = PostTransaction
		Exec = /usr/bin/sbctl sign /boot/efi/EFI/Linux/arch-linux.efi

Enroll previously generated keys (drop microsoft option if you don't want their keys):

	sbctl enroll-keys --microsoft

Reboot the system. Enable only UEFI boot in BIOS and set BIOS password so evil maid won't simply turn off the setting. If everything went fine you should first of all, boot into your system, and then verify with sbctl or bootctl:

	sbctl status
	  Installed:	✓ sbctl is installed
	  Owner GUID:	YOUR_GUID
	  Setup Mode:	✓ Disabled
	  Secure Boot:	✓ Enabled
