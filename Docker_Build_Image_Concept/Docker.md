# Docker - ASP.NET API Notes

## Docker Resources

Docker memiliki beberapa resource utama yang berdiri sendiri (sejajar):

- Image
- Container
- Volume
- Network

Container **menggunakan** Volume, bukan **memiliki** Volume.

---

# Lifecycle Docker

Semua aplikasi Docker mengikuti alur berikut:

Source Code
↓
Dockerfile
↓
docker build
↓
Image
↓
docker run / docker compose up
↓
Container

---

# Image

Image adalah blueprint/template untuk membuat Container.

Sifat Image:

- Immutable (tidak berubah)
- Read Only
- Bisa dibuat berkali-kali
- Bisa menghasilkan banyak Container

Contoh:

mysql:8.0

atau

reimbursement-api:1.0

Image **bukan aplikasi yang sedang berjalan**.

Image hanya template.

---

# Container

Container adalah instance yang dibuat dari Image.

Container:

- Bisa Start
- Bisa Stop
- Bisa Restart
- Bisa Delete

Satu Image dapat menghasilkan banyak Container.

Contoh:

Image
mysql:8.0

↓

Container A
mysql-dev

↓

Container B
mysql-testing

---

# Volume

Volume adalah tempat penyimpanan data yang persistent.

Volume berada di luar filesystem Container.

Container hanya melakukan mount ke Volume.

Contoh:

mysql_data

↓

/var/lib/mysql

Artinya:

MySQL tetap membaca:

/var/lib/mysql

Tetapi data sebenarnya disimpan di Volume.

Keuntungan:

- Hapus Container → Data tetap ada
- Buat Container baru → Data masih ada

---

# Dockerfile

Dockerfile adalah resep untuk membuat Image.

Docker membaca Dockerfile dari atas ke bawah.

Contoh isi Dockerfile nanti:

- Base Image
- Copy Source Code
- Build Project
- Publish Project
- Menjalankan aplikasi

Dockerfile TIDAK membuat Container.

Dockerfile hanya membuat Image.

---

# docker build

Perintah:

docker build

Fungsi:

Membuat Image dari Dockerfile.

Alur:

Source Code
↓

Dockerfile
↓

docker build
↓

Image

---

# docker run

Fungsi:

Membuat Container dari Image.

Image
↓

docker run
↓

Container

---

# Docker Compose

Docker Compose digunakan untuk mengelola banyak Container sekaligus.

Contoh:

- API
- MySQL
- Redis
- Nginx

cukup dijalankan:

docker compose up -d

---

# image vs build

Jika menggunakan image:

services:
  mysql:
    image: mysql:8.0

Artinya:

Gunakan Image yang sudah ada.

Bisa dari:

- Docker Hub
- Registry
- Local Image

---

Jika menggunakan build:

services:
  api:
    build: .

Artinya:

Bangun Image menggunakan Dockerfile.

Jika Image belum ada,
Compose akan melakukan build terlebih dahulu.

---

# Compose Build Behavior

docker compose up -d

Jika Image belum ada:

→ Build Image
→ Jalankan Container

Jika Image sudah ada:

→ Gunakan Image lama
→ Tidak Build lagi

Jika Source Code berubah:

Gunakan:

docker compose up --build -d

atau

docker compose build

baru

docker compose up -d

---

# Development Flow

Edit Source Code

↓

docker compose up --build -d

↓

Image Baru

↓

Container Baru

---

# Production Flow

Laptop

↓

docker build

↓

Image

↓

docker push

↓

Docker Registry

↓

VPS

↓

docker pull

↓

docker compose up -d

Server Production biasanya tidak melakukan build dari Source Code.

Server hanya menjalankan Image yang sudah dibuat.

---

# MySQL vs ASP.NET API

MySQL

Docker Hub
↓

mysql:8.0 (Image)
↓

Container

Image dibuat oleh tim MySQL.

---

ASP.NET API

Source Code
↓

Dockerfile
↓

docker build
↓

reimbursement-api:1.0 (Image)
↓

Container

Image dibuat sendiri.

---

# Konsep yang Harus Selalu Diingat

Dockerfile
=
Resep membuat Image

Image
=
Blueprint / Template

Container
=
Aplikasi yang sedang berjalan

Volume
=
Penyimpanan Data

Compose
=
Mengatur seluruh environment aplikasi

---

# Target Akhir Project Reimbursement

docker compose up -d

↓

API Container

+

MySQL Container

+

Volume mysql_data

+

Docker Network

↓

Aplikasi berjalan tanpa perlu install .NET SDK maupun MySQL secara manual.
