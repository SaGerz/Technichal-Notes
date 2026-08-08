# Docker Networking

## 1. Apa Itu Docker Network?

Docker Network adalah mekanisme yang digunakan Docker untuk memungkinkan container saling berkomunikasi.

Secara sederhana, resource utama Docker dapat dibayangkan seperti:

```text
Docker
├── Images
├── Containers
├── Networks
└── Volumes
```

Masing-masing memiliki fungsi berbeda:

```text
Image
→ Template/hasil build yang digunakan untuk membuat container.

Container
→ Instance yang menjalankan image.

Network
→ Jalur komunikasi antar-container.

Volume
→ Tempat penyimpanan data persistent yang berada di luar lifecycle container.
```

---

## 2. Docker Compose dan Default Network

Ketika menggunakan Docker Compose:

```bash
docker compose up -d
```

Docker Compose secara default akan membuat sebuah network untuk project Compose tersebut.

Misalnya project Compose bernama:

```text
reimbursement_api
```

Maka Docker biasanya membuat:

```text
reimbursement_api_default
```

Misalnya `docker-compose.yaml` memiliki:

```yaml
services:
  api:
    ...

  mysql:
    ...
```

Maka secara default kedua service tersebut akan masuk ke network yang sama:

```text
                reimbursement_api_default
              ┌─────────────────────────────┐
              │                             │
              │      API Container          │
              │                             │
              │      MySQL Container        │
              │                             │
              └─────────────────────────────┘
```

### Penting

Network tersebut **tidak dibuat karena API memiliki Connection String ke MySQL**.

Network dibuat karena Docker Compose membuat default network untuk Compose project tersebut.

Connection String baru digunakan oleh aplikasi untuk melakukan koneksi ke MySQL melalui network tersebut.

---

## 3. `docker network ls`

Untuk melihat seluruh Docker Network:

```bash
docker network ls
```

Contoh:

```text
NETWORK ID     NAME                       DRIVER    SCOPE
76f3a45ecd20   reimbursement_api_default  bridge    local
```

`docker network ls` hanya menampilkan **network**.

Bukan image dan bukan container.

Perbandingan command:

```bash
docker image ls
```

→ Melihat image.

```bash
docker ps -a
```

→ Melihat container.

```bash
docker network ls
```

→ Melihat network.

```bash
docker volume ls
```

→ Melihat volume.

Jadi jika menjalankan:

```bash
docker network ls
```

kita tidak akan menemukan:

```text
mysql-reimburstment
```

karena `mysql-reimburstment` adalah **container**, bukan network.

Yang akan terlihat adalah network seperti:

```text
reimbursement_api_default
```

---

## 4. `docker network inspect`

Untuk melihat detail sebuah network:

```bash
docker network inspect reimbursement_api_default
```

Command ini memberikan informasi mengenai network tersebut, misalnya:

* Name
* ID
* Driver
* Subnet
* Gateway
* Containers yang sedang terhubung

Jika container sedang aktif, bagian `Containers` dapat berisi container yang terhubung ke network tersebut.

Contoh konsep:

```text
reimbursement_api_default
│
├── reimbursement-api
│
└── mysql-reimburstment
```

Namun jika container sedang berhenti, kita bisa mendapatkan:

```json
"Containers": {}
```

Hal ini karena container yang berhenti tidak memiliki network endpoint aktif.

---

## 5. `docker inspect` vs `docker network inspect`

Keduanya sama-sama menggunakan konsep `inspect`, tetapi objek yang diperiksa berbeda.

### `docker inspect <container>`

Contoh:

```bash
docker inspect reimbursement-api
```

Fokusnya adalah **container**.

Pertanyaannya:

> "Bagaimana detail container ini?"

Informasi yang bisa ditemukan antara lain:

```text
Container
├── ID
├── Image
├── Command
├── Environment
├── Mounts
├── Ports
├── NetworkSettings
├── Networks
└── dll
```

Jadi:

```bash
docker inspect reimbursement-api
```

berarti kita melihat konfigurasi dan kondisi container `reimbursement-api`.

---

### `docker network inspect <network>`

Contoh:

```bash
docker network inspect reimbursement_api_default
```

Fokusnya adalah **network**.

Pertanyaannya:

> "Bagaimana detail network ini dan container apa saja yang terhubung?"

Informasi yang bisa ditemukan:

```text
Network
├── Name
├── ID
├── Driver
├── Subnet
├── Gateway
└── Containers
```

Jadi konsepnya:

```text
docker inspect <container>
        ↓
Fokus ke CONTAINER

docker network inspect <network>
        ↓
Fokus ke NETWORK
```

---

## 6. Container dan Network yang Sama

Setelah menjalankan:

```bash
docker compose up -d
```

kita dapat mengecek container:

```bash
docker inspect reimbursement-api
```

dan:

```bash
docker inspect mysql-reimburstment
```

Keduanya dapat memiliki:

```text
NetworkID:
76f3a45ecd20...
```

Jika `NetworkID` keduanya sama, berarti kedua container berada pada network yang sama.

Konsepnya:

```text
                 reimbursement_api_default
              ┌─────────────────────────────┐
              │                             │
              │  reimbursement-api          │
              │                             │
              │  mysql-reimburstment        │
              │                             │
              └─────────────────────────────┘
```

---

## 7. Docker Network Tidak Bergantung pada Connection String

Misalnya API memiliki Connection String:

```text
Server=mysql;
Port=3306;
Database=reimbursement_db;
User=root;
Password=root;
```

Connection String tersebut **bukan penyebab network dibuat**.

Urutan sebenarnya:

```text
docker compose up -d
        │
        ▼
Docker Compose membuat Network
        │
        ▼
API dan MySQL masuk ke Network
        │
        ▼
API menjalankan aplikasi
        │
        ▼
API menggunakan Connection String
        │
        ▼
API mencari host "mysql"
        │
        ▼
Docker DNS mencari container MySQL
        │
        ▼
API terhubung ke MySQL
```

Jadi:

```text
Network
=
Infrastruktur/jalur komunikasi Docker

Connection String
=
Konfigurasi aplikasi untuk menggunakan jalur tersebut
```

---

## 8. Service Name Sebagai Hostname

Misalnya Compose memiliki:

```yaml
services:

  api:
    ...

  mysql:
    image: mysql:8.0
```

Service MySQL memiliki nama:

```text
mysql
```

Docker Compose menyediakan service name tersebut sebagai hostname di network.

Maka API dapat menggunakan:

```text
Server=mysql
```

Docker akan menggunakan DNS internal untuk menemukan container MySQL.

Konsepnya:

```text
API Container
     │
     │ Server=mysql
     ▼
Docker DNS
     │
     ▼
MySQL Container
```

Jadi kita tidak perlu mengetahui IP container MySQL secara manual.

---

## 9. Kenapa Tidak Menggunakan `127.0.0.1`?

Ini merupakan salah satu konsep networking Docker yang paling penting.

### Ketika API dijalankan langsung di komputer

Misalnya API dijalankan langsung dari Windows:

```text
Windows
│
├── ASP.NET API
│
└── MySQL
```

Maka:

```text
127.0.0.1:3306
```

dapat mengarah ke MySQL yang berjalan di komputer tersebut.

---

### Ketika API dijalankan di container

Strukturnya menjadi:

```text
Docker
│
├── API Container
│     └── 127.0.0.1
│
└── MySQL Container
      └── 127.0.0.1
```

`127.0.0.1` di dalam API container berarti:

```text
API Container itu sendiri
```

Bukan MySQL container.

Karena setiap container memiliki network environment sendiri.

Jadi:

```text
API
│
└── 127.0.0.1:3306
```

tidak otomatis berarti:

```text
MySQL Container
```

Untuk komunikasi antar-container yang berada di network yang sama, gunakan service name:

```text
mysql:3306
```

---

## 10. Service Name vs Container Name

Misalnya:

```yaml
services:

  mysql:
    container_name: mysql-reimburstment
```

Di sini terdapat dua nama:

```text
Service Name:
mysql

Container Name:
mysql-reimburstment
```

Service name:

```text
mysql
```

merupakan nama yang biasanya digunakan oleh service lain untuk komunikasi di Compose network.

Contoh:

```text
Server=mysql
```

Docker Compose juga dapat memberikan alias seperti:

```text
mysql
mysql-reimburstment
```

sehingga container dapat memiliki beberapa DNS name.

---

## 11. Jangan Hardcode IP Container

Misalnya setelah menjalankan container kita menemukan:

```text
MySQL
IP Address: 172.18.0.3
```

Secara teknis kita bisa saja menggunakan:

```text
Server=172.18.0.3
```

Tetapi ini tidak disarankan.

Alasannya karena IP container dapat berubah ketika container dibuat ulang.

Contoh:

```text
Sebelum:

mysql
↓
172.18.0.3
```

Kemudian container dihapus dan dibuat ulang:

```text
Sesudah:

mysql
↓
172.18.0.5
```

Jika Connection String masih menggunakan:

```text
Server=172.18.0.3
```

maka koneksi dapat rusak.

Karena itu lebih baik:

```text
Server=mysql
```

Docker DNS akan mencari IP container yang benar.

---

## 12. Docker DNS

Docker menyediakan mekanisme DNS internal pada network.

Jika API dan MySQL berada pada network yang sama:

```text
API
│
│ mysql
▼
Docker DNS
│
▼
IP MySQL Container
│
▼
MySQL
```

API tidak perlu mengetahui IP MySQL secara langsung.

API cukup mengetahui:

```text
mysql
```

Docker akan menerjemahkannya menjadi IP container yang sesuai.

Inilah alasan kita dapat menggunakan:

```text
mysql:3306
```

sebagai alamat database di dalam Docker Compose.

---

## 13. Satu Compose Project Bisa Memiliki Banyak Network

Secara default, Compose membuat satu default network:

```text
reimbursement_api_default
```

Namun satu Compose project dapat memiliki beberapa network.

Contoh:

```yaml
services:

  nginx:
    networks:
      - frontend

  api:
    networks:
      - frontend
      - backend

  mysql:
    networks:
      - backend

networks:
  frontend:
  backend:
```

Hasilnya:

```text
              frontend
           ┌─────────────┐
           │             │
         nginx          API
                         │
                         │
                      backend
                   ┌─────┴─────┐
                   │           │
                  API         MySQL
```

Dengan konfigurasi tersebut:

```text
nginx ↔ API
API   ↔ MySQL
```

Tetapi:

```text
nginx ✕ MySQL
```

karena nginx dan MySQL tidak berada pada network yang sama.

---

## 14. Default Network Name

Jika Compose project bernama:

```text
reimbursement_api
```

maka default network biasanya:

```text
reimbursement_api_default
```

Format sederhananya:

```text
<project-name>_default
```

Jadi nama:

```text
reimbursement_api_default
```

tidak berarti kita secara manual membuat network dengan nama tersebut.

Docker Compose yang membuatnya secara otomatis sebagai default network untuk project tersebut.

---

## 15. Eksperimen yang Dilakukan

Untuk melihat network:

```bash
docker network ls
```

Untuk melihat detail network:

```bash
docker network inspect reimbursement_api_default
```

Untuk melihat detail API:

```bash
docker inspect reimbursement-api
```

Untuk melihat detail MySQL:

```bash
docker inspect mysql-reimburstment
```

Jika container belum berjalan, bagian network dapat menunjukkan:

```json
"Containers": {}
```

dan beberapa informasi seperti:

```json
"EndpointID": "",
"Gateway": "",
"IPAddress": ""
```

dapat kosong.

Setelah menjalankan:

```bash
docker compose up -d
```

container menjadi aktif.

Kemudian:

```bash
docker network inspect reimbursement_api_default
```

dapat menunjukkan container yang sedang terhubung ke network tersebut.

---

## 16. Perbedaan Environment Host dan Container

Hal yang harus selalu diingat:

```text
Host
≠
Container
```

Contoh:

```text
Windows
└── localhost
```

berbeda dengan:

```text
API Container
└── localhost
```

Dan berbeda lagi dengan:

```text
MySQL Container
└── localhost
```

Sehingga:

```text
127.0.0.1
```

selalu harus dipahami dari sudut pandang environment yang sedang menjalankannya.

---

## 17. Rule of Thumb

Gunakan pemahaman berikut:

```text
Container
=
Tempat aplikasi berjalan

Image
=
Template/hasil build untuk membuat container

Network
=
Jalur komunikasi antar-container

Volume
=
Penyimpanan data persistent

Docker Compose
=
Definisi dan orchestration beberapa service Docker
```

Untuk komunikasi antar-service dalam satu Compose network:

```text
Gunakan service name
```

Contoh:

```text
mysql:3306
redis:6379
api:8080
```

Hindari menggunakan IP container secara hardcode.

---

## 18. Mental Model Sederhana

Jika memiliki:

```yaml
services:

  api:
    ...

  mysql:
    ...

  redis:
    ...
```

maka secara default:

```text
                 Docker Compose
                       │
                       ▼
             reimbursement_api_default
          ┌────────────┼────────────┐
          │            │            │
          ▼            ▼            ▼
         API         MySQL        Redis
          │            │            │
          │            │            │
          └────────────┴────────────┘
                   Network
```

Service dapat menemukan service lainnya menggunakan nama:

```text
api
mysql
redis
```

Misalnya:

```text
API → mysql:3306
API → redis:6379
```

---

## 19. Kesimpulan

Docker Network memungkinkan container berkomunikasi satu sama lain.

Docker Compose secara default membuat satu network untuk setiap Compose project dan memasukkan service-service di dalam project tersebut ke network tersebut.

Network dibuat oleh Docker Compose, bukan karena adanya Connection String.

Connection String digunakan oleh aplikasi setelah network tersedia.

Untuk komunikasi antar-container:

```text
Gunakan:

mysql:3306
```

bukan:

```text
127.0.0.1:3306
```

karena `127.0.0.1` di dalam container mengacu pada container itu sendiri.

Docker menyediakan DNS internal sehingga service name seperti:

```text
mysql
```

dapat digunakan untuk menemukan container MySQL tanpa perlu mengetahui IP container.

Dengan demikian, konsep utama Docker Networking adalah:

```text
Docker Compose
      │
      ▼
Network
      │
      ├── API
      │
      ├── MySQL
      │
      └── Redis
           │
           ▼
      Service Name
           │
           ▼
      Docker DNS
           │
           ▼
   Container-to-Container Communication
```