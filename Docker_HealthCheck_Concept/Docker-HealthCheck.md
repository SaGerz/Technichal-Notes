# Docker Healthcheck

## 1. Pengertian Healthcheck

Healthcheck adalah mekanisme Docker untuk memeriksa kondisi kesehatan sebuah container atau aplikasi di dalamnya melalui perintah pengujian yang ditentukan.

Healthcheck membantu membedakan container yang sekadar berjalan dengan container yang sudah berhasil melewati pemeriksaan kesehatan.

- `running`: proses utama container sedang berjalan.
- `starting`: Healthcheck belum berhasil atau masih berada pada tahap awal pemeriksaan.
- `healthy`: pemeriksaan kesehatan berhasil.
- `unhealthy`: pemeriksaan kesehatan gagal berulang kali hingga melewati batas kegagalan.

Status `healthy` tidak menjamin seluruh fungsi aplikasi berjalan sempurna. Status tersebut hanya menunjukkan bahwa pemeriksaan yang dikonfigurasi berhasil.

## 2. Masalah yang Diselesaikan

Dalam aplikasi yang terdiri dari beberapa service, service yang dibutuhkan belum tentu langsung siap digunakan ketika containernya mulai berjalan.

Contoh pada aplikasi Reimbursement:

- MySQL berfungsi sebagai database.
- ASP.NET Core API membutuhkan koneksi ke MySQL.

MySQL dapat membutuhkan waktu untuk menginisialisasi database. Sementara itu, API bisa mulai berjalan dan mencoba terhubung ke MySQL sebelum database siap.

Jika koneksi gagal dan aplikasi tidak memiliki mekanisme retry, proses API dapat berhenti.

Healthcheck membantu mendeteksi kesiapan MySQL sebelum API dimulai.

## 3. Konfigurasi Healthcheck MySQL

Contoh konfigurasi pada `compose.yaml`:

```yaml
services:
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: reimbursement_db
    healthcheck:
      test: ["CMD-SHELL", "mysqladmin ping -h 127.0.0.1 -uroot -p\"$${MYSQL_ROOT_PASSWORD}\" --silent"]
      interval: 5s
      timeout: 5s
      retries: 10
      start_period: 30s
```

Konfigurasi di atas merupakan bagian service MySQL, bukan keseluruhan file Compose. Sesuaikan dengan konfigurasi project yang sudah ada.

### Penjelasan properti

#### `test`

Menentukan perintah yang digunakan untuk memeriksa kesehatan service.

- `CMD-SHELL`: menjalankan perintah melalui shell di dalam container.
- `mysqladmin ping`: memeriksa respons server MySQL.
- `-h 127.0.0.1`: memeriksa server MySQL pada container itu sendiri.
- `-uroot`: menentukan user yang digunakan.
- `-p...`: memberikan password.
- `--silent`: mengurangi output yang tidak diperlukan.

Catatan: `mysqladmin ping` terutama memeriksa apakah server MySQL merespons. Tes ini tidak menjamin seluruh query aplikasi atau akses ke database tertentu pasti berhasil.

#### `interval`

Mengatur interval pemeriksaan berkala.

```yaml
interval: 5s
```

Docker menjadwalkan pemeriksaan setiap lima detik sesuai mekanisme Healthcheck.

#### `timeout`

Mengatur batas waktu maksimal untuk satu pemeriksaan.

```yaml
timeout: 5s
```

Jika pemeriksaan tidak selesai dalam lima detik, tes dianggap gagal.

#### `retries`

Mengatur jumlah kegagalan berturut-turut sebelum container dinyatakan `unhealthy`.

```yaml
retries: 10
```

Nilai ini bukan jumlah percobaan untuk memperbaiki atau me-restart aplikasi.

#### `start_period`

Memberikan waktu awal untuk inisialisasi service.

```yaml
start_period: 30s
```

Kegagalan pemeriksaan selama masa awal ini tidak langsung dihitung sebagai kegagalan yang menentukan status `unhealthy`, selama masih mengikuti aturan Healthcheck Docker.

`start_period` bukan waktu tunggu wajib selama 30 detik. Jika pemeriksaan berhasil, container bisa menjadi `healthy` lebih awal.

## 4. Fungsi Tanda Dollar Ganda (`$$`)

Contoh:

```yaml
-p\"$${MYSQL_ROOT_PASSWORD}\"
```

Docker Compose dan shell di dalam container sama-sama dapat memproses variabel environment.

Penggunaan `$$` meminta Compose meneruskan tanda `$` secara literal agar shell di dalam container yang membaca variabel tersebut.

Alurnya:

1. Docker Compose membaca file YAML.
2. `$$` diteruskan sebagai `$` literal.
3. Shell di dalam container menerima `$MYSQL_ROOT_PASSWORD`.
4. Shell mengambil nilai variabel dari environment container.

Penggunaan `$$` menghindari Compose mencoba melakukan interpolasi variabel tersebut pada sisi host.

## 5. Hubungan Healthcheck dengan `depends_on`

### Tanpa Healthcheck

```yaml
depends_on:
  - mysql
```

Konfigurasi ini mengatur urutan startup service. MySQL dimulai sebelum API, tetapi Compose tidak menunggu sampai MySQL siap menerima koneksi.

### Dengan Healthcheck

```yaml
depends_on:
  mysql:
    condition: service_healthy
```

Konfigurasi ini membuat Compose menunggu sampai Healthcheck MySQL berhasil sebelum memulai API.

Alur startup:

1. Compose memulai MySQL.
2. Docker menjalankan Healthcheck MySQL.
3. MySQL melewati pemeriksaan dan berstatus `healthy`.
4. Compose memulai API.
5. API mencoba terhubung ke MySQL.

Penting: `service_healthy` mengatur kesiapan pada saat startup. Ini bukan jaminan bahwa MySQL akan selalu sehat selama API berjalan.

Jika MySQL menjadi `unhealthy` setelah startup, Compose tidak otomatis menghentikan API berdasarkan kondisi ini.

## 6. Hubungan Healthcheck dengan Restart Policy

Contoh:

```yaml
restart: unless-stopped
```

Restart policy mengatur perilaku Docker ketika proses utama container berhenti.

Healthcheck dan restart policy memiliki fungsi berbeda:

- Healthcheck: mendeteksi dan melaporkan kondisi kesehatan.
- Restart policy: menentukan perilaku ketika proses utama container berhenti.

`unless-stopped` memungkinkan Docker mencoba menjalankan kembali container yang berhenti, kecuali container tersebut dihentikan secara manual dan tetap dibiarkan berhenti.

Docker tidak otomatis me-restart container hanya karena Healthcheck berstatus `unhealthy`.

Jika aplikasi terus gagal ketika dimulai, restart policy saja tidak menyelesaikan akar masalahnya.

## 7. Perintah untuk Memeriksa Healthcheck

### Memeriksa status service

```bash
docker compose ps
```

### Memeriksa status kesehatan MySQL

```bash
docker inspect --format='{{.State.Health.Status}}' mysql-reimburstment
```

### Melihat hasil pemeriksaan Healthcheck

```bash
docker inspect --format='{{json .State.Health}}' mysql-reimburstment
```

### Melihat log API

```bash
docker logs --tail 100 reimbursement-api
```

### Memeriksa status container dan exit code

```bash
docker inspect --format='{{.State.Status}} | ExitCode={{.State.ExitCode}} | Error={{.State.Error}}' reimbursement-api
```

## 8. Kesimpulan

Healthcheck membantu Docker menentukan apakah suatu service sudah melewati pemeriksaan kesehatan yang ditentukan.

Dalam aplikasi Reimbursement:

- Healthcheck digunakan untuk memeriksa kesiapan MySQL.
- `depends_on` dengan `condition: service_healthy` menunda startup API sampai MySQL dinyatakan sehat.
- `restart: unless-stopped` membantu menjalankan kembali container ketika proses utamanya berhenti, sesuai aturan restart policy.
- Retry koneksi di aplikasi tetap diperlukan untuk menangani kondisi database yang belum siap atau terputus setelah startup.

Healthcheck, dependency startup, dan restart policy adalah tiga mekanisme berbeda yang dapat digunakan bersama untuk meningkatkan keandalan aplikasi berbasis Docker.
