# Tutorial Penggunaan INAPROC API Gateway

Dokumen ini berisi **panduan lengkap penggunaan API INAPROC API Gateway**, disusun agar **mudah dipahami orang awam**, namun **cukup detail untuk langsung dipraktikkan secara teknis** oleh pengembang aplikasi.

---

## 0. Gambaran Umum (Untuk Orang Awam)

INAPROC API Gateway adalah **jalur resmi untuk mengambil data pengadaan pemerintah** (RUP, Tender, E‑Purchasing, Penyedia, dll).

Cara berpikir sederhananya:

* **INAPROC** = gudang data pengadaan
* **API Gateway** = pintu masuk gudang
* **API Token** = kunci pintu
* **Aplikasi Anda** = pihak yang mengambil data

Tanpa **token**, aplikasi **tidak bisa mengambil data**.

---

## 1. Spesifikasi Dasar API

### 1.1 Base URL

Semua endpoint INAPROC diawali dengan:

```
https://data.inaproc.id/api
```

Contoh:

```
/v1/ekatalog/paket-e-purchasing
```

URL lengkapnya:

```
https://data.inaproc.id/api/v1/ekatalog/paket-e-purchasing
```

---

### 1.2 Metode Request

* Metode utama: **GET**
* Format response: **JSON**

Artinya:

* Kita hanya *mengambil data*, bukan mengubah data.

---

## 2. Autentikasi (WAJIB)

Setiap request **harus menyertakan API Token**.

### 2.1 Header Wajib

```
Authorization: Bearer API_TOKEN_ANDA
Accept: application/json
```

Jika token:

* salah
* sudah kedaluwarsa
* tidak punya izin API

Maka API akan mengembalikan error **401 atau 403**.

⚠️ Token **tidak boleh**:

* disimpan di GitHub
* ditulis langsung di source code

Gunakan `.env` atau secret manager.

---

## 3. Struktur Umum Endpoint

Pola umum endpoint:

```
/v1/{kategori}/{resource}
```

Contoh kategori:

* `rup` → Rencana Umum Pengadaan
* `tender` → Tender & Non‑Tender
* `ekatalog` → E‑Purchasing & Penyedia

---

## 4. Parameter WAJIB (Sering Terjadi Error)

Hampir semua endpoint data **WAJIB** menyertakan dua parameter berikut:

| Parameter   | Fungsi                |
| ----------- | --------------------- |
| `kode_klpd` | Kode instansi K/L/PD  |
| `tahun`     | Tahun anggaran (YYYY) |

❌ Jika salah satu tidak ada → API akan menolak request (**HTTP 400**).

Contoh benar:

```
?kode_klpd=D123&tahun=2024
```

---

## 5. Contoh Penggunaan API (Paling Dasar)

### 5.1 Contoh Kasus

Mengambil **data paket E‑Purchasing** milik satu instansi pada tahun tertentu.

### 5.2 Contoh Request (cURL)

```
curl -X GET \
"https://data.inaproc.id/api/v1/ekatalog/paket-e-purchasing?kode_klpd=D123&tahun=2024&limit=10" \
-H "Authorization: Bearer API_TOKEN_ANDA" \
-H "Accept: application/json"
```

---

## 6. Struktur Response API

Setiap endpoint INAPROC **menggunakan struktur response yang sama**.

### 6.1 Contoh Response

```
{
  "success": true,
  "data": [
    {
      "id": "pkg-001",
      "nama_paket": "Pengadaan Laptop",
      "satuan_kerja": "Dinas Kominfo",
      "pagu": 500000000,
      "tahun_anggaran": 2024,
      "status": "Aktif"
    }
  ],
  "meta": {
    "cursor": "eyJpZCI6InBrZy0wMDEifQ==",
    "has_more": true,
    "count": 1,
    "total": 150
  }
}
```

### 6.2 Penjelasan Field

| Field           | Arti                       |
| --------------- | -------------------------- |
| `success`       | Status request             |
| `data`          | Data utama (array)         |
| `meta.cursor`   | Penanda halaman berikutnya |
| `meta.has_more` | Masih ada data atau tidak  |
| `meta.count`    | Jumlah data halaman ini    |
| `meta.total`    | Total data keseluruhan     |

---

## 7. Mengambil Data Banyak (Pagination)

INAPROC **tidak menggunakan page=1,2,3**.
Sebagai gantinya digunakan **cursor**.

### 7.1 Langkah Pagination

#### Langkah 1 – Ambil halaman pertama

```
GET /v1/ekatalog/paket-e-purchasing?kode_klpd=D123&tahun=2024&limit=50
```

#### Langkah 2 – Ambil nilai `cursor` dari response

```
"cursor": "eyJpZCI6InBrZy0wNTAifQ=="
```

#### Langkah 3 – Ambil halaman berikutnya

```
GET /v1/ekatalog/paket-e-purchasing?kode_klpd=D123&tahun=2024&limit=50&cursor=eyJpZCI6InBrZy0wNTAifQ==
```

#### Langkah 4 – Ulangi sampai

```
"has_more": false
```

⚠️ Aturan penting:

* Cursor **tidak boleh diubah**
* Pagination **harus berurutan**
* Cursor **tidak disimpan untuk lain waktu**

---

## 8. Rate Limit (Batas Pemakaian)

Setiap response membawa header:

| Header                  | Fungsi      |
| ----------------------- | ----------- |
| `X‑RateLimit‑Limit`     | Total kuota |
| `X‑RateLimit‑Remaining` | Sisa kuota  |
| `X‑RateLimit‑Reset`     | Waktu reset |

Jika kuota habis:

* API mengembalikan **HTTP 429**

Solusi:

* Gunakan cache
* Jangan request berlebihan
* Tunggu reset

---

## 9. Error yang Paling Sering Terjadi

### 9.1 Error 400 – Parameter Kurang

Penyebab:

* `kode_klpd` atau `tahun` tidak dikirim

Solusi:

* Pastikan parameter wajib lengkap

---

### 9.2 Error 401 – Token Salah / Expired

Solusi:

* Generate token baru

---

### 9.3 Error 403 – Tidak Punya Akses API

Solusi:

* Pastikan API dicentang saat membuat token

---

### 9.4 Error 429 – Terkena Rate Limit

Solusi:

* Tunggu reset
* Terapkan caching

---

## 10. Praktik Terbaik Penggunaan API

✔ Gunakan cache (3–6 jam untuk data pengadaan)
✔ Gunakan `limit` 50–100
✔ Gunakan retry dengan jeda (1s → 2s → 4s)
✔ Pantau sisa kuota

❌ Jangan hard‑code token
❌ Jangan log token

---

## 11. Contoh Implementasi Kode (Python, Node.js, Java, PHP)

Bagian ini memberi contoh **kode siap pakai** untuk memanggil INAPROC API Gateway.

> Catatan penting:
>
> * Ganti `API_TOKEN_ANDA` dengan token asli.
> * Contoh menggunakan endpoint **E-Purchasing**: `/v1/ekatalog/paket-e-purchasing`
> * Pastikan menyertakan parameter wajib: `kode_klpd` dan `tahun`.

---

### 11.1 Python (requests)

**Install library (jika belum):**

```bash
pip install requests
```

**File: `inaproc_client.py`**

```python
import os
import time
import random
import requests

BASE_URL = "https://data.inaproc.id/api"
TOKEN = os.getenv("INAPROC_API_TOKEN", "API_TOKEN_ANDA")


def get_with_retry(url: str, headers: dict, max_retries: int = 3, initial_delay: float = 1.0):
    """Retry untuk error sementara (429 dan 5xx) dengan exponential backoff + jitter."""
    for attempt in range(max_retries):
        resp = requests.get(url, headers=headers, timeout=30)

        # Sukses
        if resp.status_code == 200:
            return resp

        # Non-retryable (umumnya)
        if 400 <= resp.status_code < 500 and resp.status_code != 429:
            raise RuntimeError(f"Non-retryable error: HTTP {resp.status_code} - {resp.text}")

        # Retryable: 429 / 5xx
        if resp.status_code == 429 or resp.status_code >= 500:
            if attempt == max_retries - 1:
                raise RuntimeError(f"Max retries reached: HTTP {resp.status_code} - {resp.text}")

            base_delay = initial_delay * (2 ** attempt)
            jitter = base_delay * 0.1
            delay = max(0.0, base_delay + random.uniform(-jitter, jitter))

            # Jika server mengirim Retry-After, utamakan itu
            retry_after = resp.headers.get("Retry-After")
            if retry_after:
                try:
                    delay = max(delay, float(retry_after))
                except ValueError:
                    pass

            time.sleep(delay)
            continue

        raise RuntimeError(f"Unhandled error: HTTP {resp.status_code} - {resp.text}")


def fetch_all_epurchasing(kode_klpd: str, tahun: str, limit: int = 50):
    headers = {
        "Authorization": f"Bearer {TOKEN}",
        "Accept": "application/json",
    }

    cursor = None
    all_items = []

    while True:
        url = f"{BASE_URL}/v1/ekatalog/paket-e-purchasing?kode_klpd={kode_klpd}&tahun={tahun}&limit={limit}"
        if cursor:
            url += f"&cursor={cursor}"

        resp = get_with_retry(url, headers)
        payload = resp.json()

        if not payload.get("success"):
            raise RuntimeError(f"API returned success=false: {payload}")

        data = payload.get("data", [])
        meta = payload.get("meta", {})

        all_items.extend(data)

        cursor = meta.get("cursor")
        has_more = meta.get("has_more")

        if not has_more:
            break

    return all_items


if __name__ == "__main__":
    # Contoh
    items = fetch_all_epurchasing(kode_klpd="D123", tahun="2024", limit=50)
    print(f"Total items fetched: {len(items)}")
    # Print 1 item pertama (jika ada)
    if items:
        print(items[0])
```

**Cara menjalankan:**

```bash
export INAPROC_API_TOKEN="API_TOKEN_ANDA"
python inaproc_client.py
```

---

### 11.2 Node.js (axios)

**Install library:**

```bash
npm install axios
```

**File: `inaproc_client.js`**

```js
const axios = require('axios');

const BASE_URL = 'https://data.inaproc.id/api';
const TOKEN = process.env.INAPROC_API_TOKEN || 'API_TOKEN_ANDA';

async function sleep(ms) {
  return new Promise((resolve) => setTimeout(resolve, ms));
}

async function getWithRetry(url, maxRetries = 3, initialDelay = 1000) {
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      const res = await axios.get(url, {
        timeout: 30000,
        headers: {
          Authorization: `Bearer ${TOKEN}`,
          Accept: 'application/json',
        },
        validateStatus: () => true, // biar kita handle status sendiri
      });

      if (res.status === 200) return res;

      // Non-retryable (umumnya)
      const isNonRetryable = res.status >= 400 && res.status < 500 && res.status !== 429;
      if (isNonRetryable) {
        throw new Error(`Non-retryable error: HTTP ${res.status} - ${JSON.stringify(res.data)}`);
      }

      // Retryable: 429 / 5xx
      const isRetryable = res.status === 429 || res.status >= 500;
      if (!isRetryable || attempt === maxRetries - 1) {
        throw new Error(`Failed: HTTP ${res.status} - ${JSON.stringify(res.data)}`);
      }

      const baseDelay = initialDelay * Math.pow(2, attempt);
      const jitter = baseDelay * 0.1;
      let delay = Math.max(0, baseDelay + (Math.random() * 2 - 1) * jitter);

      // Respect Retry-After jika ada
      const retryAfter = res.headers['retry-after'];
      if (retryAfter) {
        const ra = Number(retryAfter);
        if (!Number.isNaN(ra)) delay = Math.max(delay, ra * 1000);
      }

      await sleep(delay);
    } catch (err) {
      // Jika error network (timeout/dll), kita retry juga
      if (attempt === maxRetries - 1) throw err;
      const baseDelay = initialDelay * Math.pow(2, attempt);
      const jitter = baseDelay * 0.1;
      const delay = Math.max(0, baseDelay + (Math.random() * 2 - 1) * jitter);
      await sleep(delay);
    }
  }
}

async function fetchAllEPurchasing(kodeKLPD, tahun, limit = 50) {
  let cursor = null;
  const allItems = [];

  while (true) {
    let url = `${BASE_URL}/v1/ekatalog/paket-e-purchasing?kode_klpd=${encodeURIComponent(kodeKLPD)}&tahun=${encodeURIComponent(tahun)}&limit=${limit}`;
    if (cursor) url += `&cursor=${encodeURIComponent(cursor)}`;

    const res = await getWithRetry(url);
    const payload = res.data;

    if (!payload.success) {
      throw new Error(`API returned success=false: ${JSON.stringify(payload)}`);
    }

    allItems.push(...(payload.data || []));

    cursor = payload.meta?.cursor;
    const hasMore = payload.meta?.has_more;
    if (!hasMore) break;
  }

  return allItems;
}

(async () => {
  const items = await fetchAllEPurchasing('D123', '2024', 50);
  console.log('Total items fetched:', items.length);
  if (items.length) console.log('First item:', items[0]);
})();
```

**Jalankan:**

```bash
export INAPROC_API_TOKEN="API_TOKEN_ANDA"
node inaproc_client.js
```

---

### 11.3 Java (HttpClient)

**File: `InaprocClient.java`**

> Contoh ini fokus pada **panggilan satu halaman** (tanpa looping semua halaman) agar tetap ringkas.
> Pola cursor bisa diterapkan dengan mengulang request sambil menyimpan `meta.cursor` dan `meta.has_more`.

```java
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.time.Duration;

public class InaprocClient {

    private static final String BASE_URL = "https://data.inaproc.id/api";

    public static void main(String[] args) throws Exception {
        String token = System.getenv().getOrDefault("INAPROC_API_TOKEN", "API_TOKEN_ANDA");

        String kodeKLPD = "D123";
        String tahun = "2024";
        int limit = 10;

        String url = BASE_URL + "/v1/ekatalog/paket-e-purchasing" +
                "?kode_klpd=" + kodeKLPD +
                "&tahun=" + tahun +
                "&limit=" + limit;

        HttpClient client = HttpClient.newBuilder()
                .connectTimeout(Duration.ofSeconds(10))
                .build();

        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create(url))
                .timeout(Duration.ofSeconds(30))
                .header("Authorization", "Bearer " + token)
                .header("Accept", "application/json")
                .GET()
                .build();

        HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());

        System.out.println("HTTP Status: " + response.statusCode());
        System.out.println("Body: " + response.body());

        // Untuk pagination:
        // - parse JSON: ambil meta.cursor dan meta.has_more
        // - ulangi request dengan parameter &cursor=...
    }
}
```

**Jalankan:**

```bash
export INAPROC_API_TOKEN="API_TOKEN_ANDA"
# compile
javac InaprocClient.java
# run
java InaprocClient
```

---

### 11.4 PHP (cURL)

**File: `inaproc_client.php`**

```php
<?php

$baseUrl = 'https://data.inaproc.id/api';
$token = getenv('INAPROC_API_TOKEN') ?: 'API_TOKEN_ANDA';

$kode_klpd = 'D123';
$tahun = '2024';
$limit = 10;

$url = $baseUrl . '/v1/ekatalog/paket-e-purchasing'
     . '?kode_klpd=' . urlencode($kode_klpd)
     . '&tahun=' . urlencode($tahun)
     . '&limit=' . $limit;

$ch = curl_init();

curl_setopt_array($ch, [
    CURLOPT_URL => $url,
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_TIMEOUT => 30,
    CURLOPT_HTTPHEADER => [
        'Authorization: Bearer ' . $token,
        'Accept: application/json',
    ],
]);

$response = curl_exec($ch);
$httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);

if ($response === false) {
    $err = curl_error($ch);
    curl_close($ch);
    die("cURL error: $err
");
}

curl_close($ch);

echo "HTTP Status: $httpCode
";
echo "Body: $response
";

// Untuk pagination:
// - json_decode($response, true)
// - ambil meta[cursor] dan meta[has_more]
// - ulangi request dengan &cursor=...
```

**Jalankan:**

```bash
export INAPROC_API_TOKEN="API_TOKEN_ANDA"
php inaproc_client.php
```

---

## 12. Ringkasan Singkat

> Cara menggunakan API INAPROC:
>
> 1. Siapkan token → 2) Panggil endpoint dengan header Bearer → 3) Sertakan `kode_klpd` & `tahun` → 4) Jika data banyak, ulangi pakai `meta.cursor` sampai `has_more=false`.

---

Dokumen ini dapat dijadikan:

* Panduan teknis internal
* Modul pelatihan
* Lampiran dokumen integrasi
* Pegangan developer pemula
