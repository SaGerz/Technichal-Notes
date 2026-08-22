# Docker Configuration & Environment

## 1. Apa itu Configuration di ASP.NET Core?

ASP.NET Core memiliki sistem **Configuration** yang menggabungkan konfigurasi dari beberapa sumber.

Contohnya:

- `appsettings.json`
- `appsettings.{Environment}.json`
- Environment Variables
- Command-line arguments
- Configuration provider lainnya

Jadi `appsettings.json` bukan satu-satunya sumber configuration.

Secara sederhana:

```text
Configuration
      │
      ├── appsettings.json
      │
      ├── appsettings.Development.json
      │
      ├── appsettings.Production.json
      │
      └── Environment Variables
```

Semua sumber tersebut kemudian digabung menjadi satu configuration yang dapat dibaca oleh aplikasi.

---

# 2. appsettings.json

Contoh:

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "server=127.0.0.1;port=3306;database=reimbursement_db;user=root;password=root;"
  },
  "Jwt": {
    "Key": "secret",
    "Issuer": "Reimbursement_API",
    "Audience": "Reimbursement_Client",
    "ExpiresMinutes": 60
  }
}
```

Configuration key yang terbentuk antara lain:

```text
ConnectionStrings:DefaultConnection

Jwt:Key
Jwt:Issuer
Jwt:Audience
Jwt:ExpiresMinutes
```

Application dapat mengambil value tersebut melalui:

```csharp
builder.Configuration["Jwt:Key"]
```

atau:

```csharp
builder.Configuration.GetConnectionString("DefaultConnection")
```

---

# 3. appsettings.{Environment}.json

ASP.NET Core juga dapat memiliki configuration khusus berdasarkan environment.

Contohnya:

```text
appsettings.json
appsettings.Development.json
appsettings.Production.json
```

Jika environment aplikasi adalah:

```text
Development
```

maka:

```text
appsettings.json
        ↓
appsettings.Development.json
```

Jika environment:

```text
Production
```

maka:

```text
appsettings.json
        ↓
appsettings.Production.json
```

File environment-specific digunakan untuk memberikan atau mengubah configuration berdasarkan environment aplikasi.

---

# 4. Menentukan ASP.NET Core Environment

Environment ASP.NET Core ditentukan melalui environment variable:

```text
ASPNETCORE_ENVIRONMENT
```

Contoh Docker Compose:

```yaml
environment:
  ASPNETCORE_ENVIRONMENT: Development
```

atau:

```yaml
environment:
  ASPNETCORE_ENVIRONMENT: Production
```

Perhatikan penulisannya:

```text
ASPNETCORE_ENVIRONMENT
```

Bukan:

```text
ASPNETCORE_ENVIROMENT
```

`ENVIROMENT` adalah typo dan tidak dikenali sebagai environment setting standar ASP.NET Core.

---

# 5. Environment Variable

Environment variable adalah configuration yang diberikan melalui environment tempat aplikasi berjalan.

Contoh:

```yaml
environment:
  Jwt__Key: "secret"
```

ASP.NET Core akan membaca:

```text
Jwt__Key
```

sebagai:

```text
Jwt:Key
```

Double underscore (`__`) digunakan untuk merepresentasikan hierarchy configuration.

Contoh:

```text
Jwt__Key
```

menjadi:

```text
Jwt:Key
```

Contoh lainnya:

```text
ConnectionStrings__DefaultConnection
```

menjadi:

```text
ConnectionStrings:DefaultConnection
```

---

# 6. Environment Variable dapat Override appsettings

Misalnya `appsettings.json` memiliki:

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "server=127.0.0.1;port=3306;"
  }
}
```

Kemudian Docker Compose memberikan:

```yaml
environment:
  ConnectionStrings__DefaultConnection: "server=mysql;port=3306;"
```

Maka configuration yang digunakan aplikasi adalah:

```text
server=mysql;port=3306;
```

Bukan:

```text
server=127.0.0.1;port=3306;
```

Karena environment variable memiliki precedence lebih tinggi.

Secara sederhana:

```text
appsettings.json
       ↓
Environment Variables
       ↓
Final Configuration
```

Environment variable dapat digunakan untuk override configuration dari `appsettings`.

---

# 7. Configuration Tidak Harus Berasal dari appsettings

Misalnya kita punya:

```json
{
  "Jwt": {
    "Issuer": "Reimbursement_API",
    "Audience": "Reimbursement_Client",
    "ExpiresMinutes": 60
  }
}
```

Kita tidak wajib menyimpan:

```json
"Key": "secret"
```

di dalam `appsettings.json`.

Kita bisa memberikan:

```yaml
environment:
  Jwt__Key: "secret"
```

Maka hasil configuration ASP.NET Core menjadi:

```text
Jwt
├── Key             ← Environment Variable
├── Issuer          ← appsettings.json
├── Audience        ← appsettings.json
└── ExpiresMinutes  ← appsettings.json
```

Application tetap bisa menggunakan:

```csharp
builder.Configuration["Jwt:Key"]
```

---

# 8. Kenapa Secret Sebaiknya Tidak Hardcode?

Contoh yang kurang baik:

```json
{
  "Jwt": {
    "Key": "super-secret-key"
  }
}
```

Jika `appsettings.json` masuk Git, secret juga ikut masuk repository.

Contoh secret:

- JWT Secret
- Database Password
- API Key
- Payment Gateway Secret Key
- Credential lainnya

Sebaiknya secret diberikan ketika aplikasi dijalankan.

Contoh:

```yaml
environment:
  Jwt__Key: ${JWT_SECRET}
```

---

# 9. Apa itu .env?

`.env` adalah file yang umum digunakan untuk menyimpan environment variable secara lokal.

Contoh:

```env
JWT_SECRET=super-secret
DB_PASSWORD=root
```

`.env` bukan `appsettings.json`.

`.env` juga bukan otomatis menjadi file configuration ASP.NET Core.

Dalam konteks Docker Compose, alurnya adalah:

```text
.env
 ↓
Docker Compose
 ↓
Container Environment Variable
 ↓
ASP.NET Core Configuration
 ↓
Application
```

---

# 10. Contoh .env + Docker Compose

`.env`:

```env
JWT_SECRET=super-secret
DB_PASSWORD=root
```

`docker-compose.yaml`:

```yaml
services:
  api:
    environment:
      Jwt__Key: ${JWT_SECRET}
      ConnectionStrings__DefaultConnection: "server=mysql;port=3306;database=reimbursement_db;user=root;password=${DB_PASSWORD}"
```

Docker Compose membaca:

```text
${JWT_SECRET}
${DB_PASSWORD}
```

dari `.env`.

Kemudian container mendapatkan:

```text
Jwt__Key=super-secret
```

dan:

```text
ConnectionStrings__DefaultConnection=server=mysql;...
```

ASP.NET Core kemudian membacanya sebagai:

```text
Jwt:Key

ConnectionStrings:DefaultConnection
```

---

# 11. .env Tidak Otomatis Dibaca oleh appsettings.json

Ini penting.

Kita tidak bisa melakukan:

```json
{
  "Jwt": {
    "Key": "JWT_SECRET"
  }
}
```

dan berharap ASP.NET Core otomatis mencari environment variable bernama `JWT_SECRET`.

Value tersebut hanya akan dianggap sebagai string:

```text
JWT_SECRET
```

Bukan isi dari environment variable.

Jika ingin environment variable mengisi:

```text
Jwt:Key
```

maka gunakan:

```yaml
environment:
  Jwt__Key: ${JWT_SECRET}
```

Dengan:

```env
JWT_SECRET=super-secret
```

---

# 12. .env Berada di Host, Bukan Otomatis di Container

Misalnya project:

```text
Reimbursement_API/
├── Dockerfile
├── docker-compose.yaml
├── .env
└── ...
```

`.env` berada di host/project.

Docker Compose membacanya dan memberikan value yang diperlukan ke container.

Container tidak harus memiliki file:

```text
.env
```

di dalam filesystem-nya.

Misalnya:

```text
Host
│
├── .env
│     └── JWT_SECRET=secret
│
└── docker-compose.yaml
      │
      ▼
    Docker
      │
      ▼
Container
└── Environment Variable
      └── Jwt__Key=secret
```

---

# 13. Environment Variable dan Container

Environment variable adalah bagian dari configuration runtime container.

Contoh:

```yaml
services:
  api:
    environment:
      ASPNETCORE_ENVIRONMENT: Development
      Jwt__Key: ${JWT_SECRET}
```

Kita dapat melihat environment variable di dalam container:

```bash
docker exec -it reimbursement-api printenv
```

Atau:

```bash
docker exec -it reimbursement-api printenv Jwt__Key
```

Jika container dihapus, environment variable yang berada di container tersebut ikut hilang.

Namun `.env` yang berada di host tetap ada.

Ketika container dibuat kembali:

```bash
docker compose up -d
```

Docker Compose dapat memasukkan value tersebut kembali ke container.

---

# 14. Build dan Runtime adalah Dua Hal Berbeda

Docker memiliki fase:

```text
Build
  ↓
Image
```

dan:

```text
Run
  ↓
Container
```

Dockerfile:

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS builder

WORKDIR /src

COPY . .

RUN dotnet restore

RUN dotnet publish -c Release -o /app/publish

FROM mcr.microsoft.com/dotnet/aspnet:8.0

WORKDIR /app

COPY --from=builder /app/publish .

EXPOSE 8080

ENTRYPOINT ["dotnet", "Reimbursement_API.dll"]
```

Dockerfile digunakan untuk membuat image.

Sedangkan:

```yaml
environment:
  Jwt__Key: ${JWT_SECRET}
```

adalah configuration yang diberikan ketika container dijalankan.

---

# 15. Jangan Bake Secret ke dalam Image

Sebaiknya secret tidak dimasukkan ke Dockerfile.

Contoh yang sebaiknya dihindari:

```dockerfile
ENV JWT_SECRET=super-secret
```

Karena secret tersebut menjadi bagian dari image configuration.

Lebih baik:

```yaml
environment:
  Jwt__Key: ${JWT_SECRET}
```

Sehingga image dapat digunakan untuk berbagai environment tanpa membawa secret tertentu.

---

# 16. Build Once, Configure Per Environment

Konsep penting dalam containerization:

> Build once, configure per environment.

Artinya kita tidak perlu membuat image berbeda hanya karena environment berbeda.

Misalnya:

```text
Dockerfile
    ↓
reimbursement-api:1.0
```

Image yang sama dapat digunakan untuk:

```text
Development
Staging
Production
```

Yang berbeda adalah configuration saat container dijalankan.

Contoh:

```text
                    reimbursement-api:1.0
                            │
               ┌────────────┴────────────┐
               ▼                         ▼
        Development                  Production
          Container                    Container
               │                         │
        Environment A              Environment B
```

---

# 17. Container Name Tidak Menentukan Environment

Contoh:

```yaml
container_name: reimbursement-api-prod
```

tidak otomatis membuat ASP.NET menjadi Production.

Environment ditentukan oleh:

```yaml
environment:
  ASPNETCORE_ENVIRONMENT: Production
```

Jadi:

```yaml
container_name: reimbursement-api-prod

environment:
  ASPNETCORE_ENVIRONMENT: Development
```

tetap berarti ASP.NET Core berjalan dalam:

```text
Development
```

Nama container hanya nama.

---

# 18. Satu Image Bisa Digunakan oleh Dua Container

Misalnya:

```text
reimbursement-api:1.0
```

digunakan oleh:

```text
reimbursement-api-dev
reimbursement-api-prod
```

Keduanya menggunakan image yang sama tetapi environment berbeda.

Contoh:

```yaml
services:

  api-dev:
    image: reimbursement-api:1.0
    container_name: reimbursement-api-dev
    ports:
      - "5279:8080"
    environment:
      ASPNETCORE_ENVIRONMENT: Development

  api-prod:
    image: reimbursement-api:1.0
    container_name: reimbursement-api-prod
    ports:
      - "5280:8080"
    environment:
      ASPNETCORE_ENVIRONMENT: Production
```

Perhatikan bahwa host port harus berbeda jika kedua container dijalankan bersamaan.

---

# 19. `build: context: .` Tidak Berarti Development

Contoh:

```yaml
build:
  context: .
```

`.` berarti:

> Gunakan folder saat ini sebagai Docker build context.

Ini tidak berarti:

```text
Development
```

dan tidak menentukan:

```text
Production
```

Environment ditentukan ketika container dijalankan.

---

# 20. Development dan Production Biasanya Tidak Perlu Berjalan Bersamaan

Dalam deployment nyata, biasanya kita tidak menjalankan:

```text
Development API
+
Production API
```

dalam satu environment yang sama.

Biasanya:

```text
Development
    ↓
Development configuration
    ↓
API Container
```

dan:

```text
Production Server
    ↓
Production configuration
    ↓
API Container
```

Keduanya dapat menggunakan:

```text
reimbursement-api:1.0
```

---

# 21. Kenapa Konsep Ini Berguna?

Misalnya image:

```text
reimbursement-api:1.0
```

sudah dibuat dan diuji.

Kita dapat menjalankan image yang sama di production tanpa build ulang source code hanya karena configuration berbeda.

Development:

```text
Image
+
Development configuration
```

Production:

```text
Image
+
Production configuration
```

Sehingga:

```text
Same Application
Different Configuration
```

---

# 22. Pembagian Configuration yang Umum

Secara umum:

```text
Non-secret configuration
        ↓
appsettings.json
appsettings.Development.json
appsettings.Production.json

Secret / sensitive configuration
        ↓
Environment Variables
.env (development)
Secret Management (production)
```

Contoh:

```json
{
  "Jwt": {
    "Issuer": "Reimbursement_API",
    "Audience": "Reimbursement_Client",
    "ExpiresMinutes": 60
  }
}
```

Sedangkan secret:

```env
JWT_SECRET=super-secret
```

Kemudian Compose:

```yaml
environment:
  Jwt__Key: ${JWT_SECRET}
```

---

# 23. Gambaran Besar

Konsep configuration yang dipelajari:

```text
                         Configuration
                              │
            ┌─────────────────┼─────────────────┐
            │                 │                 │
            ▼                 ▼                 ▼
     appsettings.json   Environment-specific   Environment
                         appsettings             Variables
                                                   ▲
                                                   │
                                                  .env
                                                   │
                                                   │
                                            Docker Compose
```

Docker Compose kemudian menjalankan container dengan configuration tersebut.

```text
.env
 ↓
Docker Compose
 ↓
Container Environment
 ↓
ASP.NET Core Configuration
 ↓
Application
```

---

# 24. Hal yang Perlu Diingat

### `appsettings.json`

Tempat configuration umum aplikasi.

### `appsettings.Development.json`

Configuration khusus Development.

### `appsettings.Production.json`

Configuration khusus Production.

### `ASPNETCORE_ENVIRONMENT`

Menentukan environment ASP.NET Core.

```text
Development
Production
Staging
```

### Environment Variable

Configuration yang diberikan melalui environment runtime.

Contoh:

```text
Jwt__Key
ConnectionStrings__DefaultConnection
```

### `.env`

File yang sering digunakan untuk menyimpan value environment secara lokal, terutama bersama Docker Compose.

### Dockerfile

Digunakan untuk membuat image.

### Docker Compose

Digunakan untuk mendefinisikan dan menjalankan container beserta configuration/runtime setup-nya.

### Image

Artifact aplikasi yang dapat digunakan kembali di berbagai environment.

### Container

Instance dari image yang dijalankan dengan configuration tertentu.

---

# 25. Prinsip Utama

```text
Build Once
Configure Per Environment
```

Artinya:

```text
Dockerfile
    ↓
Image
    ↓
Same Image
    ├── Development + Dev Configuration
    ├── Staging     + Staging Configuration
    └── Production  + Prod Configuration
```

Image tidak perlu berbeda hanya karena configuration/environment berbeda.

Yang berubah adalah configuration ketika container dijalankan.