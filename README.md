# Siloam Meals — Backend API Service

> **Sistem Informasi & Manajemen Pemesanan Makanan Pasien Rawat Inap Rumah Sakit Berbasis Restriksi Diet Klinis.**

Backend RESTful API berbasis **NestJS** dan **PostgreSQL (Prisma ORM)** yang mengotomatiskan alur kerja penyediaan makanan pasien rawat inap rumah sakit: mulai dari pendaftaran pasien oleh perawat, telaah & validasi diet oleh ahli gizi, pemesanan mandiri oleh pasien via QR Code, hingga rekapitulasi produksi dapur untuk esok hari (H+1).

---

## 🎯 Masalah yang Diselesaikan

Pada operasional rumah sakit konvensional, pencatatan diet pasien di atas kertas rentan salah baca yang berisiko memicu alergi berbahaya bagi pasien. Selain itu, staf dapur sering kesulitan memperkirakan porsi masak yang berujung pada pemborosan bahan baku (_food waste_).

**Siloam Meals Backend** hadir sebagai mesin logika bisnis (_business rules engine_) yang menjamin:

1. **Keamanan Diet Pasien (Patient Safety):** Pasien hanya disajikan makanan yang terbukti aman dari alergen dan diagnosa medisnya.
2. **Efisiensi Produksi Dapur:** Seluruh pemesanan terkunci untuk H+1 (esok hari), memberikan dapur kepastian angka porsi masak per sesi (Pagi/Siang/Malam).
3. **Preservasi Rekam Medis:** Riwayat makanan yang pernah dikonsumsi pasien dibekukan (_snapshot pattern_) sehingga data transaksi tetap utuh untuk audit hukum, meski data pasien atau menu telah dihapus.

---

## 🔄 Alur Kerja Sistem (System Workflow)

```
[1. Perawat (Nurse)]
       │  Mendaftarkan pasien, diagnosa, dan pantangan awal
       ▼
[2. Ahli Gizi (Dietisien)]
       │  Meninjau alergen, menentukan blacklist makanan khusus, & validasi klinis
       ▼
[3. Pasien Rawat Inap]
       │  Scan QR Code kamar tanpa login ──► Sistem memfilter menu aman ──► Pesan menu H+1
       ▼
[4. Dapur RS (Kitchen)]
       │  Menerima rekap porsi pasti per sesi (Pagi, Siang, Malam) untuk dimasak esok hari
       ▼
[5. Manajemen (Admin)]
          Memantau dashboard statistik pasien rawat inap & rekap riwayat pemesanan global
```

---

## ⚙️ Logika Bisnis Utama (Core Business Rules)

- **Smart Food Safety Filter:** Saat pasien meminta menu, sistem mengeliminasi makanan terlarang berdasarkan irisan:

  $$
  \mathcal{F}_{\text{terlarang}} = \text{Pantangan Perawat} \cup \text{Pengecualian Ahli Gizi}
  $$

  Jika ada upaya memesan makanan terlarang via API, server seketika menolak dengan `HTTP 400 Bad Request`.

- **Dual-Gate Ordering Rule:** Pasien baru diobservasi 24 jam pertama (menerima menu standar dapur). Hak pemesanan mandiri hanya aktif bila status sudah `ACTIVE` (dinaikkan otomatis oleh _hourly scheduler_) **DAN** telah divalidasi oleh Ahli Gizi (`validate === true`).
- **Siklus Pre-Order H+1 (Next-Day):** Seluruh pemesanan mandiri pasien ditujukan untuk konsumsi esok hari dengan kuota tepat 1 pesanan per sesi (Menu Pagi, Siang, Malam).
- **Audit Trail Snapshot Pattern:** Bila pasien atau menu dihapus dari database, relasi diubah menjadi `NULL` (`ON DELETE SET NULL`) dan data teks identitas/makanan tetap tersimpan permanen di tabel riwayat.

---

## 👥 Aktor & Hak Akses (Role-Based Access Control)

| Aktor / Role                  | Akses Utama di Backend                                                                                          |
| :---------------------------- | :-------------------------------------------------------------------------------------------------------------- |
| **NURSE (Perawat)**           | Mendaftarkan pasien, input pantangan awal, generate QR Code, dan menyelesaikan _feedback_ diet.                 |
| **DIETISIEN (Ahli Gizi)**     | Menetapkan pengecualian makanan (`PengecualianMakanan`), validasi klinis pasien, dan mengirim feedback gizi.    |
| **KITCHEN (Staf Dapur)**      | Mengelola katalog makanan, mengunggah foto, mengatur jadwal ketersediaan bulanan, dan memantau antrean pesanan. |
| **PASIEN (Tokenless via QR)** | Akses menu yang dipersonalisasi dan submit pesanan H+1 via token unik UUID v4 (`/pesanan/:uuid`).               |
| **ADMIN (Manajemen TI/RS)**   | Hak pengawasan penuh (_super-user_), dashboard metrik operasional, dan arsip riwayat pesanan.                   |

---

## 🛠️ Tech Stack & Arsitektur

- **Framework:** [NestJS](https://nestjs.com/) (TypeScript) — Arsitektur modular layer (Controller $\rightarrow$ Service $\rightarrow$ Prisma ORM).
- **Database:** [PostgreSQL](https://www.postgresql.org/) — Relasi 10 entitas dengan penanganan integritas ACID.
- **ORM:** [Prisma ORM](https://www.prisma.io/) — Database schema migrations & relational queries.
- **Autentikasi & Keamanan:** Passport JWT, Bcrypt password hashing, dan `RolesGuard` terpusat.
- **Otomasi Background:** `@nestjs/schedule` (Cron Job) untuk auto-aktivasi status karantina pasien tiap jam.
- **File Upload & QR Code:** Multer disk storage (foto menu) dan library `qrcode` (generate data URL QR).

---

## 📁 Struktur Modul Backend

```text
src/
├── auth/        # Autentikasi JWT, password hashing, dan RolesGuard (RBAC)
├── pasien/      # CRUD Pasien, QR generator, pengecualian diet, & cron scheduler
├── makanan/     # Katalog makanan, upload gambar (Multer), dan jadwal ketersediaan
├── menu/        # Pengelompokan sesi sajian (Menu Pagi, Siang, Malam)
├── pesanan/     # Validasi keamanan pangan, pesanan publik H+1, & antrean dapur
├── feedback/    # Alur komunikasi asinkron Ahli Gizi ──► Perawat bangsal
├── app.module.ts
└── main.ts
```

---

## 🚀 Panduan Menjalankan Proyek (Quick Start)

### 1. Prasyarat

- Node.js v18+
- PostgreSQL Database

### 2. Konfigurasi Environment

Buat file `.env` di root direktori dengan format:

```env
DATABASE_URL="postgresql://username:password@localhost:5432/siloam_meals?schema=public"
JWT_SECRET="rahasia_jwt_anda"
PORT=8080
APP_URL="http://localhost:8080"
FRONTEND_URL="http://localhost:3000"
```

### 3. Instalasi & Migrasi Database

```bash
# Instalasi dependensi
$ npm install

# Jalankan migrasi Prisma
$ npx prisma migrate dev

# Isi data awal (seed user & menu dasar)
$ npm run seed
```

### 4. Menjalankan Server

```bash
# Mode development (watch mode)
$ npm run start:dev

# Mode production
$ npm run start:prod
```

Aplikasi backend akan berjalan di `http://localhost:8080`.
