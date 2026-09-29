# 🔐 Laporan Pentest — Aplikasi Perpustakaan Digital (UK1-Diki)

Repositori ini berisi laporan hasil **Penetration Testing** terhadap aplikasi *Perpustakaan Digital (UK1-Diki)* yang berjalan di lingkungan lab.

## 📋 Isi Laporan

Laporan lengkap dapat dilihat di:

👉 **[laporan/laporan.md](laporan/laporan.md)**

## 📊 Ringkasan Temuan

| Severity    | Jumlah |
|-------------|--------|
| 🔴 Critical | 2 |
| 🟠 High     | 3 |
| 🟡 Medium   | 2 |
| 🟢 Low      | 1 |

**Temuan utama:**
- Backdoor Reset Password (`reset_password.php`)
- Kredensial Database Hardcoded — `root` tanpa password
- Source Code Bocor via GitHub Public Repo
- `.git` Directory Bisa Diakses
- Session Hijacking via HTTP Sniffing
- Database SQL Dump Tersedia di Source Code
- Backup Folder `perpus_work/` Tidak Dihapus
- Informasi Server Bocor

## 🎯 Tujuan

Laporan ini dibuat untuk keperluan **pembelajaran** dan **authorized pentest**. Segala aktivitas yang dilakukan hanya pada lingkungan lab yang telah diizinkan.

## 📖 Cara Membaca

1. Buka file **[laporan/laporan.md](laporan/laporan.md)**
2. Baca **Ringkasan Eksekutif** untuk gambaran umum
3. Lihat **Temuan** untuk detail teknis + PoC
4. Ikuti **Rekomendasi Perbaikan** untuk mitigasi

## ⚠️ Disclaimer

> Laporan ini dibuat untuk keperluan pembelajaran / authorized pentest.  
> Penggunaan tanpa izin adalah **ilegal**.

---

**Author:** gh0st4n  
**Tanggal:** 28 September 2026
