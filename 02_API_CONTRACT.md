# API CONTRACT - Sistem Manajemen Hotel Multi Tenant

Dokumen ini mendefinisikan seluruh endpoint API yang boleh dibuat pada fase awal pengembangan. AI agent tidak boleh membuat endpoint di luar dokumen ini tanpa menambahkannya ke sini terlebih dahulu. Nama field response wajib sama persis dengan nama kolom di 03_DATABASE_SCHEMA.md kecuali disebutkan ada transformasi.

## 1. Aturan Umum

### Format dasar
- Base URL internal (Tenant Panel API, dipakai untuk kebutuhan internal seperti mobile app staf di masa depan): `/api/v1/...`
- Base URL publik (dipakai booking engine React): `/api/v1/public/...`
- Semua request dan response memakai format JSON.
- Semua request wajib memakai header `Accept: application/json`.

### Autentikasi
- Endpoint internal (bukan public) memakai Laravel Sanctum, token dikirim lewat header `Authorization: Bearer {token}`.
- Endpoint public tidak memakai token user, tapi wajib mengirim header `X-Tenant-Key: {tenant_slug}` supaya sistem tahu data hotel mana yang diakses.
- Endpoint login menghasilkan token, dijelaskan di bagian 3.

### Format response sukses
```
{
  "data": { ... atau [ ... ] },
  "meta": { ... opsional, dipakai untuk pagination ... }
}
```

### Format response error
```
{
  "message": "penjelasan singkat error",
  "errors": {
    "nama_field": ["pesan validasi"]
  }
}
```

### Format pagination
Semua endpoint list data wajib mendukung pagination dengan query parameter `page` dan `per_page`. Response memakai format:
```
{
  "data": [ ... ],
  "meta": {
    "current_page": 1,
    "per_page": 20,
    "total": 134
  }
}
```

### Kode status HTTP yang dipakai
| Kode | Arti |
|---|---|
| 200 | berhasil, data dikembalikan |
| 201 | berhasil membuat data baru |
| 204 | berhasil, tidak ada isi dikembalikan (misal delete) |
| 401 | belum login atau token salah |
| 403 | login benar tapi tidak punya akses |
| 404 | data tidak ditemukan |
| 422 | validasi gagal |
| 500 | error di server |

## 2. Penamaan Endpoint

Pola URL memakai kebab-case dan bentuk jamak untuk resource. Contoh: `/api/v1/room-types`, bukan `/api/v1/roomType`.

Aksi standar CRUD:
| Method | URL | Fungsi |
|---|---|---|
| GET | /resource | daftar data, dengan pagination |
| GET | /resource/{id} | detail satu data |
| POST | /resource | buat data baru |
| PUT | /resource/{id} | ubah seluruh data |
| PATCH | /resource/{id} | ubah sebagian data |
| DELETE | /resource/{id} | hapus data (soft delete jika tabel mendukung) |

## 3. Autentikasi (Internal, untuk staf hotel)

### POST /api/v1/auth/login
Request:
```
{
  "email": "staf@contohhotel.com",
  "password": "rahasia123"
}
```
Response 200:
```
{
  "data": {
    "token": "isi-token-sanctum",
    "user": {
      "id": "uuid",
      "name": "Nama Staf",
      "email": "staf@contohhotel.com",
      "role": "front_office"
    }
  }
}
```

### POST /api/v1/auth/logout
Header wajib token. Response 204, tanpa isi.

### GET /api/v1/auth/me
Header wajib token. Response 200 berisi data user yang sedang login.

## 4. Modul Front Office / Reservation (Internal)

### GET /api/v1/rooms
Query parameter opsional: status, room_type_id.
Response data per item:
```
{
  "id": "uuid",
  "room_number": "101",
  "floor": "1",
  "status": "available",
  "room_type": {
    "id": "uuid",
    "name": "Deluxe"
  }
}
```

### GET /api/v1/reservations
Query parameter opsional: status, check_in_date, check_out_date, guest_id.

### POST /api/v1/reservations
Request:
```
{
  "guest_id": "uuid",
  "room_type_id": "uuid",
  "check_in_date": "2026-08-01",
  "check_out_date": "2026-08-03",
  "notes": "opsional"
}
```
Response 201 berisi data reservasi lengkap dengan status pending.

### POST /api/v1/reservations/{id}/check-in
Request:
```
{
  "room_id": "uuid"
}
```
Mengubah status reservasi menjadi checked_in, mengisi actual_check_in_at, dan membuat folio baru untuk reservasi ini.

### POST /api/v1/reservations/{id}/check-out
Tanpa body. Mengubah status reservasi menjadi checked_out, mengisi actual_check_out_at, menutup folio (status closed).

### GET /api/v1/reservations/{id}/folio
Response berisi data folio beserta seluruh folio_items dan total_amount.

## 5. Modul Housekeeping (Internal)

### GET /api/v1/housekeeping/room-statuses
Response daftar status kebersihan seluruh kamar.

### PATCH /api/v1/housekeeping/room-statuses/{room_id}
Request:
```
{
  "status": "clean"
}
```

### GET /api/v1/housekeeping/cleaning-tasks
Query parameter opsional: status, scheduled_date, assigned_to.

### POST /api/v1/housekeeping/cleaning-tasks
Request:
```
{
  "room_id": "uuid",
  "assigned_to": "uuid, opsional",
  "scheduled_date": "2026-08-01"
}
```

### POST /api/v1/housekeeping/laundry-requests
Request:
```
{
  "room_id": "uuid",
  "item_description": "2 handuk, 1 sprei"
}
```

## 6. Modul Point of Sale (Internal)

### GET /api/v1/pos/products
Query parameter opsional: pos_outlet_id.

### POST /api/v1/pos/transactions
Request:
```
{
  "pos_outlet_id": "uuid",
  "reservation_id": "uuid, opsional jika dibebankan ke kamar",
  "items": [
    { "pos_product_id": "uuid", "quantity": 2 }
  ]
}
```
Response 201 berisi transaksi dengan status open dan total_amount terhitung.

### POST /api/v1/pos/transactions/{id}/pay
Request:
```
{
  "payment_method": "cash"
}
```
Jika payment_method adalah charge_to_room, sistem otomatis menambahkan item ke folio milik reservation_id terkait, dan memicu event yang nantinya didengar modul Accounting.

### POST /api/v1/pos/transactions/{id}/split
Request:
```
{
  "splits": [
    { "item_ids": ["uuid1", "uuid2"], "payment_method": "cash" },
    { "item_ids": ["uuid3"], "payment_method": "card" }
  ]
}
```

## 7. Modul Inventory (Internal)

### GET /api/v1/inventory/items
Query parameter opsional: category, warehouse_id, low_stock (boolean, filter item di bawah minimum_stock).

### GET /api/v1/inventory/stocks
Query parameter opsional: warehouse_id, inventory_item_id.

### POST /api/v1/inventory/stock-movements
Request:
```
{
  "warehouse_id": "uuid",
  "inventory_item_id": "uuid",
  "type": "adjustment",
  "quantity": -5
}
```
Nilai quantity boleh negatif khusus untuk type adjustment atau out.

## 8. Modul Purchasing (Internal)

### POST /api/v1/purchasing/purchase-requests
Request:
```
{
  "items": [
    { "inventory_item_id": "uuid", "quantity": 10 }
  ],
  "notes": "opsional"
}
```

### POST /api/v1/purchasing/purchase-requests/{id}/approve
Tanpa body. Mengubah status menjadi approved. Hanya bisa diakses role tertentu, diatur lewat spatie/laravel-permission, bukan logic manual di controller.

### POST /api/v1/purchasing/purchase-orders
Request:
```
{
  "purchase_request_id": "uuid, opsional",
  "supplier_id": "uuid",
  "items": [
    { "inventory_item_id": "uuid", "quantity": 10, "unit_price": 15000 }
  ]
}
```

### POST /api/v1/purchasing/purchase-orders/{id}/receive
Tanpa body. Mengubah status menjadi received, dan memicu stock_movements type in secara otomatis untuk tiap item.

## 9. Modul Accounting and Finance (Internal)

### GET /api/v1/accounting/journal-entries
Query parameter opsional: entry_date_from, entry_date_to, reference_type.

### GET /api/v1/accounting/reports/profit-loss
Query parameter wajib: date_from, date_until.

### GET /api/v1/accounting/reports/cash-flow
Query parameter wajib: date_from, date_until.

### GET /api/v1/accounting/invoices
Query parameter opsional: type, status.

## 10. Modul CRM (Internal)

### GET /api/v1/crm/members
Query parameter opsional: tier.

### POST /api/v1/crm/members
Request:
```
{
  "guest_id": "uuid",
  "tier": "silver"
}
```

### GET /api/v1/crm/promos

### POST /api/v1/crm/feedbacks
Request:
```
{
  "guest_id": "uuid",
  "reservation_id": "uuid, opsional",
  "rating": 5,
  "comment": "opsional"
}
```

## 11. Modul Reporting (Internal)

### GET /api/v1/reports/dashboard
Response ringkasan gabungan untuk halaman utama dashboard manajemen, contoh:
```
{
  "data": {
    "occupancy_rate_today": 78.5,
    "revenue_today": 45000000,
    "revenue_month_to_date": 980000000,
    "top_selling_rooms": [
      { "room_type": "Deluxe", "total_booked": 120 }
    ],
    "restaurant_performance_today": {
      "total_transactions": 34,
      "total_revenue": 6200000
    }
  }
}
```

### GET /api/v1/reports/occupancy
Query parameter wajib: date_from, date_until.

### GET /api/v1/reports/revenue
Query parameter wajib: date_from, date_until, group_by (day, week, month).

## 12. Public Booking API (dipakai booking engine React)

Semua endpoint di bagian ini wajib pakai prefix `/api/v1/public/` dan header `X-Tenant-Key`. Endpoint ini tidak memerlukan login user, tapi tetap dibatasi rate limit di level middleware supaya tidak disalahgunakan.

### GET /api/v1/public/hotel-info
Response berisi informasi umum hotel milik tenant terkait: nama, alamat, deskripsi, daftar fasilitas.

### GET /api/v1/public/room-types
Response daftar tipe kamar beserta harga dasar dan deskripsi, dipakai untuk halaman katalog kamar di booking engine.

### GET /api/v1/public/availability
Query parameter wajib: check_in_date, check_out_date.
Response daftar room_type yang masih tersedia pada rentang tanggal tersebut, beserta sisa jumlah kamar dan harga.

### POST /api/v1/public/bookings
Request:
```
{
  "room_type_id": "uuid",
  "check_in_date": "2026-08-01",
  "check_out_date": "2026-08-03",
  "guest": {
    "full_name": "Nama Tamu",
    "email": "tamu@email.com",
    "phone": "08123456789"
  }
}
```
Sistem membuat data guest baru jika email belum terdaftar, lalu membuat reservation dengan status pending dan source booking_engine.

Response 201:
```
{
  "data": {
    "reservation_id": "uuid",
    "status": "pending",
    "total_price": 1200000,
    "check_in_date": "2026-08-01",
    "check_out_date": "2026-08-03"
  }
}
```

### GET /api/v1/public/bookings/{reservation_id}
Dipakai booking engine untuk menampilkan status booking ke tamu. Wajib disertai parameter tambahan `email` sebagai verifikasi sederhana, contoh: `/api/v1/public/bookings/{reservation_id}?email=tamu@email.com`.

## 13. Yang Belum Dikerjakan (Sengaja Ditunda)

Endpoint berikut sengaja belum didefinisikan dan tidak boleh dibuat sampai ada instruksi lanjutan:
- Endpoint pembayaran online (payment gateway) untuk booking engine
- Endpoint integrasi Channel Manager dan OTA
- Endpoint webhook dari pihak ketiga manapun
