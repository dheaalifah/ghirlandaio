## 1. Memeriksa Status Jaringan
```
nmcli device status
```
## 2. Menampilkan Informasi IP
```
ip
```
## 3. Membuat Koneksi Jaringan Baru
```
Connection 'admin-connection' successfully added.
```
## 4. Membuat User Baru
```
echo "nmcli connection up admin-connection" >> ~/.bash_profile
```
## 5. Membuat Pengguna Baru
```
sudo useradd -m operator
```
## 6. Mengatur Password Pengguna
```
sudo passwd operator
```
## 7. Keluar dari Terminal
```
exit
```
