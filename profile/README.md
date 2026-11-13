# Rusuh Booking Tiket

Sistem pemesanan tiket konser untuk **flash crowd** — waiting room FCFS realtime, anti-oversell, dan jaminan
"jangan PHP": siapa pun yang sudah masuk booking room dijamin kebagian tiket.

**Go · PostgreSQL · Redis · Kafka · SSE · Midtrans · Google Cloud Run**

| Service | Peran |
|---|---|
| [user-service](https://github.com/refreshTechRusuh/user-service) | Auth (login opsional), member NIK, riwayat tiket |
| [queue-service](https://github.com/refreshTechRusuh/queue-service) | Waiting room + admission control (SSE) |
| [event-service](https://github.com/refreshTechRusuh/event-service) | Katalog & admin event |
| [booking-service](https://github.com/refreshTechRusuh/booking-service) | Hold stok atomik, cancel, stok realtime (SSE) |
| [payment-service](https://github.com/refreshTechRusuh/payment-service) | Midtrans Snap, callback idempoten |
| [worker-service](https://github.com/refreshTechRusuh/worker-service) | Email async (retry + DLQ) |
| [dukcapil-mock](https://github.com/refreshTechRusuh/dukcapil-mock) | Mock validasi NIK |

📄 **Dokumentasi lengkap** (ringkasan, HLD, LLD, PRD, ERD, OpenAPI, sequence diagram, cara menjalankan di lokal):
**[github.com/refreshTechRusuh/.github](https://github.com/refreshTechRusuh/.github#readme)**
