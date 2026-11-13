# Local Setup — menjalankan Rusuh Booking Tiket di laptop

Panduan ini menjalankan **7 service backend + infrastruktur** di laptop Anda sampai bisa menguji alur lengkap:
login → antrian (SSE) → detail tiket & stok realtime (SSE) → booking → cancel → pembayaran → email.

Semua langkah di bawah sudah diuji dari clone bersih (macOS, Docker Desktop, Go 1.26).

> Waktu yang dibutuhkan: ±15 menit (pertama kali, termasuk unduh image Docker & dependensi Go).

---

## 1. Prasyarat

| Tool | Versi | Cek |
|---|---|---|
| Git | apa saja | `git --version` |
| Docker + Docker Compose v2 | Docker Desktop ≥ 4.x, RAM Docker ≥ 4 GB | `docker compose version` |
| Go | **1.26+** (lihat `go.mod`) | `go version` |
| curl, jq | apa saja | `jq --version` |

Port yang dipakai (pastikan kosong): `5432` Postgres, `6379` Redis, `29092` Kafka, `1025`/`8025` Mailpit,
`8080`–`8086` service Go.

```bash
# cek port bentrok (macOS/Linux) — tidak boleh ada output
for p in 5432 6379 29092 1025 8025 8080 8081 8082 8083 8084 8085 8086; do lsof -nP -iTCP:$p -sTCP:LISTEN; done
```

---

## 2. Clone 7 repo ke satu folder

```bash
mkdir rusuh-local && cd rusuh-local
for r in user-service queue-service dukcapil-mock event-service booking-service payment-service worker-service; do
  git clone https://github.com/refreshTechRusuh/$r.git
done
```

Struktur akhir (file `docker-compose.yml` dibuat di langkah 3):

```
rusuh-local/
├── docker-compose.yml
├── user-service/       :8086  auth, member NIK, riwayat
├── dukcapil-mock/      :8085  mock validasi NIK
├── queue-service/      :8084  waiting room + SSE posisi
├── event-service/      :8082  katalog + admin CRUD
├── booking-service/    :8081  hold stok, cancel, SSE stok
├── payment-service/    :8083  Midtrans Snap + callback
└── worker-service/     :8080  email (Kafka consumer)
```

---

## 3. Infrastruktur (Postgres, Redis, Kafka, Mailpit)

Buat `rusuh-local/docker-compose.yml`:

```yaml
# Infrastruktur lokal Rusuh Booking Tiket.
name: rusuh-local

services:
  postgres:
    image: postgres:16-alpine
    container_name: rusuh-local-postgres
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
      POSTGRES_DB: rusuh_booking
    ports:
      - "5432:5432"
    volumes:
      - pg_data:/var/lib/postgresql/data
      # Migrasi dijalankan otomatis SEKALI saat volume masih kosong, urut nama file.
      - ./event-service/migrations/001_initial_schema.sql:/docker-entrypoint-initdb.d/10_event_001.sql:ro
      - ./event-service/migrations/002_unique_event_name.sql:/docker-entrypoint-initdb.d/11_event_002.sql:ro
      - ./event-service/migrations/003_add_poster_description.sql:/docker-entrypoint-initdb.d/12_event_003.sql:ro
      - ./booking-service/migrations/001_initial_schema.sql:/docker-entrypoint-initdb.d/20_booking_001.sql:ro
      - ./payment-service/migrations/001_initial_schema.sql:/docker-entrypoint-initdb.d/30_payment_001.sql:ro
      - ./dukcapil-mock/migrations/001_initial_schema.sql:/docker-entrypoint-initdb.d/40_dukcapil_001.sql:ro
      - ./user-service/migrations/001_initial_schema.sql:/docker-entrypoint-initdb.d/50_user_001.sql:ro
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d rusuh_booking"]
      interval: 5s
      timeout: 5s
      retries: 20

  redis:
    image: redis:7-alpine
    container_name: rusuh-local-redis
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 5s
      retries: 20

  zookeeper:
    image: confluentinc/cp-zookeeper:7.6.1
    container_name: rusuh-local-zookeeper
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000

  kafka:
    image: confluentinc/cp-kafka:7.6.1
    container_name: rusuh-local-kafka
    depends_on:
      - zookeeper
    ports:
      - "29092:29092" # dipakai service Go dari host: KAFKA_BROKERS=localhost:29092
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,PLAINTEXT_HOST://0.0.0.0:29092
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092,PLAINTEXT_HOST://localhost:29092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "true"
    healthcheck:
      test: ["CMD-SHELL", "kafka-topics --bootstrap-server localhost:9092 --list"]
      interval: 10s
      timeout: 10s
      retries: 20
      start_period: 20s

  # Membuat semua topic sekali lalu selesai (status "Exited (0)" itu normal).
  kafka-init:
    image: confluentinc/cp-kafka:7.6.1
    container_name: rusuh-local-kafka-init
    depends_on:
      kafka:
        condition: service_healthy
    entrypoint: ["/bin/sh", "-c"]
    command:
      - |
        for t in booking.created stock.changed slot.released payment.succeeded payment.expired event.sold_out; do
          kafka-topics --bootstrap-server kafka:9092 --create --if-not-exists --topic "$$t" --partitions 3 --replication-factor 1
        done
        for t in booking.created payment.succeeded payment.expired; do
          kafka-topics --bootstrap-server kafka:9092 --create --if-not-exists --topic "$$t.retry" --partitions 1 --replication-factor 1
          kafka-topics --bootstrap-server kafka:9092 --create --if-not-exists --topic "$$t.dlt" --partitions 1 --replication-factor 1
        done
        kafka-topics --bootstrap-server kafka:9092 --list

  # SMTP palsu + inbox web untuk melihat email: http://localhost:8025
  mailpit:
    image: axllent/mailpit:v1.21
    container_name: rusuh-local-mailpit
    environment:
      MP_SMTP_AUTH_ACCEPT_ANY: "1"      # worker selalu mengirim AUTH; terima kredensial apa pun
      MP_SMTP_AUTH_ALLOW_INSECURE: "1"  # izinkan AUTH tanpa TLS (khusus lokal)
    ports:
      - "1025:1025"
      - "8025:8025"

volumes:
  pg_data:
```

Jalankan dan verifikasi:

```bash
docker compose up -d
sleep 20
docker compose ps                     # postgres/redis/kafka "healthy", kafka-init "Exited (0)"
docker logs rusuh-local-kafka-init | tail -12   # 12 topic
docker exec rusuh-local-postgres psql -U postgres -d rusuh_booking -c '\dt'
#  → 9 tabel: app_user booking booking_item event member_nik payment payment_callback_log residents ticket_category
```

> **Kenapa migrasi lewat init Postgres?** Service tidak menjalankan migrasi otomatis. Init script Postgres hanya
> berjalan saat volume **masih kosong** — kalau Anda mengubah migrasi, reset dengan `docker compose down -v`.

Data awal yang langsung tersedia: **5 penduduk tiruan** di Dukcapil mock (dipakai sebagai NIK penonton).

| NIK | Nama | Tgl lahir | Nama ibu |
|---|---|---|---|
| 3273000000000001 | Budi Santoso | 1990-01-01 | Siti Aminah |
| 3273000000000002 | Ani Lestari | 1992-05-17 | Rina Marlina |
| 3273000000000003 | Candra Wijaya | 1988-11-30 | Dewi Kartika |
| 3273000000000004 | Dedi Pratama | 1995-07-09 | Eka Yuliana |
| 3273000000000005 | Eka Putri | 2000-03-25 | Fitri Handayani |

---

## 4. File `.env` per service

Setiap service membaca `.env` dari **folder repo-nya sendiri** (sudah di-`.gitignore`). Nilai di bawah adalah
**dummy khusus lokal** — jangan dipakai di production. Salin apa adanya.

Aturan yang wajib dijaga (kalau beda, request ditolak 401/403):
- `ACCESS_TOKEN_SECRET` **sama** di queue, event, booking, payment (token antrian).
- `PAYMENT_TOKEN_SECRET` **sama** di booking dan payment.
- `USER_JWT_SECRET` **sama** di user, event, booking (token login).
- `KAFKA_BROKERS=localhost:29092` (default beberapa service `localhost:9092`, itu port internal Docker).

```bash
cat > dukcapil-mock/.env <<'EOF'
APP_ENV=local
APP_PORT=8085
LOG_FORMAT=text
OTEL_ENABLED=false
DB_PASSWORD=password
EOF

cat > user-service/.env <<'EOF'
APP_ENV=local
APP_PORT=8086
LOG_FORMAT=text
OTEL_ENABLED=false
DB_PASSWORD=password
REDIS_ADDR=localhost:6379
USER_JWT_SECRET=local-user-jwt-secret
REFRESH_TOKEN_SECRET=local-refresh-token-secret
DUKCAPIL_BASE_URL=http://localhost:8085
# Akun admin dibuat otomatis saat start (dipakai untuk membuat event)
ADMIN_EMAIL=admin@rusuh.local
ADMIN_PASSWORD=Admin-Local-12345
EOF

cat > queue-service/.env <<'EOF'
APP_ENV=local
APP_PORT=8084
LOG_FORMAT=text
OTEL_ENABLED=false
DB_PASSWORD=password
REDIS_ADDR=localhost:6379
ACCESS_TOKEN_SECRET=local-access-token-secret
KAFKA_BROKERS=localhost:29092
EOF

cat > event-service/.env <<'EOF'
APP_ENV=local
APP_PORT=8082
LOG_FORMAT=text
OTEL_ENABLED=false
DB_PASSWORD=password
ACCESS_TOKEN_SECRET=local-access-token-secret
USER_JWT_SECRET=local-user-jwt-secret
KAFKA_BROKERS=localhost:29092
VALKEY_ADDR=localhost:6379
EOF

cat > booking-service/.env <<'EOF'
APP_ENV=local
APP_PORT=8081
LOG_FORMAT=text
OTEL_ENABLED=false
DB_PASSWORD=password
VALKEY_ADDR=localhost:6379
ACCESS_TOKEN_SECRET=local-access-token-secret
PAYMENT_TOKEN_SECRET=local-payment-token-secret
USER_JWT_SECRET=local-user-jwt-secret
KAFKA_BROKERS=localhost:29092
DUKCAPIL_BASE_URL=http://localhost:8085
# false = NIK benar-benar dicocokkan ke dukcapil-mock (true = lolos bila dukcapil mati)
DUKCAPIL_FAIL_OPEN=false
EVENT_SERVICE_URL=http://localhost:8082
# Opsional: server key SANDBOX Midtrans Anda (untuk meng-expire transaksi saat cancel)
MIDTRANS_SERVER_KEY=
MIDTRANS_ENV=sandbox
EOF

cat > payment-service/.env <<'EOF'
APP_ENV=local
APP_PORT=8083
LOG_FORMAT=text
OTEL_ENABLED=false
DB_PASSWORD=password
VALKEY_ADDR=localhost:6379
PAYMENT_TOKEN_SECRET=local-payment-token-secret
ACCESS_TOKEN_SECRET=local-access-token-secret
KAFKA_BROKERS=localhost:29092
# Kosong = mode lokal tanpa Midtrans (lihat langkah 7). Isi server key SANDBOX untuk Snap sungguhan.
MIDTRANS_SERVER_KEY=
MIDTRANS_ENV=sandbox
EOF

cat > worker-service/.env <<'EOF'
APP_ENV=local
APP_PORT=8080
LOG_FORMAT=text
OTEL_ENABLED=false
KAFKA_BROKERS=localhost:29092
# Mailpit lokal. Username/password dummy tapi WAJIB diisi.
EMAIL_HOST=localhost
EMAIL_PORT=1025
EMAIL_USERNAME=noreply@rusuh.local
EMAIL_PASSWORD=local
EMAIL_FROM=Rusuh Booking Tiket <noreply@rusuh.local>
EOF
```

---

## 5. Jalankan 7 service

**Opsi A — satu perintah (background, log ke file):**

```bash
mkdir -p bin logs
for r in dukcapil-mock user-service queue-service event-service booking-service payment-service worker-service; do
  (cd $r && go build -o ../bin/$r ./cmd) && echo "built $r"
done
: > pids
for r in dukcapil-mock user-service queue-service event-service booking-service payment-service worker-service; do
  (cd $r && nohup ../bin/$r > ../logs/$r.log 2>&1 & echo $! >> ../pids)   # jalankan DARI folder repo agar .env terbaca
  sleep 1
done
```

Hentikan semua: `kill $(cat pids)`.

**Opsi B — satu terminal per service** (enak untuk melihat log): di tiap folder repo jalankan `go run ./cmd`,
urutan: dukcapil-mock → user-service → queue-service → event-service → booking-service → payment-service → worker-service.

Cek semua hidup:

```bash
for p in 8085 8086 8084 8082 8081 8083; do curl -s localhost:$p/sys/ping; echo; done   # 6x "status":"00000"
curl -s -o /dev/null -w "worker %{http_code}\n" localhost:8080/healthz                # worker 200
grep -i "akun admin" logs/user-service.log                                            # admin dibuat
```

Satu WARN normal di booking-service bila `MIDTRANS_SERVER_KEY` kosong:
`MIDTRANS_SERVER_KEY kosong: cancel tidak meng-expire transaksi Midtrans`.

---

## 6. Smoke test end-to-end

Jalankan berurutan di folder `rusuh-local` (variabel shell dipakai antar langkah).

> **Access Token antrian hanya berlaku ±5 menit** (TTL booking). Bila mendapat
> `401 "Access Token tidak valid"` atau `410`, ulangi `join` (6.2) untuk token baru.

### 6.1 Admin membuat event + kategori

```bash
ADMIN=$(curl -s -X POST localhost:8086/auth/login -H 'Content-Type: application/json' \
  -d '{"email":"admin@rusuh.local","password":"Admin-Local-12345"}' | jq -r .data.authToken)

EVENT_ID=$(curl -s -X POST localhost:8082/api/event/v1/admin/create-event \
  -H "Authorization: Bearer $ADMIN" -H 'Content-Type: application/json' \
  -d '{"name":"Konser Lokal Demo","description":"Event uji lokal","status":"on_sale"}' | jq -r .data.id)

# stok kecil (6) supaya antrian terlihat: booking room = floor(6/3) = 2 orang
CAT_ID=$(curl -s -X POST "localhost:8082/api/event/v1/admin/$EVENT_ID/create-category" \
  -H "Authorization: Bearer $ADMIN" -H 'Content-Type: application/json' \
  -d '{"name":"Festival","price":500000,"capacity":6}' | jq -r .data.id)

echo "event=$EVENT_ID category=$CAT_ID"
curl -s localhost:8082/api/event/v1/list | jq -c '.data[] | {id,name,status}'
```

> Event berstatus `draft` tidak muncul di list publik — buat dengan `"status":"on_sale"`.

### 6.2 Waiting room: 3 orang join, booking room muat 2

```bash
for i in 1 2 3; do
  curl -s -X POST "localhost:8084/api/queue/v1/event/$EVENT_ID/join" > join$i.json
  jq -c '.data | {status, position, queueToken}' join$i.json
done
# → ADMITTED, ADMITTED, QUEUED position 1

ACCESS1=$(jq -r .data.accessToken join1.json)
ACCESS2=$(jq -r .data.accessToken join2.json)
Q3=$(jq -r .data.queueToken join3.json)

# SSE posisi antrian orang ke-3 (5 detik)
curl -s -N -m 5 "localhost:8084/api/queue/v1/event/$EVENT_ID/$Q3/stream"
# event: position  data: {"position":1,"etaSeconds":30}   lalu heartbeat ": ping"
```

### 6.3 Detail tiket + stok realtime (SSE)

```bash
curl -s "localhost:8082/api/event/v1/detail/$EVENT_ID" -H "X-Access-Token: $ACCESS1" \
  | jq -c '.data | {name, categories: [.categories[] | {id,name,available}]}'

curl -s -N -m 3 "localhost:8081/api/booking/v1/event/$EVENT_ID/stock/stream" -H "X-Access-Token: $ACCESS1"
# event: stock  data: {"eventId":...,"categories":[{"name":"Festival","available":6,...}]}
```

### 6.4 User login (opsional) + validasi NIK

```bash
curl -s -X POST localhost:8086/auth/register -H 'Content-Type: application/json' \
  -d '{"email":"budi@example.com","password":"rahasia123","name":"Budi"}' | jq -c '{status,message}'
USER=$(curl -s -X POST localhost:8086/auth/login -H 'Content-Type: application/json' \
  -d '{"email":"budi@example.com","password":"rahasia123"}' | jq -r .data.authToken)

# cocok → valid:true ; nama salah → valid:false
curl -s -X POST localhost:8086/api/user/v1/nik/verify -H 'Content-Type: application/json' \
  -d '{"nik":"3273000000000001","nama":"Budi Santoso","tglLahir":"1990-01-01","namaIbu":"Siti Aminah"}' | jq -c .data
curl -s -X POST localhost:8086/api/user/v1/member/create -H "Authorization: Bearer $USER" -H 'Content-Type: application/json' \
  -d '{"nik":"3273000000000001","nama":"Budi Santoso","tglLahir":"1990-01-01","namaIbu":"Siti Aminah"}' | jq -c '{status,message}'
curl -s localhost:8086/api/user/v1/member/list -H "Authorization: Bearer $USER" | jq -c '.data.items[] | {nik,nama}'
```

### 6.5 Booking (hold stok)

```bash
# NIK dengan data salah → 400 "NIK tidak valid"
curl -s -X POST localhost:8081/api/booking/v1/create -H "X-Access-Token: $ACCESS1" -H 'Content-Type: application/json' \
  -d "{\"eventId\":\"$EVENT_ID\",\"email\":\"budi@example.com\",\"items\":[{\"categoryId\":\"$CAT_ID\",\"attendees\":[
       {\"nik\":\"3273000000000001\",\"nama\":\"Salah Nama\",\"tglLahir\":\"1990-01-01\",\"namaIbu\":\"Siti Aminah\"}]}]}"

# 2 tiket (1 tiket = 1 NIK), login → user_id tersimpan
curl -s -X POST localhost:8081/api/booking/v1/create -H "X-Access-Token: $ACCESS1" -H "Authorization: Bearer $USER" \
  -H 'Content-Type: application/json' \
  -d "{\"eventId\":\"$EVENT_ID\",\"email\":\"budi@example.com\",\"items\":[{\"categoryId\":\"$CAT_ID\",\"attendees\":[
       {\"nik\":\"3273000000000001\",\"nama\":\"Budi Santoso\",\"tglLahir\":\"1990-01-01\",\"namaIbu\":\"Siti Aminah\"},
       {\"nik\":\"3273000000000002\",\"nama\":\"Ani Lestari\",\"tglLahir\":\"1992-05-17\",\"namaIbu\":\"Rina Marlina\"}]}]}" > b1.json
jq -c '.data | {bookingId,status,total,paymentExpiresAt}' b1.json      # status HELD, total 1000000
B1=$(jq -r .data.bookingId b1.json); PT1=$(jq -r .data.paymentToken b1.json)

docker exec rusuh-local-postgres psql -U postgres -d rusuh_booking -Atc \
  "select status, user_id is not null from booking where id='$B1'; select available from ticket_category where id='$CAT_ID';"
# HELD|t   lalu   4
```

Cek Mailpit http://localhost:8025 → email **"Instruksi Pembayaran"** untuk budi@example.com.

### 6.6 Cancel → stok kembali, antrian maju

```bash
# pantau SSE orang ke-3 di background
curl -s -N -m 12 "localhost:8084/api/queue/v1/event/$EVENT_ID/$Q3/stream" > sse3.txt &
sleep 2

curl -s -X POST "localhost:8081/api/booking/v1/$B1/cancel" -H "X-Payment-Token: $PT1" | jq -c .data   # CANCELLED
curl -s -X POST "localhost:8081/api/booking/v1/$B1/cancel" -H "X-Payment-Token: $PT1" | jq -c .data   # idempoten

# token antrian lama dicabut → harus antri ulang
curl -s -X POST localhost:8081/api/booking/v1/create -H "X-Access-Token: $ACCESS1" -H 'Content-Type: application/json' \
  -d "{\"eventId\":\"$EVENT_ID\",\"email\":\"budi@example.com\",\"items\":[{\"categoryId\":\"$CAT_ID\",\"attendees\":[
       {\"nik\":\"3273000000000003\",\"nama\":\"Candra Wijaya\",\"tglLahir\":\"1988-11-30\",\"namaIbu\":\"Dewi Kartika\"}]}]}"
# {"status":"410","message":"Waktu booking telah habis, silakan mulai ulang"}

sleep 10; grep "^event:" sse3.txt      # event: position  →  event: admitted  (orang ke-3 masuk lewat Kafka slot.released)
docker exec rusuh-local-postgres psql -U postgres -d rusuh_booking -Atc "select available from ticket_category where id='$CAT_ID';"  # 6
```

---

## 7. Pembayaran

### 7a. Tanpa Midtrans (sepenuhnya offline)

Dengan `MIDTRANS_SERVER_KEY` kosong, `POST /api/payment/v1/charge` gagal (500, tidak bisa memanggil Midtrans) dan
**verifikasi signature callback dilewati**. Kita buat baris payment `PENDING` secara manual (yang biasanya dibuat
`charge`), lalu mensimulasikan webhook Midtrans:

```bash
curl -s -X POST localhost:8081/api/booking/v1/create -H "X-Access-Token: $ACCESS2" -H 'Content-Type: application/json' \
  -d "{\"eventId\":\"$EVENT_ID\",\"email\":\"candra@example.com\",\"items\":[{\"categoryId\":\"$CAT_ID\",\"attendees\":[
       {\"nik\":\"3273000000000003\",\"nama\":\"Candra Wijaya\",\"tglLahir\":\"1988-11-30\",\"namaIbu\":\"Dewi Kartika\"}]}]}" > b2.json
B2=$(jq -r .data.bookingId b2.json)

docker exec rusuh-local-postgres psql -U postgres -d rusuh_booking -c \
  "insert into payment (id, booking_id, status, provider_ref) values (gen_random_uuid(), '$B2', 'PENDING', '$B2');"

curl -s -X POST localhost:8083/api/payment/v1/callback -H 'Content-Type: application/json' \
  -d "{\"order_id\":\"$B2\",\"transaction_status\":\"settlement\",\"status_code\":\"200\",\"gross_amount\":\"500000.00\",\"payment_type\":\"bank_transfer\"}"
# {"status":"00000","message":"OK","data":{"received":true}}

sleep 5
docker exec rusuh-local-postgres psql -U postgres -d rusuh_booking -Atc \
  "select b.status, p.status, p.method from booking b join payment p on p.booking_id=b.id where b.id='$B2';"
# PAID|SUCCESS|bank_transfer
curl -s localhost:8025/api/v1/messages | jq -r '.messages[] | "\(.To[0].Address) | \(.Subject)"'
# candra@example.com | [Rusuh Booking] E-Tiket Resmi — Konser Lokal Demo
```

`gross_amount` harus sama dengan total booking (`500000.00`), kalau tidak callback ditolak.
Kirim callback yang sama dua kali → tetap `OK` tanpa efek (idempoten via `payment_callback_log`).

### 7b. Dengan Midtrans Sandbox (opsional)

1. Daftar di <https://dashboard.sandbox.midtrans.com>, ambil **Server Key** sandbox (Settings → Access Keys).
2. Isi `MIDTRANS_SERVER_KEY=<server key sandbox>` di `payment-service/.env` **dan** `booking-service/.env`, restart keduanya.
3. `POST /api/payment/v1/charge` (header `X-Payment-Token`, body `{"bookingId":"..."}`) → `data.redirectUrl`;
   buka di browser, bayar dengan simulator sandbox.
4. Webhook Midtrans tidak bisa menjangkau `localhost`. Pilihan: ekspos port 8083 lewat tunnel (mis. ngrok) dan set
   *Payment Notification URL* di dashboard ke `https://<tunnel>/api/payment/v1/callback`, **atau** kirim callback
   sendiri dengan signature yang benar:
   ```bash
   SIG=$(printf '%s' "${BOOKING_ID}200${GROSS}${MIDTRANS_SERVER_KEY}" | shasum -a 512 | cut -d' ' -f1)
   # body: order_id=$BOOKING_ID, status_code="200", gross_amount=$GROSS (mis. "500000.00"), signature_key=$SIG
   ```

> Bagian 7b belum diuji ulang end-to-end di panduan ini (butuh akun sandbox milik pembaca).

### 7c. Cancel setelah Payment Page dibuka

Cancel tetap diizinkan saat payment masih `PENDING`: booking → `CANCELLED`, payment → `EXPIRED`, transaksi Midtrans
di-expire (bila server key diisi). Callback `settlement` yang datang terlambat **tidak** menerbitkan e-tiket
(log `payment settled for closed booking, needs manual refund (CS)` di payment-service).

---

## 8. Melihat isi sistem

| Yang dilihat | Cara |
|---|---|
| Email | http://localhost:8025 (Mailpit) |
| Riwayat tiket user | `curl -s localhost:8086/api/user/v1/ticket-history -H "Authorization: Bearer $USER" \| jq` |
| Pesan Kafka | `docker exec -it rusuh-local-kafka kafka-console-consumer --bootstrap-server localhost:9092 --topic booking.created --from-beginning` (ganti topic: `stock.changed`, `slot.released`, `payment.succeeded`, ...) |
| Antrian di Redis | `docker exec rusuh-local-redis redis-cli --scan --pattern "*$EVENT_ID*"` lalu `ZRANGE "queue:{$EVENT_ID}" 0 -1 WITHSCORES` |
| Database | `docker exec -it rusuh-local-postgres psql -U postgres -d rusuh_booking` |
| Log service | `tail -f logs/<service>.log` |

---

## 9. Troubleshooting

| Gejala | Penyebab | Solusi |
|---|---|---|
| Service langsung exit `required env not set key=ACCESS_TOKEN_SECRET` | `.env` tidak terbaca (dijalankan dari folder lain) atau komentar pakai `//` | Jalankan dari folder repo; komentar `.env` harus `#` |
| Admin event-service 401/403 | `USER_JWT_SECRET` beda antara user-service dan event-service | Samakan nilainya, restart |
| Detail/booking 401 "Access Token tidak valid" | `ACCESS_TOKEN_SECRET` beda antar service | Samakan di queue/event/booking/payment |
| Queue join 404 "event tidak ditemukan" | Event belum punya kategori tiket | Buat kategori dulu (6.1) |
| Tidak ada email / antrian tidak maju setelah cancel | Kafka belum siap atau `KAFKA_BROKERS` salah | Pastikan `localhost:29092`, cek `docker compose ps`; jalankan worker **sebelum** membuat booking |
| `listen tcp :808x: address already in use` | Proses lama masih jalan | `lsof -nP -iTCP:808x -sTCP:LISTEN` lalu hentikan prosesnya |
| 429 saat tes berulang | Rate limit (join 30/menit/IP, login 10/menit) | Tunggu 1 menit, atau set `QUEUE_JOIN_RATE_PER_MIN=0` / `LOGIN_RATE_PER_IP_PER_MIN=0` di `.env` |
| Detail/booking 401 setelah beberapa menit | Access Token antrian kedaluwarsa (±5 menit) | `join` ulang untuk token baru |
| Booking ditolak "1 NIK hanya boleh memiliki 1 tiket per event" | NIK sudah dipakai di booking aktif event ini | Pakai NIK seed lain, atau cancel booking sebelumnya |
| Tabel kurang / migrasi tidak jalan | Volume Postgres sudah ada dari run sebelumnya | `docker compose down -v && docker compose up -d` |

---

## 10. Berhenti & reset

```bash
kill $(cat pids)            # hentikan 7 service (opsi A)
docker compose down         # hentikan infrastruktur, data tetap
docker compose down -v      # hapus semua data (DB, Kafka) — mulai dari nol
```
