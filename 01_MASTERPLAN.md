# MASTERPLAN - Sistem Manajemen Hotel Multi Tenant

Dokumen ini adalah panduan utama untuk AI agent yang mengerjakan pengembangan sistem ini secara vibe coding. Baca dokumen ini sampai selesai sebelum menulis kode apapun. Dokumen ini harus dibaca bersamaan dengan tiga dokumen berikut, dan keempatnya harus dianggap sebagai satu kesatuan aturan yang tidak boleh dilanggar:

- 02_API_CONTRACT.md
- 03_DATABASE_SCHEMA.md
- 04_CODE_STANDARDS.md

Jika ada instruksi dari user yang bertentangan dengan dokumen ini, tanyakan dulu ke user sebelum melanjutkan, jangan menebak.

---

## 1. Gambaran Proyek

Sistem ini adalah Property Management System (PMS) untuk hotel, dengan model bisnis Software as a Service (SaaS) multi tenant. Satu instalasi sistem ini melayani banyak hotel sekaligus (disebut tenant). Setiap hotel yang berlangganan mendapat:

1. Akses ke panel manajemen internal (PMS) untuk mengelola operasional hotel sehari hari.
2. Sebuah booking engine publik (website booking) yang terhubung ke PMS lewat API, sehingga tamu bisa booking kamar langsung secara online.

Ada dua jenis pengguna sistem yang harus dibedakan dengan jelas:

- **Pemilik platform (kamu)**: mengelola daftar hotel yang berlangganan, paket langganan, billing ke hotel. Ini disebut Central Panel.
- **Staf hotel (tenant)**: resepsionis, housekeeping, kasir restoran, akunting, manajer hotel, dan seterusnya. Mereka bekerja di dalam data hotel masing masing saja, tidak bisa melihat data hotel lain. Ini disebut Tenant Panel.
- **Tamu hotel**: menggunakan booking engine React, bukan panel Filament, dan hanya bicara dengan sistem lewat API publik yang terbatas.

## 2. Tech Stack yang Wajib Dipakai

Jangan mengganti stack ini tanpa persetujuan eksplisit dari user, walaupun ada alternatif yang menurut AI lebih baik.

| Komponen | Pilihan |
|---|---|
| Backend framework | Laravel (versi terbaru stabil) |
| Admin panel | Filament |
| Database | PostgreSQL |
| Cache dan queue | Redis |
| Primary key | UUID versi 7 di semua tabel, kecuali tabel pivot murni bila disebutkan lain di 03_DATABASE_SCHEMA.md |
| Modularisasi backend | nwidart/laravel-modules |
| Multi tenancy | stancl/tenancy, mode single database dengan kolom tenant_id |
| Autentikasi API | Laravel Sanctum |
| Role dan permission | spatie/laravel-permission |
| Audit log | spatie/laravel-activitylog |
| Frontend booking engine | React |
| API style | REST, format JSON, mengikuti 02_API_CONTRACT.md |

Integrasi OTA (Traveloka, Agoda, dan sejenisnya) dan Channel Manager **tidak dikerjakan dulu**. Jangan buat kode, tabel, atau endpoint untuk fitur ini kecuali diminta secara eksplisit oleh user nanti.

## 3. Daftar Modul Sistem

Setiap modul di bawah ini adalah satu unit kerja terpisah di dalam folder Modules/. Urutan di bawah adalah urutan prioritas pengerjaan, karena modul di bawah bergantung pada modul di atasnya.

1. **Tenancy** - fondasi multi tenant, panel central, identifikasi tenant
2. **FrontOffice** - booking kamar, check-in, check-out, status kamar, data tamu, invoice tamu
3. **Housekeeping** - kebersihan kamar, status kamar (clean, dirty, vacant), jadwal cleaning, permintaan laundry
4. **PointOfSale** - transaksi restoran, cafe, minibar, room service, split bill, integrasi ke tagihan kamar
5. **Inventory** - stok makanan, perlengkapan kamar, amenity hotel, pembelian barang
6. **Purchasing** - purchase request, purchase order, approval pembelian, supplier management
7. **Accounting** - jurnal umum, hutang piutang, cash flow, laporan laba rugi, pajak
8. **CRM** - member loyalty, promo pelanggan, histori tamu, feedback customer
9. **Reporting** - dashboard okupansi hotel, revenue, kamar terlaris, performa restoran, dashboard manajemen
10. **ChannelManager** - DITUNDA, jangan dikerjakan sampai diminta

Detail tabel tiap modul ada di 03_DATABASE_SCHEMA.md. Detail endpoint tiap modul ada di 02_API_CONTRACT.md.

## 4. Aturan Ketergantungan Antar Modul

Modul boleh membaca data modul lain, tapi komunikasi perubahan data (write) antar modul wajib lewat Laravel Event dan Listener, bukan pemanggilan langsung antar service class dari modul berbeda.

Contoh yang benar:
- Modul PointOfSale menyelesaikan transaksi, lalu memicu event TransactionCompleted.
- Modul Accounting mendengarkan event tersebut lewat listener, lalu membuat jurnal otomatis.

Contoh yang salah:
- Modul PointOfSale memanggil langsung class JournalPostingService milik modul Accounting.

Alasan aturan ini: supaya tiap modul bisa dites sendiri, dan supaya AI agent yang mengerjakan satu modul tidak perlu memahami seluruh isi modul lain.

## 5. Prinsip Data dan Uang

- Semua nilai uang disimpan sebagai integer, dalam satuan rupiah terkecil (bukan float, bukan decimal). Konversi ke format tampilan dilakukan di layer presentasi saja.
- Semua tabel transaksi wajib punya kolom tenant_id, kecuali dijelaskan lain di 03_DATABASE_SCHEMA.md.
- Semua primary key memakai UUID versi 7.
- Semua tabel wajib punya created_at, updated_at. Tabel yang datanya tidak boleh hilang permanen (misal reservasi, transaksi, jurnal) wajib punya deleted_at (soft delete).

## 6. Tahapan Pengerjaan untuk Agentic AI

Kerjakan modul secara berurutan sesuai daftar di bagian 3. Untuk setiap modul, urutan kerja yang benar adalah:

1. Buat migration sesuai skema di 03_DATABASE_SCHEMA.md
2. Buat model beserta relasi, terapkan trait UUID7 dan trait tenant scope
3. Buat factory dan seeder untuk data contoh
4. Buat Filament Resource untuk modul terkait (form dan tabel di panel tenant)
5. Buat API Controller dan Route sesuai 02_API_CONTRACT.md, hanya untuk endpoint yang memang dipakai booking engine React atau integrasi eksternal
6. Tulis test dasar (minimal test untuk alur utama, bukan 100 persen coverage)
7. Baru lanjut ke modul berikutnya

Jangan mengerjakan banyak modul sekaligus secara paralel dalam satu sesi kerja. Selesaikan satu modul sampai tahap 6 sebelum pindah ke modul berikutnya, supaya perubahan mudah ditelusuri.

## 7. Definisi Selesai (Definition of Done) per Modul

Satu modul dianggap selesai jika:

- Migration bisa dijalankan dari kosong tanpa error (migrate:fresh)
- Seeder menghasilkan data contoh yang masuk akal untuk modul tersebut
- Semua endpoint yang didefinisikan di 02_API_CONTRACT.md untuk modul ini sudah berfungsi dan diuji manual minimal sekali
- Filament Resource bisa dipakai untuk create, read, update, delete data utama modul ini
- Kode sudah mengikuti 04_CODE_STANDARDS.md, tanpa emoji atau karakter aneh di manapun dalam kode maupun komentar
- Tidak ada pemanggilan langsung antar service milik modul berbeda (lihat bagian 4)

## 8. Batasan yang Harus Diingat AI Agent

- Jangan membuat fitur integrasi OTA atau Channel Manager.
- Jangan mengganti tech stack yang sudah ditentukan di bagian 2.
- Jangan membuat tabel atau kolom yang tidak ada di 03_DATABASE_SCHEMA.md tanpa menambahkannya ke dokumen tersebut terlebih dahulu. Skema database adalah sumber kebenaran, kode mengikuti skema, bukan sebaliknya.
- Jangan membuat endpoint API yang tidak ada di 02_API_CONTRACT.md tanpa menambahkannya ke dokumen tersebut terlebih dahulu.
- Jangan menulis emoji, simbol dekoratif, atau karakter unicode aneh di dalam kode, nama variabel, nama file, commit message, atau komentar kode. Bahasa Indonesia baku boleh dipakai di komentar dan nama field yang memang berbahasa Indonesia sesuai skema (misal jenis_kamar), tapi penulisannya harus dengan karakter ASCII standar.

## 9. Alur Kerja Booking Engine React Terhadap PMS

Booking engine React adalah klien API biasa, sama seperti aplikasi mobile. Booking engine tidak boleh mengakses database secara langsung, dan tidak boleh punya kredensial admin. Semua interaksi lewat endpoint publik yang khusus dibuat untuk tamu, didefinisikan di bagian "Public Booking API" pada 02_API_CONTRACT.md.

Setiap hotel tenant punya identitas tenant tersendiri (lewat subdomain atau tenant key), dan booking engine wajib mengirim identitas tenant ini di setiap request supaya sistem tahu data hotel mana yang diakses.

## 10. Referensi Silang Dokumen

Saat mengerjakan modul apapun, AI agent wajib membuka dan menyamakan tiga dokumen berikut secara bersamaan:

- Struktur tabel dan nama kolom yang tepat: 03_DATABASE_SCHEMA.md
- Bentuk request dan response API yang tepat: 02_API_CONTRACT.md
- Cara penamaan file, class, folder, dan gaya penulisan kode: 04_CODE_STANDARDS.md

Jika ada perbedaan antara apa yang tertulis di masterplan ini dengan tiga dokumen tersebut soal detail teknis, dokumen yang lebih spesifik yang menang (misal soal nama kolom, ikuti 03_DATABASE_SCHEMA.md, bukan penjelasan singkat di masterplan ini).
