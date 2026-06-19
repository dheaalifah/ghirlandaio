# Dokumentasi Install OS Server

## Hubungkan ke internet terlebih dahulu
```
iwctl
device list
station wlan0 scan
station wlan0 get-network
station wlan0 connect "nama wifi"
exit
```

## Cek koneksi
```
ping 8.8.8.8
```

## Aktifkan asciinema
```
asciinema rec [nama rekaman].cast
```

## Cek partisi
```
lsblk
```

## Buat Logical Volume
```
lvcreate -L 8G comet -n root
lvcreate -L 5G comet -n vars
lvcreate -L 1G comet -n vlog
lvcreate -L 1G comet -n vaud
lvcreate -L 1.5G comet -n vtmp
lvcreate -l50%FREE comet -n home
lvcreate -l100%FREE comet -n podman
```

## Format partisi
```
mkfs.ext4 /dev/comet/root
mkfs.vfat -F32 /dev/partisi boot
mkfs.ext4 /dev/comet/vars
mkfs.ext4 /dev/comet/vlog
mkfs.ext4 /dev/comet/vaud
mkfs.ext4 /dev/comet/vtmp
mkfs.ext4 /dev/comet/home
mkfs.ext4 /dev/comet/podman
```

## a
