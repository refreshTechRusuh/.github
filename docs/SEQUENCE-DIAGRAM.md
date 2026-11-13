# Sequence Diagram — Rusuh Booking Tiket

> Semua endpoint di bawah lewat API Gateway (`https://<gateway>`). Base path & header lengkap ada di [openapi.yaml](openapi.yaml).

## 1. Alur utama: antrian → booking → bayar sukses

```mermaid
sequenceDiagram
    actor C as Customer
    participant U as user-service
    participant Q as queue-service
    participant E as event-service
    participant B as booking-service
    participant D as dukcapil-mock
    participant P as payment-service
    participant M as Midtrans
    participant K as Kafka
    participant W as worker-service

    opt Login (opsional)
        C->>U: POST /auth/login
        U-->>C: authToken (JWT)
    end
    C->>E: GET /api/event/v1/list
    C->>Q: POST /api/queue/v1/event/{eventId}/join
    alt booking room masih muat (orang < floor(stok/3))
        Q-->>C: ADMITTED + accessToken (5 menit)
    else penuh
        Q-->>C: QUEUED + queueToken + position
        C->>Q: GET .../{queueToken}/stream (SSE)
        loop sampai giliran
            Q-->>C: event: position {position, etaSeconds}
        end
        Q-->>C: event: admitted {accessToken, bookingExpiresAt}
    end
    C->>E: GET /api/event/v1/detail/{eventId} (X-Access-Token)
    C->>B: GET /api/booking/v1/event/{eventId}/stock/stream (SSE, X-Access-Token)
    B-->>C: event: stock {categories[available]}
    C->>B: POST /api/booking/v1/create (X-Access-Token, Bearer opsional)
    B->>D: POST /api/dukcapil/v1/verify (per penonton)
    D-->>B: valid
    B->>B: 1 transaksi: lock NIK, kuota 1 tiket/NIK, HOLD stok, insert booking
    B-->>C: HELD + bookingId + paymentToken (10 menit)
    B--)K: booking.created, stock.changed (HOLD)
    K--)W: booking.created → email instruksi bayar
    C->>P: POST /api/payment/v1/charge (X-Payment-Token)
    P->>M: Snap create (order_id = bookingId)
    P-->>C: snapToken + redirectUrl
    C->>M: bayar (VA / QRIS / e-wallet)
    M->>P: POST /api/payment/v1/callback (settlement)
    P->>P: cek signature + gross_amount, idempoten, booking PAID
    P--)K: payment.succeeded, slot.released (PAID)
    K--)W: email e-tiket
    K--)Q: lepas kursi → promosi antrian
```

## 2. Cancel booking (user kembali dari Payment Page)

```mermaid
sequenceDiagram
    actor C as Customer
    participant B as booking-service
    participant R as Redis
    participant M as Midtrans
    participant K as Kafka
    participant Q as queue-service

    C->>B: POST /api/booking/v1/{bookingId}/cancel (X-Payment-Token)
    B->>B: SELECT booking FOR UPDATE
    alt PAID atau payment SUCCESS
        B-->>C: 409
    else EXPIRED
        B-->>C: 410
    else CANCELLED
        B-->>C: 200 (idempoten, tanpa efek)
    else HELD
        B->>B: payment PENDING → EXPIRED, booking → CANCELLED, stok kembali (1 transaksi)
        B->>R: SET queue:revoked:{queueToken}
        B-->>C: 200 CANCELLED
        B--)K: slot.released {CANCELLED, queueToken}, stock.changed (RELEASE)
        K--)Q: lepas kursi → orang berikutnya ADMITTED
        opt Snap sudah dibuka
            B->>M: POST /v2/{bookingId}/expire
        end
    end
    Note over C,B: Create ulang dengan Access Token lama → 410 (harus antri ulang)
```

## 3. Access Token habis sebelum submit (idle di Detail Ticket)

```mermaid
sequenceDiagram
    actor C as Customer
    participant Q as queue-service
    participant B as booking-service

    Note over C,Q: Access Token berlaku 5 menit sejak admitted
    C-->>B: tidak submit
    Q->>Q: sweeper (tiap 1 detik): expiresAt lewat → kursi dilepas
    Q->>Q: promosi orang berikutnya (tanpa Kafka, tidak ada stok yang di-hold)
    C->>B: POST /create dengan token kedaluwarsa
    B-->>C: 401 / 410 → kembali ke antrian
```

## 4. Batas bayar 10 menit habis

```mermaid
sequenceDiagram
    actor C as Customer
    participant B as booking-service
    participant P as payment-service
    participant K as Kafka
    participant Q as queue-service
    participant W as worker-service

    C-->>P: tidak membayar
    alt belum pernah charge
        B->>B: sweeper (30 detik): HELD → EXPIRED, stok kembali
        B--)K: slot.released (BOOKING_TTL), stock.changed (RELEASE)
    else sudah charge (payment PENDING)
        P->>P: sweeper (60 detik): payment & booking → EXPIRED, stok kembali
        P--)K: payment.expired, slot.released (PAYMENT_TTL)
        K--)W: email pembatalan
    end
    K--)Q: promosi antrian
    Note over B,P: Status hanya berubah bila masih HELD → stok tidak pernah dilepas dua kali
```

## 5. Sold out → antrian ditutup

```mermaid
sequenceDiagram
    participant B as booking/payment-service
    participant E as event-service
    participant K as Kafka
    participant Q as queue-service
    actor C as Customer di antrian

    B->>B: semua kategori available = 0 DAN tidak ada booking HELD
    B->>E: POST /internal/event/v1/{eventId}/sold-out
    B--)K: event.sold_out
    K--)Q: tutup antrian
    Q-->>C: SSE event: sold_out
    Note over Q: join baru → 410 SOLD_OUT
```

## 6. Pembayaran dari Mobile (WebView)

```mermaid
sequenceDiagram
    actor C as Customer
    participant A as Mobile App
    participant V as WebView (payment page web)
    participant P as payment-service
    participant M as Midtrans

    C->>A: lanjut bayar
    A->>V: buka payment page (bookingId + paymentToken)
    V->>P: GET /api/payment/v1/booking/{bookingId} (X-Payment-Token)
    P-->>V: total, remainingSeconds, backDisabled, redirectUrl
    V->>P: POST /api/payment/v1/charge
    P-->>V: snapToken / redirectUrl
    V->>M: Snap (UI yang sama dengan web)
    M->>P: callback (final, idempoten)
    V-->>A: redirect hasil → app menampilkan status
```

## 7. Kanal realtime

| Kanal | Endpoint | Event SSE | Ditutup saat |
|---|---|---|---|
| Posisi antrian | `GET /api/queue/v1/event/{eventId}/{queueToken}/stream` | `position`, `admitted`, `sold_out`, `expired`, `error`; heartbeat `: ping` | `admitted` / `sold_out` / `expired` |
| Stok per kategori | `GET /api/booking/v1/event/{eventId}/stock/stream` (header `X-Access-Token`) | `stock` (batch ≤ 1/detik); heartbeat 30 detik | `soldOut` |

Stream stok butuh header, jadi klien memakai SSE berbasis `fetch` (bukan `EventSource` bawaan browser). Lewat gateway, stream maksimal 3570 detik → klien auto-reconnect.
