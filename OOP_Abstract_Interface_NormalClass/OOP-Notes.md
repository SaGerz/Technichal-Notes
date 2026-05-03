# 📚 Catatan OOP: Interface vs Abstract Class vs Class Biasa

Catatan ini dibuat untuk memahami perbedaan antara **Kontrak**, **Template**, dan **Warisan** dalam ekosistem .NET.

---

### 1. INTERFACE (Si "Buku Menu" / Kontrak)
*   **Keyword:** `interface IName`
*   **Isinya:** Hanya daftar judul fungsi saja. **DILARANG** memiliki logika/isi `{ }`.
*   **Karakteristik:**
    *   **Sangat Ketat:** Kelas yang menggunakan interface ini **WAJIB** mengimplementasikan seluruh fungsinya tanpa terkecuali.
    *   **Tanpa Logika:** Tidak memberikan warisan kode sama sekali, murni hanya "Daftar Janji".
*   **Contoh Kode:**
```csharp
public interface IReimbursementService {
    void Ajukan(decimal nominal); // Wajib diisi di class anak
}
```

### 2. ABSTRACT CLASS (Si "Template Setengah Jadi")
*   **Keyword:** `public abstract class Name`
*   **Isinya:** Campuran. Boleh ada fungsi yang **sudah ada isinya** (tinggal pakai), dan fungsi **`abstract`** (masih berupa judul).
*   **Karakteristik:**
    *   **Tidak Bisa Berdiri Sendiri:** Kamu tidak bisa memanggil `new BaseService()`. Harus diwariskan dulu ke kelas anak.
    *   **Wajib & Opsional:** Anak hanya **WAJIB** menulis ulang fungsi yang berlabel `abstract`. Fungsi lainnya otomatis menjadi "Warisan Gratis".
*   **Contoh Kode:**
```csharp
public abstract class BaseService {
    public void Log(string m) => Console.WriteLine(m); // Warisan Gratis
    public abstract bool Validasi(); // Wajib diisi di class anak
}
```

### 3. CLASS BIASA (Si "Warisan Full")
*   **Keyword:** `public class Name`
*   **Isinya:** Semua fungsi **WAJIB** memiliki logika lengkap di dalam `{ }`.
*   **Karakteristik:**
    *   **Siap Pakai:** Bisa langsung dipanggil dengan `new MyService()`.
    *   **Pewarisan Penuh:** Semua yang dimiliki Bapak, otomatis dimiliki Anak. Anak tidak wajib menulis ulang apa pun.
*   **Contoh Kode:**
```csharp
public class ReimbursementService : BaseService, IReimbursementService {
    public override bool Validasi() => true; // Mengisi mandat Abstract
    public void Ajukan(decimal n) => Log("Ok"); // Mengisi janji Interface
}
```

---

### 💡 Tabel Perbandingan Cepat


| Fitur | Interface | Abstract Class | Class Biasa |
| :--- | :--- | :--- | :--- |
| **Boleh ada isi fungsi?** | ❌ Tidak | ✅ Sebagian | ✅ Harus Ada |
| **Wajib tulis ulang?** | ✅ SEMUANYA | ✅ Yang `abstract` saja | ❌ Tidak Wajib |
| **Bisa dipanggil langsung?**| ❌ Tidak | ❌ Tidak | ✅ Bisa |
| **Berapa banyak?** | ✅ Boleh banyak | ❌ Cuma 1 Bapak | ❌ Cuma 1 Bapak |
| **Analogi** | **Buku Menu** | **Rumah Contoh** | **Rumah Jadi** |

---

### ⚠️ Aturan Emas C# (Satu Bapak, Banyak SOP)
Dalam C#, sebuah kelas hanya diperbolehkan memiliki **SATU BAPAK** (Class atau Abstract), namun diperbolehkan mengikuti **BANYAK SOP** (Interface).

```csharp
// CONTOH STRUKTUR YANG BENAR:
public class ReimbursementService : BaseService, IReimbursementService, ILogger
{
   // BaseService = Bapak kandung (Cuma satu)
   // IReimbursementService & ILogger = SOP tambahan (Boleh banyak)
}
```
