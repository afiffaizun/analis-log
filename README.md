# 🔐 Panduan Log Autentikasi Linux

> Referensi lengkap untuk memantau keamanan sistem melalui log autentikasi di Linux (Debian/Ubuntu).

---

## 📋 Daftar Isi

- [auth.log — Log Autentikasi Utama](#1-authlog--log-autentikasi-utama-)
- [last — Riwayat Login Pengguna](#2-last--riwayat-login-pengguna-)
- [Perintah Analisis Lanjutan](#-perintah-analisis-lanjutan)
- [Skenario Keamanan & Cara Mendeteksi](#-skenario-keamanan--cara-mendeteksi)
- [Tips & Best Practice](#-tips--best-practice)

---

## 1. `auth.log` — Log Autentikasi Utama

###  Lokasi File

```
/var/log/auth.log          # Debian / Ubuntu
/var/log/secure            # CentOS / RHEL / Fedora
```

###  Fungsi Utama

File ini mencatat **semua aktivitas autentikasi** yang terjadi di sistem, meliputi:

| Kategori              | Yang Dicatat                                      |
|-----------------------|--------------------------------------------------|
| 🔑 Login SSH          | Berhasil/gagal, IP sumber, username, port         |
| 🛡️ Sudo               | Perintah yang dijalankan dengan `sudo`, oleh siapa |
| ❌ Kegagalan Auth     | Password salah, user tidak ada, key ditolak       |
| 🔨 Brute Force        | Percobaan login berulang dari IP yang sama        |
| 👤 Pembuatan User     | Akun baru dibuat via `useradd` / `adduser`        |
| 🔄 Ganti Password     | Perubahan password via `passwd`                   |

---

### 🔍 Perintah Dasar

#### Lihat log terbaru (real-time)
```bash
sudo tail -f /var/log/auth.log
```

#### Lihat 50 baris terakhir
```bash
sudo tail -n 50 /var/log/auth.log
```

#### Cari login SSH yang berhasil
```bash
sudo grep "Accepted" /var/log/auth.log
```

#### Cari percobaan login yang gagal
```bash
sudo grep "Failed password" /var/log/auth.log
```

#### Cari aktivitas sudo
```bash
sudo grep "sudo" /var/log/auth.log
```

#### Cari pembuatan user baru
```bash
sudo grep "useradd\|adduser\|new user" /var/log/auth.log
```

#### Cari perubahan password
```bash
sudo grep "password changed\|chpasswd\|passwd" /var/log/auth.log
```

---

### 📖 Contoh Output & Cara Membacanya

#### ✅ Login SSH Berhasil
```
Jun 27 08:15:42 server sshd[1234]: Accepted publickey for alice from 192.168.1.10 port 52341 ssh2
```
| Field        | Nilai              | Arti                          |
|--------------|--------------------|-------------------------------|
| Tanggal/Waktu | Jun 27 08:15:42   | Kapan kejadian terjadi        |
| Hostname     | server             | Nama mesin                    |
| Service      | sshd[1234]         | SSH daemon, PID 1234          |
| Status       | Accepted publickey | Login berhasil via SSH key    |
| User         | alice              | Pengguna yang login           |
| IP Asal      | 192.168.1.10       | Dari mana koneksi datang      |

#### ❌ Login SSH Gagal
```
Jun 27 08:20:01 server sshd[1235]: Failed password for root from 45.33.32.156 port 4821 ssh2
```
> ⚠️ Percobaan login sebagai `root` dari IP asing — indikasi potensi brute force.

#### 🛡️ Eksekusi Sudo
```
Jun 27 09:00:15 server sudo: alice : TTY=pts/0 ; PWD=/home/alice ; USER=root ; COMMAND=/bin/systemctl restart nginx
```
> Menunjukkan `alice` menjalankan `systemctl restart nginx` sebagai root.

#### 👤 User Baru Dibuat
```
Jun 27 10:05:33 server useradd[1500]: new user: name=bob, UID=1002, GID=1002, home=/home/bob
```

---

## 2. `last` — Riwayat Login Pengguna 🕒

### 📌 Fungsi Utama

Perintah `last` membaca file `/var/log/wtmp` dan menampilkan **riwayat semua login dan logout** pengguna secara kronologis.

### 🔍 Perintah Dasar

#### Tampilkan semua riwayat login
```bash
last
```

#### Login pengguna tertentu
```bash
last alice
```

#### Login dari IP/hostname tertentu
```bash
last -i 192.168.1.10
```

#### Tampilkan N login terakhir
```bash
last -n 20
```

#### Tampilkan login + IP penuh (tanpa DNS lookup)
```bash
last -a -i
```

#### Lihat siapa yang sedang login sekarang
```bash
who
# atau
w
```

#### Lihat login yang gagal (dari /var/log/btmp)
```bash
sudo lastb
sudo lastb -n 20      # 20 percobaan gagal terakhir
```

#### Waktu reboot sistem
```bash
last reboot
```

---

### 📖 Contoh Output `last`

```
alice    pts/0        192.168.1.10     Fri Jun 27 08:15   still logged in
bob      pts/1        10.0.0.5         Fri Jun 27 07:30 - 09:00  (01:30)
root     pts/2        203.0.113.45     Thu Jun 26 23:00 - 23:05  (00:05)
reboot   system boot  5.15.0-91-generic Thu Jun 26 22:55   still running
```

| Kolom        | Arti                                          |
|--------------|-----------------------------------------------|
| `alice`      | Username yang login                           |
| `pts/0`      | Terminal (pts = SSH, tty = lokal)             |
| `192.168.1.x`| IP asal koneksi                              |
| `Fri Jun 27` | Tanggal login                                 |
| `still logged in` | Masih aktif / sudah logout + durasi     |

---

### ⚠️ Hal yang Perlu Diwaspadai di `last`

```bash
# Login root dari IP eksternal
last root | grep -v "0.0.0.0\|:0\|reboot"

# Sesi sangat singkat (< 1 menit) — mencurigakan
last | grep "00:0"

# Login di jam tidak wajar (tengah malam)
last | grep "23:\|00:\|01:\|02:\|03:"
```

---

## 🔧 Perintah Analisis Lanjutan

### Hitung IP yang paling sering gagal login (deteksi brute force)
```bash
sudo grep "Failed password" /var/log/auth.log \
  | awk '{print $(NF-3)}' \
  | sort | uniq -c | sort -rn | head -20
```

### Daftar username yang dicoba saat brute force
```bash
sudo grep "Invalid user" /var/log/auth.log \
  | awk '{print $8}' \
  | sort | uniq -c | sort -rn | head -20
```

### Ringkasan statistik kegagalan login
```bash
sudo grep "Failed password" /var/log/auth.log | wc -l
```

### Monitor log secara langsung (live)
```bash
sudo tail -f /var/log/auth.log | grep --line-buffered "Failed\|Accepted\|sudo\|useradd"
```

### Cari aktivitas dalam rentang waktu tertentu
```bash
sudo awk '/Jun 27 08:00/,/Jun 27 09:00/' /var/log/auth.log
```

---

## 🚨 Skenario Keamanan & Cara Mendeteksinya

### 🔴 Brute Force SSH
**Tanda-tanda:**
- Ratusan baris `Failed password` dari IP yang sama dalam waktu singkat
- Mencoba username berbeda-beda (`admin`, `root`, `ubuntu`, dll.)

**Deteksi:**
```bash
sudo grep "Failed password" /var/log/auth.log \
  | awk '{print $(NF-3)}' | sort | uniq -c | sort -rn
```

**Mitigasi:**
```bash
# Blokir IP dengan fail2ban
sudo apt install fail2ban
# atau blokir manual
sudo ufw deny from 45.33.32.156
```

---

### 🟠 Login Root Langsung via SSH
**Tanda-tanda:**
```
Accepted password for root from 203.0.113.45
```

**Deteksi:**
```bash
sudo grep "Accepted.*for root" /var/log/auth.log
```

**Mitigasi:** Nonaktifkan login root di `/etc/ssh/sshd_config`:
```
PermitRootLogin no
```

---

### 🟡 Eskalasi Privilege Mencurigakan
**Tanda-tanda:**
- User biasa tiba-tiba menggunakan `sudo` untuk perintah sensitif

**Deteksi:**
```bash
sudo grep "sudo" /var/log/auth.log | grep -v "pam_unix"
```

---

### 🔵 Pembuatan User Tidak Sah
**Tanda-tanda:**
```
new user: name=hacker, UID=0
```

**Deteksi:**
```bash
sudo grep "new user\|useradd" /var/log/auth.log
```

---

## 💡 Tips & Best Practice

### Rotasi Log Otomatis
Log dirotasi oleh `logrotate`. Lihat log lama di:
```bash
ls /var/log/auth.log*
# auth.log  auth.log.1  auth.log.2.gz  auth.log.3.gz ...
```
Untuk cari di log lama:
```bash
sudo zgrep "Failed password" /var/log/auth.log.2.gz
```

### Aktifkan Notifikasi Otomatis
Install `fail2ban` untuk blokir otomatis dan kirim notifikasi email:
```bash
sudo apt install fail2ban
sudo systemctl enable fail2ban
```

### Rekomendasi Keamanan SSH

```bash
# /etc/ssh/sshd_config — konfigurasi yang disarankan
PermitRootLogin no           # Larang login root
PasswordAuthentication no    # Hanya izinkan SSH key
MaxAuthTries 3               # Maksimal 3 percobaan
LoginGraceTime 30            # Timeout 30 detik
AllowUsers alice bob         # Hanya izinkan user tertentu
```

---

## 📚 Referensi Cepat

| Perintah                          | Fungsi                                      |
|-----------------------------------|---------------------------------------------|
| `tail -f /var/log/auth.log`       | Monitor log real-time                       |
| `grep "Failed" /var/log/auth.log` | Semua kegagalan login                       |
| `grep "Accepted" /var/log/auth.log`| Semua login berhasil                       |
| `last`                            | Riwayat login semua user                    |
| `last -n 20`                      | 20 login terakhir                           |
| `lastb`                           | Riwayat login yang gagal                    |
| `who`                             | Siapa yang sedang login                     |
| `w`                               | Login aktif + aktivitas saat ini            |
| `last reboot`                     | Riwayat reboot sistem                       |

---

> 📝 **Catatan:** Semua perintah `grep` pada `/var/log/auth.log` memerlukan `sudo` atau akses root. Simpan hasil analisis secara berkala sebagai dokumentasi audit keamanan sistem.
