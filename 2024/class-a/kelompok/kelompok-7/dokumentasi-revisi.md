# Instalasi Arch Linux dengan Disk Layout CIS

Setup: LUKS on LVM, systemd-boot + booster, XFCE desktop

---

## Spesifikasi

| Komponen | Value |
|----------|-------|
| Boot | booster |
| Disk Encryption | LUKS on LVM |
| Desktop | XFCE |
| File Manager | Superfile (spf) |
| Disk Layout | CIS |
| Container | Podman, Podman Desktop |
| Firewall | IPTables |
| Multimedia | MPD, MPC, MPV |
| Password Manager | KeePassXC |
| Key Manager | Secrets |
| Remote Access | OpenSSH |

---

## 1. Connect WiFi

```bash
iwctl
device list
station wlan0 get-network
station wlan0 scan
station wlan0 connect "(nama wifi)"
exit
ping 1.1.1.1 -c 4
```

---

## 2. Partisi

Cek partisi yang ada:
```bash
lsblk -o name,fstype,size
lsblk
```

Buat partisi baru:
```bash
cfdisk /dev/(nama disk)
# contoh: cfdisk /dev/nvme0n1
```

Layout yang dibuat (sesuaikan dengan disk masing-masing):
- `(partisi boot)` = 2G → EFI System (boot)
- `(partisi lvm)` = sisa → Linux filesystem (LVM)

```bash
lsblk
```

---

## 3. Setup LVM

> Ganti `(partisi lvm)` dengan partisi yang dibuat tadi, contoh: `/dev/nvme0n1p6`

```bash
pvcreate /dev/(partisi lvm)
vgcreate proc /dev/(partisi lvm)
lvcreate -L 20G proc -n root
lvcreate -L 10G proc -n vars
lvcreate -L 4G proc -n vtmp
lvcreate -L 4G proc -n vlog
lvcreate -L 2G proc -n vaud
lvcreate -L 10G proc -n home
lvcreate -l 50%FREE proc -n crab
lsblk
```

---

## 4. Formatting

```bash
mkfs.ext4 /dev/proc/root
mkfs.vfat -F32 -n BOOT /dev/(partisi boot)
mkfs.ext4 /dev/proc/vars
mkfs.ext4 /dev/proc/vtmp
mkfs.ext4 /dev/proc/vlog
mkfs.ext4 /dev/proc/vaud
mkfs.ext4 /dev/proc/home
mkfs.ext4 /dev/proc/crab
```

---

## 5. Setup LUKS

Volume `crab` di-encrypt pakai LUKS, nantinya di-mount ke `/home/(username)` saat login.

> Nama device LUKS pakai username kalian, contoh: `dika`

```bash
cryptsetup luksFormat /dev/proc/crab
# ketik YES lalu masukkan passphrase (ingat passphrase ini, harus sama dengan password user nanti)
cryptsetup luksOpen /dev/proc/crab (username)
mkfs.ext4 /dev/mapper/(username)
```

---

## 6. Mounting

> Ganti `(partisi boot)` dengan partisi EFI kalian

```bash
mount /dev/proc/root /mnt
mount --mkdir -o uid=0,gid=0,dmask=0077,fmask=0077 /dev/(partisi boot) /mnt/boot
mount --mkdir -o rw,nodev,nosuid,relatime /dev/proc/vars /mnt/var
mount --mkdir -o rw,nodev,nosuid,noexec,relatime /dev/proc/vtmp /mnt/var/tmp
mount --mkdir -o rw,nodev,nosuid,noexec,relatime /dev/proc/vlog /mnt/var/log
mount --mkdir -o rw,nodev,nosuid,noexec,relatime /dev/proc/vaud /mnt/var/log/audit
mount --mkdir -o rw,nodev,nosuid,noexec,relatime /dev/proc/home /mnt/home
lsblk
```

---

## 7. Install Packages

Intel:
```bash
pacstrap /mnt intel-ucode linux-lts linux-lts-headers linux-firmware lvm2 base base-devel neovim openssh superfile podman podman-desktop iptables mpd mpc mpv keepassxc secrets booster networkmanager pam_mount
```

AMD:
```bash
pacstrap /mnt amd-ucode linux-lts linux-lts-headers linux-firmware lvm2 base base-devel neovim openssh superfile podman podman-desktop iptables mpd mpc mpv keepassxc secrets booster networkmanager pam_mount
```

---

## 8. Generate fstab

```bash
genfstab -U /mnt > /mnt/etc/fstab
echo "/tmpfs /tmp tmpfs defaults,nosuid,nodev,noexec,size=1G 0 0" >> /mnt/etc/fstab
```

---

## 9. Chroot

```bash
arch-chroot /mnt
```

---

## 10. Hostname

> Ganti `(nama hostname)` sesuai keinginan

```bash
echo (nama hostname) > /etc/hostname
```

---

## 11. Timezone

```bash
ln -fs /usr/share/zoneinfo/Asia/Jakarta /etc/localtime
hwclock --systohc
```

---

## 12. Locale

Edit `/etc/locale.gen`, uncomment `en_US.UTF-8 UTF-8`:

```bash
nvim /etc/locale.gen
```

Generate:
```bash
locale-gen
locale > /etc/locale.conf
```

Edit `/etc/locale.conf`:
```bash
nvim /etc/locale.conf
```

Isi:
```
LANG=en_US.UTF-8
LC_CTYPE="C.UTF-8"
LC_NUMERIC="C.UTF-8"
LC_TIME="C.UTF-8"
LC_COLLATE="C.UTF-8"
LC_MONETARY="C.UTF-8"
LC_MESSAGES=
LC_PAPER="C.UTF-8"
LC_NAME="C.UTF-8"
LC_ADDRESS="C.UTF-8"
LC_TELEPHONE="C.UTF-8"
LC_MEASUREMENT="C.UTF-8"
LC_IDENTIFICATION="C.UTF-8"
LC_ALL=en_US.UTF-8
```

---

## 13. User & Sudo

> Ganti `(username)` dengan username kalian

```bash
mkdir /home/user
useradd -d /home/user (username)
passwd (username)
# masukkan password, harus sama dengan passphrase LUKS tadi
chown -R (username):(username) /home/user
passwd
# set password root, boleh sama atau beda
echo '(username) ALL=(ALL:ALL) ALL' >> /etc/sudoers.d/none
```

> **Penting:** password user harus sama dengan passphrase LUKS supaya pam_mount bisa otomatis unlock `/home/(username)` saat login

---

## 14. Konfigurasi pam_mount

Edit `/etc/security/pam_mount.conf.xml`:

```bash
nvim /etc/security/pam_mount.conf.xml
```

> Ganti `(username)` dengan username kalian

```xml
<?xml version="1.0" encoding="utf-8" ?>
<!DOCTYPE pam_mount SYSTEM "pam_mount.conf.xml.dtd">
<pam_mount>

<debug enable="0" />

<mntoptions allow="nosuid,nodev,loop,encryption,fsck,nonempty,allow_root,allow_other" />
<mntoptions require="nosuid,nodev" />

<logout wait="0" hup="no" term="no" kill="no" />

<volume
    user="(username)"
    fstype="crypt"
    path="/dev/proc/crab"
    mountpoint="/home/(username)"
/>

<mkmountpoint enable="1" remove="true" />

</pam_mount>
```

Edit `/etc/pam.d/system-login`:

```bash
nvim /etc/pam.d/system-login
```

```
#%PAM-1.0
auth       required   pam_shells.so
auth       requisite  pam_nologin.so
auth       include    system-auth
auth       optional   pam_mount.so

account    required   pam_access.so
account    required   pam_nologin.so
account    include    system-auth

password   include    system-auth

session    optional   pam_loginuid.so
session    optional   pam_keyinit.so       force revoke
session    include    system-auth
session    optional   pam_lastlog2.so      silent
session    optional   pam_motd.so
session    optional   pam_mail.so          dir=/var/spool/mail standard quiet
session    optional   pam_umask.so
session    optional   pam_mount.so
-session   optional   pam_systemd.so
session    required   pam_env.so
```

---

## 15. Booster

Edit `/etc/booster.yaml`:

```bash
nvim /etc/booster.yaml
```

```yaml
network:
  dhcp: on
universal: false
modules: -*,ext4,nvme
extra_files: fsck,fsck.ext4
strip: true
enable_lvm: true
```

Build initramfs:

```bash
cd /boot
ls /usr/lib/modules
# catat versi kernel yang muncul, lalu ganti di command berikut
booster build --kernel-version (versi kernel) /boot/booster-linux-lts-new.img
rm -fr booster-linux-lts.img
```

> Contoh versi kernel: `6.18.33-1-lts`

---

## 16. Systemd-boot

```bash
bootctl --path=/boot install
```

Edit `/boot/loader/entries/booster.conf`:

```bash
nvim /boot/loader/entries/booster.conf
```

```
title    arch with booster
linux    /vmlinuz-linux-lts
initrd   /intel-ucode.img
initrd   /booster-linux-lts-new.img
options  root=/dev/proc/root rw
```

> Kalau pakai AMD ganti `intel-ucode.img` jadi `amd-ucode.img`

Edit `/boot/loader/loader.conf`:

```bash
nvim /boot/loader/loader.conf
```

```
#timeout 3
#console-mode keep
default  booster.conf
```

```bash
bootctl --graceful update
```

---

## 17. Install Desktop

```bash
pacman -S xfce4 sddm pipewire pipewire-pulse pipewire-jack pipewire-alsa xorg-server xorg-xauth xf86-video-vesa xf86-video-fbdev xfce4-screensaver xfce4-power-manager
systemctl enable sddm
systemctl enable NetworkManager
```

Tambahkan user ke group yang diperlukan:

> Ganti `(username)` dengan username kalian

```bash
usermod -aG video,input,tty (username)
echo "needs_root_rights = yes" > /etc/X11/Xwrapper.config
```

---

## 18. Selesai

```bash
exit
umount -R /mnt
reboot
```

---
