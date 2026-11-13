# ERD — Rusuh Booking Tiket

> Skema PostgreSQL (database tunggal `rusuh_booking`) persis seperti hasil migrasi di repo masing-masing service.
> Antrian dan token **tidak** disimpan di PostgreSQL, tapi di Redis (lihat [LLD.md §2.1](LLD.md#21-queue-service-8084)).

## Diagram

```mermaid
erDiagram
    APP_USER ||--o{ MEMBER_NIK : "menyimpan"
    APP_USER |o..o{ BOOKING : "user_id (opsional, logis)"
    EVENT ||--o{ TICKET_CATEGORY : "punya"
    EVENT ||..o{ BOOKING : "event_id (logis)"
    BOOKING ||--|{ BOOKING_ITEM : "berisi 1-3 tiket"
    TICKET_CATEGORY ||..o{ BOOKING_ITEM : "category_id (logis)"
    BOOKING ||--o{ PAYMENT : "dibayar via"
    BOOKING ||..o{ PAYMENT_CALLBACK_LOG : "booking_id (logis)"
    RESIDENTS ||..o{ BOOKING_ITEM : "nik dicocokkan saat booking"
    RESIDENTS ||..o{ MEMBER_NIK : "nik dicocokkan saat simpan"

    APP_USER {
        uuid id PK
        text email UK
        text password_hash "bcrypt"
        text role "customer, admin"
        timestamptz created_at
    }
    MEMBER_NIK {
        uuid id PK
        uuid user_id FK "ON DELETE CASCADE"
        char16 nik "UNIQUE (user_id, nik)"
        text nama
        date tgl_lahir
        text nama_ibu
        timestamptz created_at
    }
    RESIDENTS {
        char16 nik PK "data penduduk tiruan (dukcapil-mock)"
        text nama
        date tgl_lahir
        text nama_ibu
    }
    EVENT {
        uuid id PK
        text name "UNIQUE lower(name)"
        timestamptz sale_opens_at
        text status "draft, on_sale, sold_out, closed"
        text poster_url
        text description
        timestamptz created_at
    }
    TICKET_CATEGORY {
        uuid id PK
        uuid event_id FK "ON DELETE CASCADE"
        text name
        bigint price "rupiah"
        int capacity
        int available "CHECK 0 <= available <= capacity"
    }
    BOOKING {
        uuid id PK "= order_id Midtrans"
        uuid user_id "NULL untuk guest"
        uuid event_id
        text email "tujuan e-tiket"
        text status "HELD, PAID, EXPIRED, CANCELLED"
        timestamptz booking_expires_at "batas Access Token"
        timestamptz payment_expires_at "batas bayar (10 menit)"
        timestamptz created_at
    }
    BOOKING_ITEM {
        uuid id PK
        uuid booking_id FK "ON DELETE CASCADE"
        uuid category_id
        char16 nik "1 tiket = 1 NIK penonton"
        bigint price_snapshot "harga saat booking"
    }
    PAYMENT {
        uuid id PK
        uuid booking_id FK
        text method "payment_type dari callback"
        text status "PENDING, SUCCESS, FAILED, EXPIRED"
        text provider_ref "order_id Midtrans"
        text snap_token
        timestamptz created_at
    }
    PAYMENT_CALLBACK_LOG {
        text idempotency_key PK "order_id:transaction_status"
        uuid booking_id
        text result
        timestamptz processed_at
    }
```

Garis putus-putus = relasi logis tanpa foreign key (tabel dimiliki service berbeda; konsistensi dijaga di aplikasi).

## Kepemilikan tabel

| Tabel | Ditulis oleh | Dibaca juga oleh |
|---|---|---|
| `app_user`, `member_nik` | user-service | — |
| `residents` | dukcapil-mock (seed) | — |
| `event`, `ticket_category` | event-service (CRUD), booking & payment (`available`) | queue-service (warm-up stok) |
| `booking`, `booking_item` | booking-service, payment-service (`status`) | user-service (riwayat) |
| `payment`, `payment_callback_log` | payment-service, booking-service (`PENDING → EXPIRED` saat cancel) | — |

## Index & constraint penting

| Objek | Fungsi |
|---|---|
| `ticket_category.available CHECK (available >= 0)` + `CHECK (available <= capacity)` | Lapis terakhir anti-oversell |
| `idx_booking_item_nik (nik)` + `idx_booking_event_status (event_id, status)` | Hitung kuota 1 tiket per NIK per event |
| `idx_booking_payment_expires (payment_expires_at) WHERE status='HELD'` | Sweeper booking kedaluwarsa |
| `payment_callback_log.idempotency_key` (PK) | Webhook Midtrans idempoten |
| `idx_event_name_unique (lower(name))` | Nama event unik tanpa membedakan huruf |
| `member_nik UNIQUE (user_id, nik)` | Satu user tidak menyimpan NIK yang sama dua kali |

## Catatan desain

| Keputusan | Alasan |
|---|---|
| `nik` di `booking_item`, bukan di `booking` | 1 tiket = 1 NIK; satu booking (≤ 3 tiket) bisa berisi beberapa penonton |
| `booking.user_id` nullable | Login opsional; guest membeli dengan email + data penonton |
| `price_snapshot` | Harga bisa diubah admin; tagihan mengikuti harga saat booking |
| `booking.id` = `order_id` Midtrans | Charge & callback idempoten tanpa tabel pemetaan |
| Migrasi tanpa auto-migrate | Dijalankan saat provisioning (lokal: init script Postgres, lihat [LOCAL-SETUP.md](LOCAL-SETUP.md)) |
