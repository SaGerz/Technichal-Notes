# 📚 Catatan ASP.NET Core: appsettings, Environment, dan EF Migration Saat Pindah Laptop

Catatan ini dibuat untuk memahami bagaimana konfigurasi ASP.NET Core bekerja ketika project dipindahkan ke device/laptop baru, terutama terkait:

- `appsettings.json`
- `appsettings.Development.json`
- `launchSettings.json`
- `ASPNETCORE_ENVIRONMENT`
- EF Core Migration

---

# 🧠 Kenapa `appsettings.json` Biasanya Tidak Berisi Credential Asli?

Dalam ASP.NET Core, file seperti:

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "server=;port=;database=;user=;password=;"
  }
}
```

biasanya dijadikan **template/config dasar** dan tetap di-push ke Git.

Sedangkan credential asli seperti:
- password database
- JWT key
- secret API

biasanya dipisahkan ke file lain seperti:

```text
appsettings.Development.json
```

yang tidak di-push ke repository (`.gitignore`).

---

# 📁 Struktur Konfigurasi ASP.NET Core

Biasanya struktur project seperti ini:

```text
Reimbursement_API/
│
├── appsettings.json
├── appsettings.Development.json
├── Properties/
│   └── launchSettings.json
```

---

# 1. appsettings.json (Base Configuration)

File utama/default configuration.

Biasanya aman untuk di-push ke Git karena hanya berisi template.

### Contoh:

```json
{
  "ConnectionStrings": {
    "DefaultConnection": ""
  },
  "jwt": {
    "Key": "",
    "Issuer": "",
    "Audience": "",
    "ExpiresMinutes": 60
  }
}
```

---

# 2. appsettings.Development.json (Local Development Config)

File khusus untuk development/local.

Biasanya berisi credential asli.

### Contoh:

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "server=127.0.0.1;port=3306;database=reimbursement_db;user=root;password=;"
  }
}
```

## ⚠️ Penting
File ini:
- **TERPISAH**
- bukan ditulis di dalam `appsettings.json`

ASP.NET Core akan otomatis menggabungkan (merge) config tersebut.

---

# 🔥 Cara ASP.NET Core Merge Configuration

Urutannya:

1. Load `appsettings.json`
2. Load `appsettings.{Environment}.json`
3. Value yang sama akan dioverride

Contoh:

## appsettings.json

```json
{
  "ConnectionStrings": {
    "DefaultConnection": ""
  }
}
```

## appsettings.Development.json

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "REAL_CONNECTION_STRING"
  }
}
```

Maka hasil akhirnya:

```text
REAL_CONNECTION_STRING
```

---

# 🧠 Bagaimana ASP.NET Tahu Lagi Environment Apa?

ASP.NET Core membaca environment variable:

```text
ASPNETCORE_ENVIRONMENT
```

Contoh:

```text
Development
```

atau:

```text
Production
```

---

# 📌 Role launchSettings.json

Saat project dijalankan dari Visual Studio, file:

```text
Properties/launchSettings.json
```

digunakan untuk menentukan environment.

### Contoh:

```json
{
  "profiles": {
    "https": {
      "commandName": "Project",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    }
  }
}
```

Artinya:
- Visual Studio menjalankan project sebagai `Development`
- maka ASP.NET otomatis membaca:

```text
appsettings.json
appsettings.Development.json
```

---

# 🔥 Program.cs dan Configuration

Biasanya config dibaca seperti ini:

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseMySql(
        builder.Configuration.GetConnectionString("DefaultConnection"),
        ServerVersion.AutoDetect(
            builder.Configuration.GetConnectionString("DefaultConnection")
        )
    )
);
```

Artinya:
- ASP.NET mengambil `DefaultConnection`
- dari hasil merge configuration
- sesuai environment aktif

---

# 📦 Pindah Laptop dan Database Baru

Saat clone project ke laptop baru:
- database biasanya masih kosong
- table belum ada

Yang perlu dilakukan:

## 1. Isi appsettings.Development.json

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "server=127.0.0.1;port=3306;database=reimbursement_db;user=root;password=;"
  }
}
```

---

## 2. Buat Database Kosong

Contoh:

```text
reimbursement_db
```

⚠️ Nama database sebenarnya bebas, tidak harus sama seperti laptop lama.

---

## 3. Jalankan EF Migration

```bash
dotnet ef database update
```

EF Core akan:
- membaca migration
- membuat table otomatis

---

# ⚠️ Error: dotnet ef Tidak Dikenali

Jika muncul:

```text
Could not execute because the specified command or file was not found.
```

biasanya:
- global tool `dotnet-ef` belum terinstall

Install dengan:

```bash
dotnet tool install --global dotnet-ef
```

---

# ⚠️ Error: Build failed

Jika muncul:

```text
Build failed. Use dotnet build to see the errors.
```

berarti:
- EF migration gagal karena project gagal compile

Cek dengan:

```bash
dotnet build
```

---

# ⚠️ Error JSON: ; expected

Kasus yang sempat terjadi:

```text
; expected
```

penyebabnya:
- file `appsettings.Development.json`
  terbaca sebagai file C# (`.cs`)
- bukan JSON file

---

# ✅ Solusi

Buat ulang file sebagai:

```text
appsettings.Development.json
```

dan pastikan:
- extension benar `.json`
- bukan `.cs`
- bukan `.txt`

---

# 💡 Insight Penting

### appsettings.json
➡️ Aman untuk Git  
➡️ Template/Base config

### appsettings.Development.json
➡️ Khusus local/dev  
➡️ Jangan dipush

### launchSettings.json
➡️ Mengatur environment saat debugging di Visual Studio

### ASPNETCORE_ENVIRONMENT
➡️ Penentu config mana yang dipakai

### EF Migration
➡️ Digunakan untuk recreate database schema saat pindah laptop/server

---

# 🚀 Alur Aman Clone ASP.NET Core Project ke Laptop Baru

```text
1. Clone repository
2. Open solution di Visual Studio
3. Restore package
4. Buat appsettings.Development.json
5. Isi connection string
6. Buat database kosong
7. Install dotnet-ef (jika belum ada)
8. Jalankan:
   dotnet ef database update
9. Run project
```