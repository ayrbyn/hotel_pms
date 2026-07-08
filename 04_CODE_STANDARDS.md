# CODE STANDARDS - Sistem Manajemen Hotel Multi Tenant

Dokumen ini adalah aturan wajib penulisan kode untuk AI agent. Tujuannya supaya kode yang dihasilkan konsisten, mudah dibaca, dan mudah dilanjutkan oleh sesi AI lain atau developer manusia di kemudian hari.

## 1. Aturan Paling Penting (Tidak Boleh Dilanggar)

- Jangan pernah menulis emoji di manapun: kode, komentar, commit message, nama file, nama variabel, pesan error, seeder, dokumentasi yang dihasilkan otomatis.
- Jangan pernah memakai karakter unicode dekoratif seperti tanda panah unik, bintang dekoratif, garis pemisah artistik, atau simbol serupa. Pakai karakter ASCII standar saja.
- Semua nama variabel, class, method, dan file memakai bahasa Inggris, kecuali untuk istilah domain yang memang berbahasa Indonesia dan sudah dipakai di 03_DATABASE_SCHEMA.md (contoh: kolom jenis_kamar jika suatu saat ada). Jika ragu, pakai bahasa Inggris.
- Setiap file yang dibuat harus langsung bisa dijalankan tanpa syntax error. Jangan meninggalkan kode setengah jadi dengan komentar TODO tanpa implementasi, kecuali memang diminta secara eksplisit oleh user.

## 2. Standar Backend (Laravel dan Filament)

### Penamaan
| Elemen | Aturan | Contoh |
|---|---|---|
| Nama class | PascalCase, kata benda tunggal | ReservationService, RoomController |
| Nama method dan variabel | camelCase | calculateTotalPrice, checkInDate |
| Nama tabel database | snake_case, jamak | reservations, room_types |
| Nama kolom database | snake_case | check_in_date, total_amount |
| Nama route | kebab-case | /purchase-orders, /room-types |
| Nama file migration | format Laravel standar | 2026_01_01_000000_create_reservations_table.php |

### Struktur Modul
Setiap modul di folder Modules/{NamaModul} wajib mengikuti struktur berikut, jangan menaruh file di luar strukturnya sendiri:

```
Modules/{NamaModul}/
  Database/
    Migrations/
    Seeders/
    Factories/
  Entities/                  (berisi Eloquent Model)
  Services/                  (logic bisnis, dipanggil dari controller atau Filament)
  Events/
  Listeners/
  Http/
    Controllers/
      Api/
    Requests/                (Form Request untuk validasi)
    Resources/                (API Resource untuk format response)
  Filament/
    Resources/
  Providers/
  routes/
    api.php
  Tests/
```

### Aturan Controller
- Controller hanya boleh berisi: menerima request, memanggil satu method dari Service class, mengembalikan response. Tidak boleh ada logic bisnis (perhitungan, kondisi bisnis, query kompleks) langsung di controller.
- Validasi input wajib memakai Form Request class, bukan validasi manual di dalam method controller.
- Response API wajib memakai API Resource class supaya format konsisten dengan 02_API_CONTRACT.md, bukan mengembalikan model secara langsung dengan toArray.

Contoh struktur controller yang benar:
```php
class ReservationController extends Controller
{
    public function __construct(
        private readonly ReservationService $reservationService
    ) {
    }

    public function store(StoreReservationRequest $request): JsonResponse
    {
        $reservation = $this->reservationService->create($request->validated());

        return (new ReservationResource($reservation))
            ->response()
            ->setStatusCode(201);
    }
}
```

### Aturan Service Class
- Satu Service class fokus pada satu domain (contoh: ReservationService, tidak digabung dengan HousekeepingService).
- Method di Service class harus punya nama yang jelas menunjukkan aksi bisnis, bukan aksi teknis. Contoh yang benar: checkInGuest, bukan updateStatus.
- Perubahan data yang perlu diketahui modul lain wajib memicu Event, jangan memanggil Service class modul lain secara langsung. Lihat bagian 4 dokumen 01_MASTERPLAN.md soal aturan ini.

### Aturan Model (Entities)
- Semua model wajib memakai trait UUID7 yang sudah didefinisikan di app/Traits/HasUuid7.php.
- Semua model milik tenant wajib memakai Global Scope tenant otomatis dari package stancl/tenancy, jangan menulis where tenant_id manual di setiap query.
- Relasi antar model (hasMany, belongsTo, dan seterusnya) wajib didefinisikan dengan nama method yang sesuai nama relasi, bukan nama tabel. Contoh: reservation()->belongsTo(Guest::class) memakai method guest(), bukan getGuest().
- Cast tipe data wajib didefinisikan eksplisit di property $casts, khususnya untuk kolom tanggal, uang (integer), dan jsonb.

### Aturan Filament Resource
- Satu Filament Resource untuk satu Entity utama modul.
- Form dan Table di Filament Resource tidak boleh mengandung logic bisnis kompleks, cukup mapping field ke komponen form. Logic bisnis tetap ada di Service class, dipanggil lewat action button jika diperlukan.
- Label yang tampil di panel Filament boleh berbahasa Indonesia (karena dipakai staf hotel), tapi kode di baliknya (nama field, nama variabel) tetap bahasa Inggris.

### Aturan Migration
- Satu migration untuk satu tabel saja, jangan menggabungkan pembuatan beberapa tabel tidak berelasi dalam satu file migration.
- Kolom foreign key wajib didefinisikan dengan constraint eksplisit (foreignUuid, constrained, onDelete yang sesuai konteks bisnis, misal restrict untuk data finansial, cascade untuk data anak yang memang harus ikut terhapus).
- Jangan pernah membuat migration yang mengubah data (data seeding) di dalam file migration. Data contoh wajib lewat Seeder, bukan lewat migration.

### Aturan Testing
- Setiap modul minimal punya Feature Test untuk alur utama (contoh: modul Reservation wajib ada test untuk create reservation, check-in, check-out).
- Nama method test wajib deskriptif dalam bahasa Inggris, format snake_case dengan awalan test atau memakai atribut #[Test]. Contoh: it_creates_a_reservation_with_pending_status.
- Test tidak boleh bergantung pada urutan eksekusi test lain. Setiap test harus bisa dijalankan sendirian.

## 3. Standar Frontend (React untuk Booking Engine)

### Struktur Folder
```
src/
  components/
    common/            (button, input, modal, dan komponen umum lain)
    booking/            (komponen khusus alur booking)
  pages/
    HomePage.jsx
    RoomListPage.jsx
    BookingFormPage.jsx
    BookingStatusPage.jsx
  services/
    apiClient.js        (satu file konfigurasi axios atau fetch wrapper)
    reservationService.js
    roomService.js
  hooks/
  context/
    TenantContext.jsx    (menyimpan identitas tenant dari subdomain)
  utils/
    formatCurrency.js
    formatDate.js
```

### Penamaan
| Elemen | Aturan | Contoh |
|---|---|---|
| Nama komponen | PascalCase | RoomCard, BookingForm |
| Nama file komponen | sama dengan nama komponen | RoomCard.jsx |
| Nama function biasa dan variabel | camelCase | fetchAvailability, selectedRoomType |
| Nama folder | camelCase atau lowercase | booking, services |

### Aturan Komunikasi API
- Semua pemanggilan API wajib lewat file di folder services/, komponen tidak boleh memanggil fetch atau axios secara langsung.
- Base URL API dan tenant key wajib diambil dari environment variable, jangan hardcode di dalam kode.
- Semua response error dari API wajib ditangani dan ditampilkan ke user dengan pesan yang jelas, jangan biarkan error tanpa penanganan.

### Aturan Komponen
- Komponen sebaiknya kecil dan fokus satu tanggung jawab. Jika satu file komponen sudah melebihi kira kira 200 baris, pertimbangkan untuk memecahnya.
- State yang dipakai banyak halaman (misal identitas tenant, data booking yang sedang berjalan) memakai React Context, bukan disebar lewat props berlapis lapis.
- Style memakai pendekatan yang konsisten di seluruh proyek (contoh: satu pilihan antara CSS Modules atau Tailwind), jangan mencampur banyak pendekatan styling berbeda dalam satu proyek yang sama.

## 4. Standar Commit dan Dokumentasi Kode

### Format commit message
Pakai format berikut, dalam bahasa Inggris, tanpa emoji:
```
type: penjelasan singkat

Contoh:
feat: add check-in endpoint for reservations
fix: correct total price calculation on folio
refactor: extract journal posting logic to service class
test: add feature test for pos transaction split
```

Jenis type yang dipakai: feat, fix, refactor, test, docs, chore.

### Komentar Kode
- Komentar dipakai untuk menjelaskan alasan (why), bukan mengulang apa yang sudah jelas dari kode (what). Contoh yang tidak perlu: `// increment counter` di atas `$counter++;`.
- Komentar boleh berbahasa Indonesia atau Inggris, konsisten dalam satu file, tapi tidak boleh memakai emoji atau karakter dekoratif apapun.

## 5. Checklist Sebelum Dianggap Selesai

Sebelum AI agent melaporkan satu bagian pekerjaan selesai, pastikan:

- Tidak ada emoji atau karakter unicode aneh di file manapun yang dibuat atau diubah
- Semua nama mengikuti aturan penamaan di dokumen ini
- Tidak ada logic bisnis yang nyasar di controller atau di komponen React
- Semua endpoint yang dibuat sudah sesuai dengan 02_API_CONTRACT.md
- Semua tabel dan kolom yang dipakai sudah sesuai dengan 03_DATABASE_SCHEMA.md
- Kode sudah dicoba jalan minimal sekali, bukan hanya ditulis tanpa diuji
