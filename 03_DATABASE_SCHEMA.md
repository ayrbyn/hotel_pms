# DATABASE SCHEMA - Sistem Manajemen Hotel Multi Tenant

Dokumen ini adalah sumber kebenaran untuk struktur database. Kode, migration, dan model wajib mengikuti dokumen ini. Jika ada kebutuhan kolom atau tabel baru saat development, dokumen ini harus diperbarui dulu sebelum migration ditulis.

Database yang dipakai: PostgreSQL.

## 1. Aturan Umum Semua Tabel

- Primary key bernama `id`, bertipe UUID, diisi dengan UUID versi 7 dari aplikasi (bukan default database).
- Semua tabel milik tenant (hampir semua tabel operasional) wajib punya kolom `tenant_id` bertipe UUID, dengan foreign key ke tabel `tenants`.
- Semua tabel punya `created_at` dan `updated_at`.
- Tabel yang menyimpan data transaksi atau data penting yang tidak boleh hilang permanen wajib punya `deleted_at` (soft delete). Ditandai dengan (soft delete) di tiap tabel di bawah.
- Nilai uang disimpan sebagai kolom bertipe bigint, satuan rupiah terkecil, bukan decimal atau float.
- Nama tabel memakai snake_case, bentuk jamak. Contoh: rooms, reservations, purchase_orders.
- Nama kolom foreign key memakai format `nama_singular_id`. Contoh: room_id, guest_id.
- Index wajib dibuat untuk semua foreign key dan kolom yang sering dipakai untuk filter (misal status, tanggal).

## 2. Modul Tenancy (Central, bukan milik tenant)

Tabel di bagian ini tidak punya kolom tenant_id, karena ini adalah tabel milik sistem pusat.

### tenants
| Kolom | Tipe | Keterangan |
|---|---|---|
| id | uuid | primary key |
| name | varchar | nama hotel |
| slug | varchar, unique | dipakai untuk subdomain |
| subscription_plan | varchar | contoh: basic, pro, enterprise |
| subscription_status | varchar | active, suspended, trial, cancelled |
| trial_ends_at | timestamp, nullable | |
| database_config | jsonb, nullable | disiapkan untuk kebutuhan isolasi lanjutan di masa depan, tidak dipakai di fase awal |
| created_at | timestamp | |
| updated_at | timestamp | |

### tenant_domains
| Kolom | Tipe | Keterangan |
|---|---|---|
| id | uuid | primary key |
| tenant_id | uuid | foreign key ke tenants |
| domain | varchar, unique | subdomain atau custom domain |
| created_at | timestamp | |
| updated_at | timestamp | |

### users
| Kolom | Tipe | Keterangan |
|---|---|---|
| id | uuid | primary key |
| tenant_id | uuid, nullable | null jika user adalah admin pusat |
| name | varchar | |
| email | varchar, unique per tenant | |
| password | varchar | |
| role | varchar | dikelola juga lewat spatie/laravel-permission |
| is_active | boolean, default true | |
| created_at | timestamp | |
| updated_at | timestamp | |
| deleted_at | timestamp, nullable | (soft delete) |

## 3. Modul Front Office / Reservation

### room_types
| Kolom | Tipe | Keterangan |
|---|---|---|
| id | uuid | primary key |
| tenant_id | uuid | |
| name | varchar | contoh: Deluxe, Superior |
| description | text, nullable | |
| base_price | bigint | harga dasar per malam |
| max_occupancy | integer | |
| created_at | timestamp | |
| updated_at | timestamp | |
| deleted_at | timestamp, nullable | (soft delete) |

### rooms
| Kolom | Tipe | Keterangan |
|---|---|---|
| id | uuid | primary key |
| tenant_id | uuid | |
| room_type_id | uuid | foreign key ke room_types |
| room_number | varchar | |
| floor | varchar, nullable | |
| status | varchar | available, occupied, maintenance |
| created_at | timestamp | |
| updated_at | timestamp | |
| deleted_at | timestamp, nullable | (soft delete) |

Catatan: status kebersihan kamar (clean, dirty, vacant) TIDAK disimpan di tabel ini, itu tanggung jawab modul Housekeeping lewat tabel room_housekeeping_statuses, supaya batas modul tetap jelas.

### rate_plans
| Kolom | Tipe | Keterangan |
|---|---|---|
| id | uuid | primary key |
| tenant_id | uuid | |
| room_type_id | uuid | foreign key ke room_types |
| name | varchar | contoh: Weekday Rate, Weekend Rate |
| price | bigint | |
| valid_from | date, nullable | |
| valid_until | date, nullable | |
| created_at | timestamp | |
| updated_at | timestamp | |

### guests
| Kolom | Tipe | Keterangan |
|---|---|---|
| id | uuid | primary key |
| tenant_id | uuid | |
| full_name | varchar | |
| email | varchar, nullable | |
| phone | varchar, nullable | |
| id_number | varchar, nullable | nomor KTP atau paspor |
| address | text, nullable | |
| created_at | timestamp | |
| updated_at | timestamp | |
| deleted_at | timestamp, nullable | (soft delete) |

### reservations
| Kolom | Tipe | Keterangan |
|---|---|---|
| id | uuid | primary key |
| tenant_id | uuid | |
| guest_id | uuid | foreign key ke guests |
| room_id | uuid, nullable | diisi saat sudah dialokasikan kamar |
| room_type_id | uuid | tipe kamar yang dipesan |
| check_in_date | date | |
| check_out_date | date | |
| actual_check_in_at | timestamp, nullable | |
| actual_check_out_at | timestamp, nullable | |
| status | varchar | pending, confirmed, checked_in, checked_out, cancelled, no_show |
| source | varchar | walk_in, booking_engine, admin |
| total_price | bigint | |
| notes | text, nullable | |
| created_at | timestamp | |
| updated_at | timestamp | |
| deleted_at | timestamp, nullable | (soft delete) |

### folios
Folio adalah tagihan induk milik satu reservasi, tempat semua item tagihan (kamar, POS, layanan lain) dikumpulkan.

| Kolom | Tipe | Keterangan |
|---|---|---|
| id | uuid | primary key |
| tenant_id | uuid | |
| reservation_id | uuid | foreign key ke reservations |
| status | varchar | open, closed |
| total_amount | bigint | dihitung dari folio_items |
| created_at | timestamp | |
| updated_at | timestamp | |

### folio_items
| Kolom | Tipe | Keterangan |
|---|---|---|
| id | uuid | primary key |
| tenant_id | uuid | |
| folio_id | uuid | foreign key ke folios |
| source_type | varchar | room_charge, pos, service_charge, tax, discount |
| source_id | uuid, nullable | id transaksi asal, misal id dari pos_transactions |
| description | varchar | |
| amount | bigint | |
| created_at | timestamp | |
| updated_at | timestamp | |

## 4. Modul Housekeeping

### room_housekeeping_statuses
| Kolom | Tipe | Keterangan |
|---|---|---|
| id | uuid | primary key |
| tenant_id | uuid | |
| room_id | uuid | foreign key ke rooms, unique per room |
| status | varchar | clean, dirty, vacant, out_of_order |
| updated_by | uuid, nullable | foreign key ke users |
| created_at | timestamp | |
| updated_at | timestamp | |

### cleaning_tasks
| Kolom | Tipe | Keterangan |
|---|---|---|
| id | uuid | primary key |
| tenant_id | uuid | |
| room_id | uuid | foreign key ke rooms |
| assigned_to | uuid, nullable | foreign key ke users |
| scheduled_date | date | |
| status | varchar | pending, in_progress, done |
| created_at | timestamp | |
| updated_at | timestamp | |

### laundry_requests
| Kolom | Tipe | Keterangan |
|---|---|---|
| id | uuid | primary key |
| tenant_id | uuid | |
| room_id | uuid | foreign key ke rooms |
| requested_by | uuid, nullable | foreign key ke users |
| item_description | text | |
| status | varchar | requested, in_process, completed |
| created_at | timestamp | |
| updated_at | timestamp | |

## 5. Modul Point of Sale

### pos_outlets
| Kolom | Tipe | Keterangan |
|---|---|---|
| id | uuid | primary key |
| tenant_id | uuid | |
| name | varchar | contoh: Restoran Utama, Cafe Lobby |
| type | varchar | restaurant, cafe, minibar, room_service |
| created_at | timestamp | |
| updated_at | timestamp | |

### pos_products
| Kolom | Tipe | Keterangan |
|---|---|---|
| id | uuid | primary key |
| tenant_id | uuid | |
| pos_outlet_id | uuid | foreign key ke pos_outlets |
| name | varchar | |
| price | bigint | |
| is_active | boolean, default true | |
| created_at | timestamp | |
| updated_at | timestamp | |

### pos_transactions
| Kolom | Tipe | Keterangan |
|---|---|---|
| id | uuid | primary key |
| tenant_id | uuid | |
| pos_outlet_id | uuid | foreign key ke pos_outlets |
| reservation_id | uuid, nullable | diisi jika transaksi dibebankan ke kamar |
| cashier_id | uuid | foreign key ke users |
| status | varchar | open, paid, void |
| payment_method | varchar, nullable | cash, card, charge_to_room |
| total_amount | bigint | |
| created_at | timestamp | |
| updated_at | timestamp | |
| deleted_at | timestamp, nullable | (soft delete) |

### pos_transaction_items
| Kolom | Tipe | Keterangan |
|---|---|---|
| id | uuid | primary key |
| tenant_id | uuid | |
| pos_transaction_id | uuid | foreign key ke pos_transactions |
| pos_product_id | uuid | foreign key ke pos_products |
| quantity | integer | |
| unit_price | bigint | |
| subtotal | bigint | |
| created_at | timestamp | |
| updated_at | timestamp | |

## 6. Modul Inventory

### warehouses
| Kolom | Tipe | Keterangan |
|---|---|---|
| id | uuid | primary key |
| tenant_id | uuid | |
| name | varchar | contoh: Gudang Utama, Gudang F&B |
| created_at | timestamp | |
| updated_at | timestamp | |

### inventory_items
| Kolom | Tipe | Keterangan |
|---|---|---|
| id | uuid | primary key |
| tenant_id | uuid | |
| name | varchar | |
| category | varchar | food, room_supply, amenity, other |
| unit | varchar | contoh: pcs, kg, liter |
| minimum_stock | integer, default 0 | |
| created_at | timestamp | |
| updated_at | timestamp | |
| deleted_at | timestamp, nullable | (soft delete) |

### inventory_stocks
| Kolom | Tipe | Keterangan |
|---|---|---|
| id | uuid | primary key |
| tenant_id | uuid | |
| warehouse_id | uuid | foreign key ke warehouses |
| inventory_item_id | uuid | foreign key ke inventory_items |
| quantity | integer, default 0 | |
| created_at | timestamp | |
| updated_at | timestamp | |

Catatan: kombinasi warehouse_id dan inventory_item_id harus unique.

### stock_movements
| Kolom | Tipe | Keterangan |
|---|---|---|
| id | uuid | primary key |
| tenant_id | uuid | |
| warehouse_id | uuid | foreign key ke warehouses |
| inventory_item_id | uuid | foreign key ke inventory_items |
| type | varchar | in, out, adjustment |
| quantity | integer | |
| reference_type | varchar, nullable | purchase_order, pos_transaction, manual |
| reference_id | uuid, nullable | |
| created_at | timestamp | |
| updated_at | timestamp | |

## 7. Modul Purchasing

### suppliers
| Kolom | Tipe | Keterangan |
|---|---|---|
| id | uuid | primary key |
| tenant_id | uuid | |
| name | varchar | |
| phone | varchar, nullable | |
| email | varchar, nullable | |
| address | text, nullable | |
| created_at | timestamp | |
| updated_at | timestamp | |
| deleted_at | timestamp, nullable | (soft delete) |

### purchase_requests
| Kolom | Tipe | Keterangan |
|---|---|---|
| id | uuid | primary key |
| tenant_id | uuid | |
| requested_by | uuid | foreign key ke users |
| status | varchar | draft, submitted, approved, rejected |
| notes | text, nullable | |
| created_at | timestamp | |
| updated_at | timestamp | |

### purchase_request_items
| Kolom | Tipe | Keterangan |
|---|---|---|
| id | uuid | primary key |
| tenant_id | uuid | |
| purchase_request_id | uuid | foreign key ke purchase_requests |
| inventory_item_id | uuid | foreign key ke inventory_items |
| quantity | integer | |
| created_at | timestamp | |
| updated_at | timestamp | |

### purchase_orders
| Kolom | Tipe | Keterangan |
|---|---|---|
| id | uuid | primary key |
| tenant_id | uuid | |
| purchase_request_id | uuid, nullable | foreign key ke purchase_requests |
| supplier_id | uuid | foreign key ke suppliers |
| status | varchar | draft, sent, received, cancelled |
| total_amount | bigint | |
| created_at | timestamp | |
| updated_at | timestamp | |

### purchase_order_items
| Kolom | Tipe | Keterangan |
|---|---|---|
| id | uuid | primary key |
| tenant_id | uuid | |
| purchase_order_id | uuid | foreign key ke purchase_orders |
| inventory_item_id | uuid | foreign key ke inventory_items |
| quantity | integer | |
| unit_price | bigint | |
| subtotal | bigint | |
| created_at | timestamp | |
| updated_at | timestamp | |

## 8. Modul Accounting and Finance

### chart_of_accounts
| Kolom | Tipe | Keterangan |
|---|---|---|
| id | uuid | primary key |
| tenant_id | uuid | |
| code | varchar | contoh: 1000, 4000 |
| name | varchar | contoh: Kas, Pendapatan Kamar |
| type | varchar | asset, liability, equity, revenue, expense |
| created_at | timestamp | |
| updated_at | timestamp | |

### journal_entries
| Kolom | Tipe | Keterangan |
|---|---|---|
| id | uuid | primary key |
| tenant_id | uuid | |
| entry_date | date | |
| description | varchar | |
| reference_type | varchar, nullable | pos_transaction, purchase_order, manual |
| reference_id | uuid, nullable | |
| created_at | timestamp | |
| updated_at | timestamp | |

### journal_entry_lines
| Kolom | Tipe | Keterangan |
|---|---|---|
| id | uuid | primary key |
| tenant_id | uuid | |
| journal_entry_id | uuid | foreign key ke journal_entries |
| chart_of_account_id | uuid | foreign key ke chart_of_accounts |
| debit | bigint, default 0 | |
| credit | bigint, default 0 | |
| created_at | timestamp | |
| updated_at | timestamp | |

### invoices
| Kolom | Tipe | Keterangan |
|---|---|---|
| id | uuid | primary key |
| tenant_id | uuid | |
| folio_id | uuid, nullable | foreign key ke folios, untuk invoice tamu |
| supplier_id | uuid, nullable | untuk invoice hutang ke supplier |
| type | varchar | receivable, payable |
| status | varchar | unpaid, partially_paid, paid, overdue |
| total_amount | bigint | |
| due_date | date, nullable | |
| created_at | timestamp | |
| updated_at | timestamp | |

## 9. Modul CRM

### crm_members
| Kolom | Tipe | Keterangan |
|---|---|---|
| id | uuid | primary key |
| tenant_id | uuid | |
| guest_id | uuid | foreign key ke guests |
| membership_number | varchar, unique per tenant | |
| tier | varchar | contoh: silver, gold, platinum |
| points_balance | integer, default 0 | |
| created_at | timestamp | |
| updated_at | timestamp | |

### crm_promos
| Kolom | Tipe | Keterangan |
|---|---|---|
| id | uuid | primary key |
| tenant_id | uuid | |
| name | varchar | |
| description | text, nullable | |
| discount_type | varchar | percentage, fixed |
| discount_value | integer | |
| valid_from | date | |
| valid_until | date | |
| created_at | timestamp | |
| updated_at | timestamp | |

### crm_feedbacks
| Kolom | Tipe | Keterangan |
|---|---|---|
| id | uuid | primary key |
| tenant_id | uuid | |
| guest_id | uuid | foreign key ke guests |
| reservation_id | uuid, nullable | foreign key ke reservations |
| rating | integer | angka 1 sampai 5 |
| comment | text, nullable | |
| created_at | timestamp | |
| updated_at | timestamp | |

## 10. Modul Reporting

Modul ini tidak punya tabel sendiri untuk data mentah. Semua data diambil dari modul lain, lalu diagregasi. Boleh menambahkan tabel cache berikut untuk mempercepat dashboard:

### report_snapshot_cache
| Kolom | Tipe | Keterangan |
|---|---|---|
| id | uuid | primary key |
| tenant_id | uuid | |
| report_key | varchar | contoh: daily_occupancy, monthly_revenue |
| period_date | date | |
| payload | jsonb | hasil agregasi siap tampil |
| generated_at | timestamp | |

Tabel ini diisi ulang lewat scheduled job, bukan dihitung langsung tiap request dashboard dibuka.

## 11. Diagram Relasi Ringkas (Level Modul)

```
tenants 1---N users
tenants 1---N room_types 1---N rooms
rooms 1---1 room_housekeeping_statuses
guests 1---N reservations
room_types 1---N reservations
reservations 1---1 folios 1---N folio_items
pos_outlets 1---N pos_transactions 1---N pos_transaction_items
pos_transactions N---1 reservations (nullable, saat charge to room)
warehouses 1---N inventory_stocks
inventory_items 1---N inventory_stocks
purchase_requests 1---N purchase_request_items
purchase_orders 1---N purchase_order_items
purchase_orders N---1 suppliers
journal_entries 1---N journal_entry_lines
journal_entry_lines N---1 chart_of_accounts
guests 1---1 crm_members
```
