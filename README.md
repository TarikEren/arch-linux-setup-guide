# Arch Linux Setup
- The following guide will describe installing Arch Linux in an opinionated way:
    - File system: BTRFS (LUKS encrypted)
    - Boot loader: Limine
    - Window Compositor: Hyprland (No desktop environment)

- The following guide is also for UEFI systems. You can always refer to the [Arch Wiki](https://wiki.archlinux.org) whenever you get stuck, need help or encounter something this guide doesn't cover.
    - The wiki will be your best friend when using Arch Linux anyway. Use it liberally.

## 0. Preparation

```bash
loadkeys trq        # Loads keymap (Turkish-Q in this case)
setfont ter-132b    # Sets a more high resolution font as the console font
```

## 1. Connecting to Internet (Skip if connected via ethernet)

```bash
iwctl station <device> connect <SSID>   # Connect to <SSID> using <device>
pacman -Syy archlinux-keyring           # Update packages and keyring
```
## 2. Setting Up Disk Partitions

- Note:
    - For the sake of this tutorial, `nvme0n1` will be used as the disk name. Use:

    ```bash
    lsblk
    ```
    to see the available disks.

```bash
sgdisk --zap-all /dev/nvme0n1                   # Wipe the disk clean

# Create a 2GB EFI partition and create a btrfs Linux partition using the rest
parted --script /dev/nvme0n1 \
mklabel gpt mkpart ESP fat32 1MiB 2049MiB \
set 1 esp on mkpart Linux btrfs 2050MiB 100%

mkfs.fat -F 32 /dev/nvme0n1p1                   # Format nvme0n1p1 as FAT32
cryptsetup luksFormat /dev/nvme0n1p2            # Create LUKS encrypted container (Will prompt encryption password)
cryptsetup open /dev/nvme0n1p2 root             # Open the encrypted container as 'root'
mkfs.btrfs /dev/mapper/root                     # Format partition as BTRFS
mount /dev/mapper/root /mnt                     # Mount partition
btrfs subvolume create /mnt/@					# Create partitions
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@var_log
btrfs subvolume create /mnt/@var_cache
btrfs subvolume create /mnt/@snapshots
umount /mnt                                     # Temporarily unmount the disk

### Mounting partitions ###
# 'compress=zstd:1' is for saving space by using ZStandard compression. Delete ':1' if you don't want to mention a compression level. Omit for no compression.
# 'noatime' reduces unnecessary wear on SSD by telling mount not to save the last access time on the disk.
mount -o compress=zstd:1,noatime,subvol=@ /dev/mapper/root /mnt
mount --mkdir -o compress=zstd:1,noatime,subvol=@home /dev/mapper/root /mnt/home
mount --mkdir -o compress=zstd:1,noatime,subvol=@var_log /dev/mapper/root /mnt/var/log
mount --mkdir -o compress=zstd:1,noatime,subvol=@var_cache /dev/mapper/root /mnt/var/cache
mount --mkdir -o compress=zstd:1,noatime,subvol=@snapshots /dev/mapper/root /mnt/.snapshots
mount --mkdir /dev/nvme0n1p1 /mnt/boot
```

## 3. Installing Packages
- Note: For a **very** minimal installation:
```bash
pacstrap -K /mnt base linux linux-firmware
```
This won't include any useful packages like bluetooth, drivers etc. The system will work though.

The following is the script that I use for my installations. Feel free to modify any packages except the packages denoted as `Base Packages`

```bash
pacstrap -K /mnt \  # Download to /mnt

# Base Packages (Don't delete any)
base linux linux-firmware base-devel btrfs-progs efibootmgr limine \
networkmanager iwd dhcpcd cryptsetup util-linux \

# Feel free to add or remove packages here on out
git bash-completion avahi acpi acpi_call acpid alsa-utils \
nvim \ # You can swap nvim out with vim, nano or any text editor you want.
pipewire pipewire-alsa pipewire-pulse pipewire-jack wireplumber \
sof-firmware firewalld bluez bluez-utils cups openssh snap-pac \
reflector man sudo rsync intel-ucode nvidia-open udisks2
```

## 4. Configuring The System
```bash
genfstab -U /mnt >> /mnt/etc/fstab          # Generate fstab file (Contains information regarding disks and UUIDs)
arch-chroot /mnt                            # 'Chroot' into system
ln -sf /usr/share/zoneinfo/Europe/Istanbul /etc/localtime           # Set local time (Adjust if necessary)
hwclock --systohc                           # Set hardware clock using the system clock
echo -e "tr_TR.UTF-8 UTF-8\nen_US.UTF-8 UTF-8" > /etc/locale.gen    # Set locale (Adjust if necessary)
echo LANG=en_US.UTF-8 > /etc/locale.conf    # Set system language (Adjust if necessary)
echo KEYMAP=trq > /etc/vconsole.conf        # Set keyboard layout (Adjust if necessary)
echo TarikEren > /etc/hostname              # Set hostname (Adjust)
passwd                                      # Set root password
useradd -mg wheel TarikEren                 # Add the user to the wheel group (Adjust name)
passwd TarikEren                            # Set user password (Adjust name)

EDITOR=nvim visudo                          # Edit visudo file. (Find and uncomment the line '%wheel ALL=(ALL:ALL) ALL')

# Remove the old mkinitcpio.conf and write necessary values
# Feel free to write the values by hand.
cat > /etc/mkinitcpio.conf <<EOF
	MODULES=(btrfs)
	BINARIES=(/usr/bin/btrfs)
	FILES=()
	HOOKS=(base udev autodetect microcode modconf kms keyboard keymap block encrypt filesystems resume fsck)
EOF
mkinitcpio -P   # Generate system image based on the new mkinitcpio.conf file
```

## 5. Setting Up Limine Boot Loader

This section is important as there are a few caveats. Some MSI motherboards tend to cause issues when it comes to booting from a non-Windows source.
As of writing this, my own motherboard (MSI PRO H610M-E DDR4) is one of the problematic motherboards.

You can check your motherboard model by typing:
```bash
cat /sys/devices/virtual/dmi/id/board_name
```

I strongly advise you to research if your motherboard has caused Limine issues etc. as [the standard Limine setup](#512-setup-for-non-compliant-uefi-motherboards) won't work.

If you don't know if yours is UEFI or BIOS, use:
```bash
cat /sys/firmware/efi/fw_platform_size
```

If the output is `64` or `32`, your system uses UEFI.

This tutorial will be focused on UEFI systems. Check out [the arch linux limine tutorial](https://wiki.archlinux.org/title/Limine) for more info.

### 5.1.1 Setup for Compliant UEFI Motherboards
This section will cover most UEFI motherboards. Refer to [step 5.1.2](#512-setup-for-non-compliant-uefi-motherboards) if your motherboard is a 'problematic' one.

```bash
mkdir -p /boot/EFI/limine                           # Create the necessary directories
cp /usr/share/limine/BOOTX64.EFI /boot/EFI/limine/  # Copy the EFI binary to the limine directory
efibootmgr --create --disk /dev/nvme0n1 --part 1 \  # Create boot entry using efibootmgr
--label "Arch Linux Limine Bootloader" \
--loader '\EFI\limine\BOOTX64.EFI' --unicode
```

### 5.1.2 Setup for Non-Compliant UEFI Motherboards
This section will cover Limine setup for the aforementioned problematic motherboards. Skip this step if you've done [step 5.1](#511-setup-for-compliant-uefi-motherboards) already.

- We will create a *temporary* configuration file as the following steps will create a concrete one.

```bash
mkdir -p /boot/EFI/BOOT/                            # Create fallback directory
cp /usr/share/limine/BOOTX64.EFI /boot/EFI/BOOT     # Copy the EFI binary to the fallback directory
efibootmgr --create --disk /dev/nvme0n1 --part 1 \  # Create boot entry using efibootmgr
--label "Arch Linux Limine Bootloader" \
--loader '\EFI\BOOT\BOOTX64.EFI' --unicode
```

### 5.2 Setting Up Configuration File

Edit the limine.conf file by running:
```bash
nvim /boot/EFI/limine/limine.conf
```
if you've performed [step 5.1](#511-setup-for-compliant-uefi-motherboards) or:

```bash
nvim /boot/EFI/BOOT/limine.conf
```
if you've performed [step 5.1.2](#512-setup-for-non-compliant-uefi-motherboards)


Run `nvim /boot/EFI/BOOT/limine.conf` and write the following:
```bash
timeout: 3

/Arch Linux
    protocol: linux
    path: boot():/vmlinuz-linux
    cmdline: cryptdevice=UUID=<device-UUID>:root root=/dev/mapper/root rw rootflags=subvol=@ rootfstype=btrfs
    module_path: boot():/initramfs-linux.img

/Arch Linux (fallback)
    protocol: linux
    path: boot():/vmlinuz-linux
    cmdline: cryptdevice=UUID=<device-UUID>:root root=/dev/mapper/root rw rootflags=subvol=@ rootfstype=btrfs
    module_path: boot():/initramfs-linux-fallback.img
```

- Tip: Run `:r !cryptsetup luksUUID /dev/<root_partition>` in nvim/vim normal mode to paste the UUID of the root partition directly into the file.

## 6. Final Installation Steps
```bash
systemctl enable NetworkManager dhcpcd iwd \    # Start the downloaded services
systemd-networkd systemd-resolved bluetooth \   # Omit the ones you haven't installed
cups avahi-daemon firewalld acpid reflector.timer

exit                    # Exit chroot
umount -R /mnt          # Unmount everything
cryptsetup close root   # Close encrypted container
reboot                  # Reboot system (Unplug ISO device after)
```

## 7. Post-Install Steps

These steps are all optional but they are are improvements nonetheless.

### 7.1 Install Yay
- Yay is a AUR package helper that installs and manages AUR packages.
```bash
sudo pacman -S --needed git base-devel && git clone https://aur.archlinux.org/yay.git && cd yay && makepkg -si
```

### 7.2 Enable TRIM Support
- Check if your system supports TRIM:
```bash
lsblk --discard
```

- If the output shows non-zero values under column `DISC-GRAN` (Discard Granularity) and `DISC-MAX` (Discard Max Bytes), your disk supports TRIM. Run:
```bash
sudo systemctl enable --now fstrim.timer
```
to enable it.

### 7.3 Pacman Post Install Hook for Limine

Create a new pacman hook by running:
```bash
sudo nvim /etc/pacman.d/hooks/99-limine.hook
```

If you've followed step 5.1.1, write the following to the opened file:
```
[Trigger]
Operation = Install
Operation = Upgrade
Type = Package
Target = limine

[Action]
Description = Deploying Limine after upgrade...
When = PostTransaction
Exec = /usr/bin/cp /usr/share/limine/BOOTX64.EFI /boot/EFI/limine/
```

If you've followed step 5.1.2, write the following to the opened file instead:
```
[Trigger]
Operation = Install
Operation = Upgrade
Type = Package
Target = limine              

[Action]
Description = Deploying Limine after upgrade...
When = PostTransaction
Exec = /usr/bin/cp /usr/share/limine/BOOTX64.EFI /boot/EFI/BOOT/
```

### 7.4 Swap and Hibernation
```bash
sudo btrfs subvolume create /swap   # Create swap subvolume
sudo btrfs filesystem mkswapfile \  # Create swapfile (Change size accordingly)
--size 64g --uuid clear /swap/swapfile
sudo swapon -p 0 /swap/swapfile
sudo nvim /etc/fstab # Open fstab and append `/swap/swapfile none swap defaults,pri=0 0 0` as a new line
sudo mkinitcpio -P                  # Re-create system images
```

### 7.5 Snapper
- Snapper is really useful, highly recommend installing it.
- Lets the user create snapshots and with `limine-snapper-sync` you can synchronize snapshots between limine and snapper for a more secure system.

```bash
# Install the necessary programs
sudo pacman -Syu snapper
yay -S limine-snapper-sync limine-mkinitcpio-hook

# Remove the pre-made snapshots subvolume
sudo umount /.snapshots
sudo rm -rf /.snapshots

# Create snapper configs
sudo snapper -c root create-config /
sudo snapper -c home create-config /home

# Add config values (Adjust to your liking)
sudo sed -i 's/^TIMELINE_CREATE="yes"/TIMELINE_CREATE="no"/' /etc/snapper/configs/{root,home}
sudo sed -i 's/^NUMBER_LIMIT="50"/NUMBER_LIMIT="5"/' /etc/snapper/configs/{root,home}
sudo sed -i 's/^NUMBER_LIMIT_IMPORTANT="10"/NUMBER_LIMIT_IMPORTANT="5"/' /etc/snapper/configs/{root,home}

sudo cp /etc/limine-entry-tool.conf /etc/default/limine # Copy the limine entry tool config to /etc/default/limine
sudo nvim /etc/default/limine                           # Edit the default limine config.
                                                        # Find `ROOT_SNAPSHOTS_PATH` and edit it as "/@snapshots" (i.e. ROOT_SNAPSHOTS_PATH="/@snapshots")
```

- Add snapshots section to `/boot/limine.conf`

```bash
sudo nvim /boot/limine.conf # Edit limine config at EFI.
```

By now, the default config at `/boot/limine.conf` would look something like:
```
timeout: 3

/Arch Linux
	comment: Arch Linux
	comment: machine-id=<ID>
	protocol: linux
	path: boot():/vmlinuz-linux
	cmdline: cryptdevice=UUID=<UUID>:root root=/dev/mapper/root rw rootflags=subvol=@ rootfstype=btrfs
	module_path: boot():/initramfs-linux.img

  //linux
  ### This kernel entry is auto-generated by limine-entry-tool
  comment: Kernel version: 7.0.2-arch1-1
  comment: kernel-id=linux 
  protocol: linux
  module_path: boot():/<machine-id>/linux/initramfs-linux#...
  path: boot():/<machine-id>/linux/vmlinuz-linux#...
  cmdline: cryptdevice=UUID=<UUID>:root root=/dev/mapper/root rw rootflags=subvol=@ rootfstype=btrfs
```

Add the phrase `//Snapshots` after the `cmdline` line. So it would look like:
```
timeout: 3

/Arch Linux
	comment: Arch Linux
	comment: machine-id=<ID>
	protocol: linux
	path: boot():/vmlinuz-linux
	cmdline: cryptdevice=UUID=<UUID>:root root=/dev/mapper/root rw rootflags=subvol=@ rootfstype=btrfs
	module_path: boot():/initramfs-linux.img

  //linux
  ### This kernel entry is auto-generated by limine-entry-tool
  comment: Kernel version: 7.0.2-arch1-1
  comment: kernel-id=linux 
  protocol: linux
  module_path: boot():/<machine-id>/linux/initramfs-linux#...
  path: boot():/<machine-id>/linux/vmlinuz-linux#...
  cmdline: cryptdevice=UUID=<UUID>:root root=/dev/mapper/root rw rootflags=subvol=@ rootfstype=btrfs

  //Snapshots
```

Run:
```bash
limine-snapper-sync
```

If no errors are encountered, run:
```bash
systemctl enable --now limine-snapper-sync.service
```
to enable `limine-snapper-sync`.

Some important commands:
- List snapshots via `limine-snapper-list`
- Synchronize snapper snapshots with limine snapshots via `limine-snapper-sync`
- Get information regarding snapshots via `limine-snapper-info`
- Restore the system by using a select snapshot via `limine-snapper-restore`
- Remove a snapshot via `limine-snapper-remove`

### 7.6 Automatic Firmware Upgrades
```bash
sudo pacman -Syu fwupd  # Install firmware updater
fwupdmgr get-devices
fwupdmgr refresh
fwupdmgr get-updates
fwupdmgr update
sudo systemctl enable --now fwupd-refresh.timer # Enable updater service
```

### 7.7 Finalizing Installation
- By now you should have a pretty concrete Arch Linux system. To make it more productive and desktop-like, we need to install something *visual*.
- I'll be using Hyprland and my other select packages. By all means, you don't need to follow the following section to a T.
    - If you are new to Hyprland, I highly recommend you to use the [Hyprland master tutorial](https://wiki.hypr.land/Getting-Started/Master-Tutorial/). It explains everything in detail.
```bash
sudo pacman -S hyprland hyprlang hyprpaper \
hyprlock hypridle hyprpolkitagent \
hyprshot kitty waybar swaync \
thunar thunar-volman gvfs ttf-jetbrains-mono-nerd \
zip unzip unrar tumbler xdg-user-dirs xdg-desktop-portal-hyprland \
ripgrep pavucontrol npm python nodejs tree-sitter-cli\
libreoffice-fresh libreoffice-fresh-tr firefox btop gdu \
impala bluetui

yay -S walker elephant elephant-desktopapplications # Walker app launcher

start-hyprland  # And enjoy your working system
```

### 8. Quality of Life

#### Dark Theme: 
```bash
sudo pacman -S nwg-look
nwg-look			# Select Adwaita-dark
```

#### iwd Backend for NetworkManager:
Create `/etc/NetworkManager/conf.d/wifi_backend.conf` and add
```
[device]
wifi.backend=iwd
```

#### Signing secure boot keys for Windows and Arch Linux:
1. Install `sbctl`
```bash
sudo pacman -S sbctl	# This will be used to sign the boot options
sbctl status			# The `Firmware` field will have the text
						# `Your firmware has known quirks` if it has
						# any. Refer to that page if you want or
						# proceed from this tutorial.
```

2. Enter firmware setup either using `systemctl reboot --firmware-setup` or pressing the setup key based on your motherboard (Usually delete, F2 or F8)

3. Delete boot keys to enter "Setup Mode":
	- In some MSI motherboards you need to:
		- Set "Secure Boot Mode" to `Custom`
  		- Check if you have the setting "Secure Boot Preset" or "Image Execution Policy":
    		- If you have "Secure Boot Preset" set it to `Maximum Security`.
      		- If you have "Image Execution Policy":
           		- Change "Option ROM", "Removable Media" and "Fixed Media" fields to `Deny Execute` inside it.	  
	- **Important:** Check if your BIOS has a setting that makes it enroll keys on startup and *disable* it.
 	- Find a setting that deletes boot keys or all secure boot variables.
  	- Save and restart. 

4. Check if Setup Mode is enabled:
```bash
sbctl status	# The setup mode field should be `Enabled`
```
If it is enabled, proceed with the tutorial. If not, check if you successfully deleted the boot keys/variables. Re-check if there is a setting that enrolls boot variables on reboot and disable it if you haven't.

5. Create and enroll keys
```bash
sudo sbctl create-keys
sudo sbctl enroll-keys --microsoft
```

6. Fix unsigned entries:
There will be other entries that are not signed automatically (Like the EFI Fallback entry). You can either delete them or sign them yourself.

- *If you want to clear unsigned entries, run:*
```bash
sudo nvim /boot/limine.conf
```
and delete or comment out any unsigned boot entries.

- *For signing entries (The example uses the "EFI Fallback" entry):*
```bash
sudo nvim /etc/limine-entry-tool.conf	# Find line `ENABLE_LIMINE_FALLBACK`
										# Uncomment it and set it to `no`
sudo limine-install --fallback			# Re-generate the fallback binary
b2sum /boot/limine.conf					# Copy the hash from this command
sudo limine-enroll-config /boot/EFI/BOOT/BOOTX64.EFI <PASTE_YOUR_HASH_HERE>
sudo sbctl sign -s /boot/EFI/BOOT/BOOTX64.EFI
```
- **IMPORTANT:** Run
```bash
sudo limine-enroll-config
sudo limine-update
```
if you have edited `/boot/limine.conf`. Else it panics on boot.
