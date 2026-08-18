# Dokumentasi Sistem Backend (BE) Monitoring Job

Dokumentasi ini menyajikan panduan arsitektur, skema database, pemetaan API, serta konfigurasi lingkungan untuk sistem Backend Monitoring Job aplikasi **Kencana Print**.

---

## 1. Pendahuluan & Tujuan Sistem

Sistem Backend Monitoring Job ini dikembangkan untuk mencatat, mengolah, dan menyajikan data produktivitas pekerja (realisasi target SPK) secara langsung per jam operasional di berbagai cabang, lini produksi, dan kelompok kerja.

Sistem ini membantu manajemen untuk:

- Memantau pencapaian target produksi secara _real-time_ per jam.
- Mengontrol alokasi tenaga kerja (_Man Power_) harian di setiap lini.
- Menghasilkan laporan performa (persentase realisasi terhadap target SPK).

---

## 2. Arsitektur Sistem (Pendekatan Hybrid)

Aplikasi ini dibangun menggunakan **Node.js** dan **Express** dengan pendekatan akses data **hybrid**:

1.  **Sequelize ORM:** Digunakan untuk mempermudah operasi CRUD (_Create, Read, Update, Delete_) sederhana pada data master seperti User, Lini, dan Manpower.
2.  **Raw SQL Query:** Digunakan untuk memproses kueri analitis yang rumit (pada modul Laporan dan Monitoring per jam) demi performa kecepatan kueri yang maksimal di database MySQL.

```mermaid
graph TD
    Client[Aplikasi Client / Frontend] -->|HTTP Request| Express[Express.js Server]
    Express -->|Middleware| Auth[JWT Authenticator & CORS]
    Express -->|CRUD Sederhana| ORM[Sequelize ORM]
    Express -->|Analitis & Aggregation| RawSQL[Raw SQL Query]
    ORM -->|Koneksi DB| DB[(MySQL: kencanaprint)]
    RawSQL -->|Koneksi DB| DB
```

---

## 3. Skema Database & Tabel Utama

Sistem ini terhubung ke database **`kencanaprint`** di server VPS. Berikut adalah 8 tabel utama yang saling berelasi:

```mermaid
erDiagram
    tuser {
        string user_kode PK
        string user_password
        string user_nama
        string user_cab
        string user_bagian
        string user_kelompok
    }
    tlini {
        string lini_kode PK
        string lini_nama
    }
    tspk {
        string spk_nomor PK
        string spk_nama
        int spk_jumlah
        date spk_tanggal
        date spk_dateline
        tinyint spk_close
    }
    monjob_spk {
        string ms_nomor PK
        string ms_cab PK
        string ms_lini PK
        double ms_targetperjam
        string user_create
        date date_create
    }
    monjob_manpower {
        dateonly mm_tanggal PK
        string mm_cab PK
        string mm_lini PK
        string mm_kelompok PK
        int mm_mp
        string user_create
    }
    monjob_realisasi {
        dateonly mr_tanggal PK
        string mr_cab PK
        string mr_lini PK
        string mr_kelompok PK
        string mr_jam PK
        string mr_spk_nomor PK
        double mr_realisasi
        double mr_target
        int mr_mp
        string user_create
    }
    monjob_jam {
        int mj_id PK
        string mj_jam
    }
    tkelompok {
        string cab PK
        string lini PK
        string kelompok PK
    }

    monjob_spk ||--o| tspk : "relasi ms_nomor -> spk_nomor"
    monjob_realisasi ||--o| tspk : "relasi mr_spk_nomor -> spk_nomor"
    monjob_realisasi ||--o| monjob_jam : "pencocokan mr_jam -> mj_jam"
```

### Penjelasan Field & Relasi:

- **`tuser`**: Menyimpan kredensial admin/operator cabang.
- **`tspk`**: Master data Surat Perintah Kerja (SPK) dari sistem ERP utama.
- **`monjob_spk`**: Menyimpan pengaturan target kuantitas per jam untuk tiap SPK.
- **`monjob_manpower`**: Menyimpan total alokasi tenaga kerja per tanggal operasional.
- **`monjob_realisasi`**: Pencatatan output kerja riil pekerja per jam operasional.

---

## 4. Standarisasi Format Response API

Semua endpoint API mengembalikan format JSON yang seragam untuk mempermudah integrasi di sisi Frontend.

### 4.1 Format Respon Sukses (HTTP 200 OK)

```json
{
  "ok": true,
  "message": "Pesan keberhasilan operasi",
  "data": { ... } // Berisi objek, array, atau null
}
```

### 4.2 Format Respon Gagal (HTTP 4xx / 5xx)

```json
{
  "ok": false,
  "message": "Detail pesan kesalahan"
}
```

---

## 5. Dokumentasi API (Endpoints)

### 5.1 Autentikasi & Pengguna (Tanpa Proteksi JWT)

- **POST** `/api/admin/login`
  - _Deskripsi:_ Autentikasi operator cabang.
  - _Payload:_ `{ "user_kode": "...", "password": "..." }`
  - _Response Data:_ `{ "token": "...", "user": { "user_kode", "user_nama", "user_cab", ... } }`

- **PUT** `/api/admin/change-password` _(Memerlukan JWT)_
  - _Deskripsi:_ Mengubah password operator yang sedang login.
  - _Payload:_ `{ "user_kode": "...", "old_password": "...", "new_password": "..." }`

### 5.2 Pengaturan Target SPK

- **GET** `/api/spk-lini`
  - _Deskripsi:_ Mengambil daftar lini produksi aktif.
- **GET** `/api/spk-target?lini=...`
  - _Deskripsi:_ Mengambil daftar target SPK berdasarkan lini.
- **POST** `/api/spk-target`
  - _Deskripsi:_ Menyimpan pengaturan target per jam untuk suatu SPK.
- **PUT** `/api/spk-target/:nomor`
  - _Deskripsi:_ Memperbarui target per jam SPK.
- **DELETE** `/api/spk-target/:nomor`
  - _Deskripsi:_ Menghapus pengaturan target SPK.
- **GET** `/api/spk-cari?nomor=...`
  - _Deskripsi:_ Mencari detail SPK dari master ERP berdasarkan nomor SPK.

### 5.3 Input Man Power (Tenaga Kerja)

- **GET** `/api/manpower?lini=...&tanggal=...`
  - _Deskripsi:_ Mengambil daftar jumlah manpower per kelompok kerja.
- **POST** `/api/manpower`
  - _Deskripsi:_ Menyimpan jumlah manpower baru.
- **PUT** `/api/manpower`
  - _Deskripsi:_ Memperbarui jumlah manpower.
- **DELETE** `/api/manpower`
  - _Deskripsi:_ Menghapus data setting manpower.

### 5.4 Input Realisasi Produksi

- **GET** `/api/realisasi/jam-options`
  - _Deskripsi:_ Mengambil opsi daftar jam operasional monitoring.
- **GET** `/api/realisasi?cab=...&tanggal=...`
  - _Deskripsi:_ Memuat daftar transaksi input realisasi.
- **POST** `/api/realisasi`
  - _Deskripsi:_ Menyimpan/meng-update (upsert) data input realisasi produksi pekerja.
- **DELETE** `/api/realisasi`
  - _Deskripsi:_ Menghapus data input realisasi.

### 5.5 Monitoring Live

- **GET** `/api/monitoring?cab=...&tanggal=...&lini=...&kelompok=...`
  - _Deskripsi:_ Mengambil rangkuman persentase pencapaian serta data monitoring per jam.
  - _Response Data:_ `{ "persen": 85.5, "list": [ { "jam", "mp", "spk", "target", "realisasi", "persen" } ] }`
- **GET** `/api/monitoring/detail`
  - _Deskripsi:_ Mengambil detail pemenuhan kuantitas total SPK yang sedang diproduksi.

### 5.6 Laporan Analitis

- **GET** `/api/laporan?cab=...&date_from=...&date_to=...`
  - _Deskripsi:_ Mengambil data akumulasi performa laporan berkinerja tinggi.
  - _Response Data:_ `{ "summary": {...}, "by_date": [...], "by_per_line": [...], "by_spk": [...] }`

---

## 6. Menjalankan di Lingkungan Lokal (Debugging)

1.  Unduh dependensi:
    ```bash
    npm install
    ```
2.  Jalankan server development (menggunakan nodemon):
    ```bash
    npm run dev
    ```
