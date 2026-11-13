# PRD — Rusuh Booking Tiket

> Product Requirement Document. Sumber kebenaran bisnis untuk seluruh tim (BE, FE Web, Mobile).
> Perbedaan antara requirement dan implementasi saat ini dicatat di [§11 Catatan implementasi](#11-catatan-implementasi).
> Diturunkan dari dokumen tender "Evaluasi Expertise Solution Development 2026 H2 — Tender Rusuh Booking Tiket".

| Field | Nilai |
|---|---|
| Produk | Rusuh Booking Tiket |
| Klien | PT. Tiga Sekawan Angin Ribut |
| Versi dokumen | 1.4 |
| Tanggal | 16 Oktober 2026 (rev. 13 November 2026 — sinkron dengan implementasi) |
| Deadline project | 13 November 2026 (±1 bulan) |
| Tim | 2 Backend, 2 Frontend Web, 1 Mobile |

---

## 1. Latar Belakang & Masalah

PT. Tiga Sekawan Angin Ribut menjual tiket konser secara manual via gerai retail. Mereka ingin ekspansi ke penjualan online lewat aplikasi **Rusuh Booking Tiket**.

Masalah inti **bukan fitur fungsional biasa**, melainkan **menangani lonjakan trafik ekstrem (flash crowd)** saat penjualan dibuka, di mana jumlah permintaan jauh melampaui jumlah tiket. Sistem harus tetap stabil, adil, dan tidak memutus journey pengguna di tengah jalan.

### Dua prinsip kunci (tidak bisa ditawar)

1. **Jaga User Experience apa pun kondisi sistem di belakang.** Saat beban puncak, Waiting Room melindungi layanan inti dari overload.
2. **"Jangan PHP".** Pengguna yang sudah masuk antrian dan memulai booking **dijamin** bisa menyelesaikan journey-nya sampai akhir tanpa diputus sepihak (session & hold dipertahankan sepanjang TTL).

> **Perubahan v1.2 (hasil MOM 20-10-2026 + Form Klarifikasi CEO).** Keputusan resmi yang mengubah/menegaskan requirement:
> 1. **Login OPTIONAL** (seperti KAI), bukan wajib. Guest boleh mengantre & membeli; saat memilih tiket, guest **wajib mengisi NIK + email** (NIK untuk kuota & e-tiket, email untuk pengiriman). _(Klarifikasi Q2 & Q12, MOM #1)_
> 2. **Identitas antrian = NIK.** 1 NIK hanya boleh 1 antrian per event (bukan per akun) — mencegah satu orang titip banyak order. _(Klarifikasi Q12)_
> 3. **1 kode booking = maks 3 tiket, 1 tiket = 1 NIK.** Satu akun/pembeli boleh membeli 3 tiket dalam satu transaksi, tiap tiket mencatat 1 NIK penonton. **Kuota 1 tiket per NIK per event** (v1.4 — setiap NIK hanya boleh terdaftar di 1 tiket untuk event yang sama). _(Klarifikasi Q3)_
> 4. **CRUD data member/NIK.** Backend menyediakan CRUD NIK (NIK tersimpan agar dapat dipilih saat membeli, mis. NIK keluarga). _(MOM #2, #3)_
> 5. **Mock Dukcapil service** untuk validasi data KTP/NIK (bukan hanya cek 16 digit). _(MOM #9)_
> 6. **Disable tombol back di Payment** saat TTL pilih tiket hampir habis (mis. sisa < 30 detik) agar customer tidak kehilangan tiket karena timeout saat revisi. _(MOM #8)_
> 7. **Refund OUT OF SCOPE** — ditangani Customer Service dengan data/evidence, bukan fitur sistem. Harga **all-in**. Bayar → tiket **issued** (final, tanpa refund). _(Klarifikasi Q4, Q7, Q9, MOM #7)_
> 8. **Email wajib hanya e-tiket (issued).** Email instruksi bayar & expired bersifat opsional/disesuaikan (tetap diimplementasi, status "wajib" hanya e-tiket). _(Klarifikasi Q11)_
> 9. **Estimasi waktu tunggu (FR-09) dinaikkan** dari COULD → diminta eksplisit oleh CEO; termasuk perhitungan kapasitas waiting/booking room & estimasi tunggu. _(MOM #4, #5)_

> **Perubahan v1.3 (flow fix 21-10-2026).** Penajaman flow setelah review:
> 10. **Validasi NIK via pencocokan data (v1.3.1, revisi).** Tambah member & guest-isi-NIK dicocokkan `{nik, nama, tgl lahir, nama ibu kandung}` ke Dukcapil mock. **OTP/SMS dihilangkan** (di dunia nyata validasi NIK tidak pakai OTP; lebih sederhana & realistis). _(FR-16; MOM #9)_
> 11. **Logout & riwayat tiket** untuk user login (FR-A8, FR-A9).
> 12. **Batas kapasitas antrian (queue cap)** — antrian penuh = "ramai, coba lagi" (`QUEUE_FULL`), beda dari sold-out (FR-10c). _(MOM #4)_
> 13. **Disable metode bayar dinamis** berdasarkan estimasi durasi vs sisa TTL (FR-27). _(MOM #6)_
> 14. **TTL ditegaskan**: pilih tiket **5 menit**, pembayaran **10 menit**; ambang disable-back **30 detik**. **Token**: JWT login (`authToken`, opsional), queue token, Access Token (antrian→booking, pembawa TTL booking).
> 15. **Anti-PHP absolut (BR-13):** jumlah orang di booking room ≤ `floor(stok/3)` (sisa ≤3 → 1 orang) → yang sudah masuk dijamin dapat tiket. Promosi antrean gelombang. _(menutup kasus "2 user rebut 1 tiket")_
> 16. **Hold stok ditutup**: hold DB real **hanya saat submit**; masuk booking room tidak hold (pembatasan jumlah orang = reservasi implisit). Tombol tambah/kurang = pilih qty (UI).
> 17. **Standar response API ditetapkan** = envelope `{status, message, data}` (+`partial`); `status="00000"` sukses / string kode HTTP gagal; transaction ID di **header**; kode error dipetakan ke HTTP status internal. Lihat [openapi.yaml](openapi.yaml).

---

## 2. Tujuan & Metrik Sukses

| Tujuan | Metrik |
|---|---|
| Tidak ada oversell | 0 tiket terjual melebihi kapasitas kategori |
| Antrian adil | Urutan FCFS terjaga; posisi tidak bisa dilangkahi |
| Sistem tidak crash saat flash crowd | Layanan inti tetap hidup; Waiting Room men-throttle beban |
| Journey tidak terputus | Customer dengan slot aktif selesai sampai akhir dalam TTL |
| Latensi rendah | p95 update antrian & home page < 1 detik di beban tinggi |
| Pembayaran tidak double-charge | Callback idempoten dengan idempotency key |

---

## 3. Ruang Lingkup

### In Scope
- **Autentikasi pengguna (login/logout) — OPTIONAL.** Login/logout tersedia (menyimpan NIK favorit, riwayat tiket) tetapi **tidak wajib** untuk membeli (pola seperti KAI). Guest boleh mengantre & membeli dengan mengisi **NIK + email** saat memilih tiket.
- **Riwayat tiket user (login).** User login bisa melihat daftar booking/e-tiket sebelumnya. _(Flow fix)_
- **CRUD data member/NIK.** User (login) menyimpan & mengelola daftar NIK (mis. NIK keluarga) untuk dipilih (dropdown) saat membeli tanpa input ulang. Setiap NIK **divalidasi ke Dukcapil** (pencocokan `nik+nama+tgl lahir+nama ibu`) saat disimpan. _(MOM #2–#3, flow fix)_
- **Mock Dukcapil (pencocokan data).** Validasi NIK/KTP terhadap layanan Dukcapil tiruan yang menyimpan `{nik, nama, tgl lahir, nama ibu kandung}` — `valid` hanya bila keempat field cocok. **Tanpa OTP/SMS** (lebih realistis). _(MOM #9, flow fix)_
- **Manajemen event oleh admin (back-office).** CRUD event + kategori tiket lewat Event Service + admin page.
- Daftar event + status ketersediaan (available / sold out).
- Virtual Waiting Room dengan posisi antrian real-time; **1 NIK = 1 antrian per event**; **batas kapasitas antrian (queue cap)** untuk lindungi server (antrian penuh ≠ sold-out). _(flow fix, MOM #4)_
- Pemilihan kategori tiket (harga berbeda), **1 tiket per NIK per event** (v1.4); **1 kode booking ≤ 3 tiket, tiap tiket mencatat 1 NIK penonton berbeda**.
- **Estimasi waktu tunggu** di Waiting Room (diminta CEO) + perhitungan kapasitas waiting/booking room.
- **Stok tiket real-time di Detail Ticket Page** (push via SSE) agar angka available tidak basi ("jangan PHP").
- **Kapasitas booking room terikat stok (anti-PHP absolut).** Jumlah orang di booking room sebuah event dibatasi `floor(stok/3)` (sisa ≤3 → 1 orang) → begitu masuk booking room, customer **dijamin** bisa menyelesaikan pembelian tanpa direbut. Promosi antrean dilakukan **gelombang** (banyak orang sekaligus), bukan satu-satu. Hold stok real hanya saat submit. _(flow fix, NFR-02)_
- Alur pembayaran dengan **timeout 10 menit** dan pelepasan stok otomatis; **disable tombol back saat TTL pilih tiket hampir habis** (sisa < 30 detik, anti kehilangan tiket). _(MOM #8)_
- **Disable metode bayar dinamis**: metode yang estimasi durasi prosesnya lebih lama dari sisa TTL pembayaran dinonaktifkan otomatis (mis. metode 3 menit dimatikan saat sisa TTL < 3 menit), agar customer tidak gagal bayar karena kehabisan waktu. _(MOM #6, flow fix)_
- Notifikasi email: **e-tiket (wajib)**; instruksi bayar & expired (opsional, tetap diimplementasi).
- **Shared payment API** yang dikonsumsi web dan mobile, dengan payment page berbasis web yang reusable di dalam mobile (WebView).
- **Penutupan antrian saat sold out total** + pemberitahuan ke customer di antrian bahwa tiket habis (anti-PHP).

### Out of Scope
- Implementasi internal mitra pembayaran — diserahkan ke payment gateway pihak ketiga (**Midtrans**).
- **Refund & pembatalan oleh sistem** — ditangani **Customer Service** dengan data/evidence; bayar → tiket issued final. _(Klarifikasi Q4, Q9, MOM #7)_
- Program loyalti, diskon, dan voucher.
- Manajemen peran/izin admin yang kompleks (RBAC lanjutan) — cukup satu peran admin dasar untuk assessment.
- Sistem scanning/validasi tiket di venue (entry gate) — di luar lingkup backend ini.

---

## 4. Aktor & Peran

| Aktor | Tipe | Peran |
|---|---|---|
| Customer | Manusia | Memilih event, mengantre, memilih tiket, membayar. Login **optional** (guest cukup isi NIK + email) |
| Admin | Manusia | Kelola event & kategori tiket (CRUD) lewat admin page |
| User Service | Sistem | Autentikasi (login, optional), CRUD member/NIK, penerbitan & validasi sesi/token user |
| Dukcapil Mock | Sistem (mock) | Validasi NIK/data KTP tiruan untuk assessment (MOM #9) |
| Event Service | Sistem | CRUD event & kategori tiket (sumber katalog), back-office admin |
| Queue Service | Sistem | Antrian virtual, token akses, posisi antrian real-time, penutupan antrian saat sold out |
| Booking Service | Sistem | Stok tiket, reservasi sementara (hold), status booking, push stok real-time |
| Payment Gateway (Midtrans) | Sistem Eksternal | Memproses pembayaran web/mobile via Midtrans (Snap/Core API), kirim callback/webhook status |
| Notification/Email Service | Sistem | Kirim e-tiket atau notifikasi booking expired |
| Mobile App | Sistem Klien | Mengonsumsi API pembayaran yang sama dengan web |

---

## 5. Glosarium

| Istilah | Definisi |
|---|---|
| Waiting Room | Ruang tunggu virtual tempat customer menunggu giliran masuk halaman pembelian |
| Access Token / Slot | Hak akses sementara untuk masuk ke Detail Ticket Page |
| Hold / Reservasi | Penguncian stok tiket sementara saat customer masuk halaman payment |
| TTL Booking | Batas waktu di Detail Ticket Page untuk memilih tiket (±5 menit) sebelum sesi kedaluwarsa |
| TTL Pembayaran | Batas waktu 10 menit menyelesaikan pembayaran sebelum booking dibatalkan & stok di-release |
| NIK | Nomor Induk Kependudukan (16 digit), basis pembatasan kuota **1 tiket per NIK per event** (v1.4) |
| Sesi Pembelian | Satu rangkaian proses pembelian oleh satu customer dalam satu slot akses |
| Akun / User | Identitas login customer (mis. email + password), **opsional**. Dipakai untuk menyimpan NIK (member) dan riwayat tiket; booking guest tetap dapat dibuat dengan NIK + email |
| Admin | User dengan peran back-office untuk mengelola katalog event |
| SSE | Server-Sent Events — kanal push server→klien satu arah, dipakai untuk posisi antrian **dan** stok real-time di Detail Ticket Page |

---

## 6. Halaman Utama

| Halaman | Fungsi |
|---|---|
| Login Page | Autentikasi customer (**OPTIONAL** — hanya untuk simpan NIK/riwayat; tidak wajib untuk membeli) |
| Home Page | Daftar event yang tiketnya dijual online |
| Waiting Room | Ruang tunggu antrian virtual sebelum boleh membeli (posisi real-time via SSE, estimasi waktu tunggu) |
| Detail Ticket Page | Pemilihan kategori & jumlah tiket, **input NIK + email (guest)** / pilih NIK tersimpan (login), stok real-time via SSE |
| Payment Page | Pembayaran dengan batas 10 menit, dikonsumsi web & mobile; tombol back di-disable saat TTL pilih tiket hampir habis |

> Halaman admin (**Admin Page**) terpisah dari journey customer: form CRUD event & kategori tiket, dikonsumsi dari Event Service.
> Halaman **Member/NIK** (porsi FE, login): CRUD daftar NIK untuk dipilih saat membeli (MOM #2–#3).

---

## 7. Functional Requirements (MoSCoW)

### 7.0 Autentikasi & Member NIK (User Service)
| ID | Requirement | Prioritas |
|---|---|---|
| FR-A1 | Customer dapat register & login (mis. email + password) | MUST |
| FR-A2 | Backend menerbitkan token sesi (mis. JWT) setelah login sukses | MUST |
| FR-A3 | Login **OPTIONAL** untuk membeli (seperti KAI). Guest boleh mengantre & membeli dengan mengisi **NIK + email** saat memilih tiket | MUST |
| FR-A4 | Antrian, booking, dan e-tiket ditautkan ke **NIK** (dan ke `userId` bila login) | MUST |
| FR-A5 | Admin login dengan peran `admin` untuk mengakses Admin Page | MUST |
| FR-A6 | User (login) dapat **CRUD daftar NIK/member** (mis. NIK keluarga) untuk dipilih saat membeli | SHOULD |
| FR-A7 | NIK divalidasi ke **mock Dukcapil service** (bukan hanya format 16 digit) saat dipakai/ditambahkan | SHOULD |
| FR-16 | NIK divalidasi via **pencocokan data** (`nik+nama+tgl lahir+nama ibu kandung`) ke Dukcapil mock saat tambah member & saat booking — data penonton tepercaya (tanpa OTP) | SHOULD |
| FR-A8 | User/admin dapat **logout** (mencabut sesi login: `authToken` + `refreshToken`) | SHOULD |
| FR-A10 | Klien dapat memperbarui sesi lewat **refresh token** (`POST /auth/refresh`) tanpa login ulang selama refresh token berlaku (default 7 hari); refresh token sekali pakai | SHOULD |
| FR-A9 | User (login) dapat melihat **riwayat tiket/booking** miliknya | SHOULD |

### 7.0b Manajemen Event oleh Admin (Event Service)
| ID | Requirement | Prioritas |
|---|---|---|
| FR-E1 | Admin dapat membuat event baru (nama, jadwal sale, status) | MUST |
| FR-E2 | Admin dapat membuat/mengubah kategori tiket (nama, harga, kapasitas) per event | MUST |
| FR-E3 | Admin dapat memperbarui & menghapus/menonaktifkan event | MUST |
| FR-E4 | Admin dapat melihat daftar seluruh event beserta status & oksupansi stok | MUST |
| FR-E5 | Perubahan kapasitas kategori tidak boleh membuat `available` menjadi negatif | MUST |
| FR-E6 | Endpoint admin diproteksi (hanya role `admin`) | MUST |

### 7.1 Home Page
| ID | Requirement | Prioritas |
|---|---|---|
| FR-01 | Menampilkan daftar seluruh event yang dijual online | MUST |
| FR-02 | Event available & masih ada stok tampil aktif/clickable | MUST |
| FR-03 | Event sold out / tidak available tampil non-aktif + label status | MUST |
| FR-04 | Saat klik event dan ada antrian, customer diarahkan ke Waiting Room | MUST |
| FR-05 | Jika tidak ada antrian, customer langsung masuk Detail Ticket Page | MUST |

### 7.2 Waiting Room
| ID | Requirement | Prioritas |
|---|---|---|
| FR-06 | Menampilkan posisi antrian dihitung dari paling depan ("Anda antrian ke-n") | MUST |
| FR-07 | Posisi antrian diperbarui otomatis saat antrian di depan berkurang | MUST |
| FR-08 | Customer otomatis dipindah ke Detail Ticket Page saat gilirannya tiba | MUST |
| FR-09 | Menampilkan estimasi waktu tunggu berdasarkan laju antrian (diminta CEO; termasuk perhitungan kapasitas waiting/booking room) | SHOULD |
| FR-10 | Posisi antrian konsisten saat refresh (tidak kehilangan giliran) | MUST |
| FR-10a | Push posisi antrian real-time memakai **SSE** (satu arah server→klien, hemat untuk ratusan ribu koneksi) | MUST |
| FR-10b | Saat seluruh kategori event **sold out**, THE sistem menutup antrian: tidak menerima enqueue baru DAN memberi notifikasi "tiket sudah habis" ke customer yang masih di antrian, lalu mengeluarkan mereka dari Waiting Room (anti-PHP) | MUST |
| FR-10c | **Batas kapasitas antrian (queue cap)**: saat antrian sudah mencapai batas, enqueue baru ditolak dengan pesan "ramai, coba lagi" (`QUEUE_FULL`) — dibedakan dari sold-out | SHOULD |

### 7.3 Detail Ticket Page
| ID | Requirement | Prioritas |
|---|---|---|
| FR-11 | Menampilkan detail event beserta kategori tempat (seat category) | MUST |
| FR-12 | Setiap kategori menampilkan jumlah available dan harga berbeda | MUST |
| FR-12a | Jumlah available per kategori diperbarui **real-time via SSE** selama customer di Detail Ticket Page, agar stok tidak basi (anti-PHP) | MUST |
| FR-13 | Maks 3 tiket per **kode booking**; **tiap tiket mencatat 1 NIK penonton** dan satu NIK maks **1 tiket per event** (BR-01 v1.4). Guest wajib mengisi NIK + email sebelum lanjut | MUST |
| FR-14 | Menolak jumlah yang melebihi stok, melebihi 3 per booking, atau NIK yang sudah memegang tiket `HELD/PAID` untuk event yang sama (lintas sesi/perangkat) | MUST |
| FR-15 | Validasi format NIK (16 digit) **+ validasi ke mock Dukcapil**; cegah NIK sama melampaui kuota **1 tiket per event** | MUST |
| FR-16a | Saat submit booking, stok di-hold (reservasi sementara) | MUST |
| FR-17 | Menampilkan countdown TTL booking (±5 menit) di Detail Ticket Page | MUST |
| FR-18 | Jika idle melebihi TTL booking: sesi dibatalkan, hold dilepas, slot dilepas untuk antrian berikutnya, customer dinotifikasi & diarahkan keluar | MUST |
| FR-19 | Timer booking server-authoritative (tak bisa di-bypass klien) & konsisten saat refresh | MUST |

### 7.4 Payment Page
| ID | Requirement | Prioritas |
|---|---|---|
| FR-20 | Customer dapat memilih metode pembayaran dari mitra yang tersedia | MUST |
| FR-21 | Menampilkan countdown TTL pembayaran 10 menit sejak booking | MUST |
| FR-22 | Jika pembayaran sukses tepat waktu, kirim email e-tiket | MUST |
| FR-23 | Jika tidak selesai dalam 10 menit: booking dibatalkan, email expired dikirim, stok dilepas | MUST |
| FR-24 | API pembayaran dikonsumsi web & mobile (shared backend API) | MUST |
| FR-25 | Menangani callback status (success/fail/pending) secara idempoten | MUST |
| FR-26 | Saat TTL pilih tiket hampir habis (mis. sisa < 30 detik), backend menandai agar klien **menonaktifkan tombol back** ke Detail Ticket (mitigasi kehilangan tiket karena timeout saat revisi) | SHOULD |
| FR-27 | **Disable metode bayar dinamis**: metode dengan estimasi durasi proses > sisa TTL pembayaran dinonaktifkan; `charge` metode tak cukup waktu ditolak server | SHOULD |

---

## 8. Non-Functional Requirements

| ID | Kategori | Requirement |
|---|---|---|
| NFR-01 | Availability/UX | UX terjaga apa pun kondisi backend; Waiting Room melindungi layanan inti dari overload |
| NFR-02 | Reliability | "Jangan PHP": slot/booking aktif dijamin selesai dalam TTL; session & hold dipertahankan |
| NFR-03 | Scalability | Tahan flash crowd jauh di atas kapasitas tiket; auto-scaling pada layer stateless |
| NFR-04 | Consistency | Tidak boleh oversell; hold/pengurangan stok harus atomic (distributed lock / atomic counter) |
| NFR-05 | Fairness | Antrian FCFS; posisi adil & tidak bisa dilangkahi |
| NFR-06 | Performance | Update antrian & home page p95 < 1 detik di beban tinggi |
| NFR-06a | Realtime channel | Posisi antrian **dan** stok Detail Ticket pakai **SSE** (satu arah server→klien, hemat untuk ratusan ribu klien, seragam satu teknologi). Hindari polling berat. |
| NFR-07 | Security | Akses Detail/Payment diproteksi token; data pembayaran ikut standar (mis. PCI-DSS pada gateway) |
| NFR-08 | Portability | API pembayaran platform-agnostic (REST/JSON) untuk web & mobile tanpa ubah backend |
| NFR-09 | Observability | Metrik antrian, laju konversi, rate expired dipantau real-time |
| NFR-10 | Resilience | Kegagalan satu layanan (mis. email) tidak membatalkan transaksi; async dengan retry & DLQ |

---

## 9. Business Rules

| ID | Aturan |
|---|---|
| BR-01 | **1 tiket per NIK per event** (v1.4). Dalam 1 kode booking maks 3 tiket; tiap tiket mencatat 1 NIK penonton berbeda |
| BR-02 | Satu NIK tidak dapat melampaui 1 tiket pada event yang sama, meski via sesi/perangkat/booking berbeda |
| BR-03 | TTL Booking = ±5 menit sejak akses diberikan |
| BR-04 | Jika TTL Booking habis: sesi batal → hold dilepas → slot diberikan ke antrian berikutnya |
| BR-05 | TTL Pembayaran = 10 menit sejak booking dibuat |
| BR-06 | Jika TTL Pembayaran habis: booking batal → stok dilepas → email expired → slot ke antrian berikutnya |
| BR-07 | **Satu NIK hanya memegang satu antrian/slot aktif per event** (bukan per akun) — mencegah titip banyak order lewat akun berbeda. _Implementasi: ditegakkan saat booking (1 tiket per NIK per event), bukan saat join antrian — lihat §11_ |
| BR-08 | Stok yang sudah dibayar bersifat final; stok yang di-hold tidak dapat dibeli customer lain |
| BR-09 | **Login OPTIONAL** untuk membeli (seperti KAI); guest mengisi NIK + email saat memilih tiket. Antrian & booking ditautkan ke **NIK** (dan ke `userId` bila login) |
| BR-10 | Saat seluruh kategori event sold out (available = 0 & tidak ada hold yang bisa kembali jadi tersedia), antrian ditutup: enqueue baru ditolak, customer di antrian dinotifikasi "sold out" dan dikeluarkan |
| BR-11 | Hanya user dengan peran `admin` yang boleh melakukan CRUD event/kategori |
| BR-12 | **Refund out of scope** (ditangani Customer Service); pembayaran sukses → tiket **issued** final. Harga **all-in** |
| BR-13 | **Jumlah orang di booking room per event ≤ `floor(stok/3)`** (sisa ≤3 → 1 orang). Menjamin anti-PHP tanpa hold saat masuk; hold stok real hanya saat submit. Promosi antrean dilakukan gelombang. _(flow fix, Req 23)_ |

---

## 10. Asumsi & Risiko

### Asumsi
- Data event (jadwal, kategori, harga, kapasitas) dikelola **admin lewat Event Service** (bukan sistem eksternal).
- Login **opsional**; email diisi saat membeli dan dipakai untuk pengiriman e-tiket. Akun (User Service) untuk menyimpan NIK, riwayat, dan admin.
- Mitra pembayaran (**Midtrans**) menyediakan mekanisme callback/webhook status (sandbox tersedia untuk testing).
- Satu peran admin dasar cukup untuk assessment (tanpa RBAC kompleks).

### Risiko & Mitigasi
| Risiko | Mitigasi |
|---|---|
| Oversell saat trafik puncak | Atomic counter / distributed lock pada operasi hold stok |
| Sistem inti overload oleh flash crowd | Virtual Waiting Room sebagai throttle; auto-scaling layer stateless |
| Customer kehilangan giliran saat refresh | Simpan posisi antrian berbasis token persisten, bukan session browser |
| Email gagal terkirim | Async dengan retry & DLQ; status booking tetap valid |
| Double-charge pembayaran | Callback diproses idempoten dengan idempotency key |
| Satu orang memborong tiket banyak sesi | Kuota 1 tiket per NIK per event lintas sesi/perangkat; validasi atomic saat booking |
| NIK palsu / format tidak valid | Validasi format 16 digit + pencocokan data ke Dukcapil (mock) |
| Customer menahan slot dengan diam | TTL Booking server-authoritative; sesi otomatis berakhir & slot dilepas |

---

## 11. Catatan implementasi

Status per 13 November 2026. Requirement di atas tetap menjadi acuan; tabel ini menjelaskan cara/tingkat implementasinya.

| Requirement | Implementasi saat ini |
|---|---|
| Item 2 / BR-07 — 1 NIK = 1 antrian per event | Join antrian **tidak** meminta NIK (NIK baru diketahui saat memilih tiket). Pencegahan titip order ditegakkan saat booking: **1 tiket per NIK per event**, dicek atomik dengan lock per NIK. Identitas antrian = queue token. |
| FR-14 — akumulasi per NIK > 3 ditolak | Digantikan BR-01 v1.4: maksimal **1 tiket per NIK per event**; maksimal 3 tiket per kode booking. |
| FR-15 / FR-16 — validasi NIK ke Dukcapil | Diimplementasikan: 4 field (`nik, nama, tglLahir, namaIbu`) dicocokkan. Saat Dukcapil tidak terjangkau, booking-service mengikuti `DUKCAPIL_FAIL_OPEN` (default lolos agar journey tidak terputus); simpan member selalu fail-closed. |
| FR-18 — TTL booking habis | Kursi booking room dilepas otomatis oleh sweeper antrian (tiap 1 detik) dan langsung diberikan ke antrian berikutnya. Tidak ada stok yang di-hold sebelum submit. |
| FR-26 — disable tombol back | `GET /api/booking/v1/{bookingId}` mengembalikan `backDisabled=true` saat sisa waktu < 30 detik. |
| FR-27 — disable metode bayar dinamis | **Belum diimplementasikan.** Expiry transaksi Midtrans Snap disamakan dengan sisa waktu bayar, sehingga transaksi tidak bisa melewati batas TTL. |
| NFR-09 — metrik real-time | Log terstruktur (Cloud Logging) dan distributed tracing (Cloud Trace). **Dashboard metrik (Prometheus/Grafana) belum diimplementasikan.** |
| Tambahan — cancel booking | User dapat membatalkan booking `HELD` (mis. kembali dari Payment Page). Stok kembali, transaksi Midtrans di-expire, user harus antri ulang. |
| Tambahan — callback untuk booking tertutup | Pembayaran yang masuk setelah booking dibatalkan/kedaluwarsa tidak menerbitkan e-tiket dan ditandai untuk refund manual oleh CS (BR-12). |
