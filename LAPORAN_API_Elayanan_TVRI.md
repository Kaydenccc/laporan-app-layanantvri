# Dokumentasi API — Elayanan TVRI (SIMPEG TVRI)

> Hasil reverse engineering APK v3.1 + dekompilasi bytecode Hermes (`hermes-dec`) + uji endpoint langsung.
> Base URL: `https://simpeg.tvri.go.id/`
> Tanggal: 28 Oktober 2026

**Label keyakinan:**
- ✅ **PASTI** — field dibaca langsung dari kode terdekompilasi (`decompiled.js`) dan/atau diuji live.
- Semua endpoint di bawah berasal dari kode asli app, bukan tebakan.

---

## 1. Informasi Umum

| Item | Nilai |
|---|---|
| Base URL | `https://simpeg.tvri.go.id/` |
| Framework backend | CodeIgniter (routing `Controller/method`) |
| Framework app | React Native (Hermes bytecode v96) |
| Server | nginx di belakang CDN PWS |
| Autentikasi | **Session cookie** `simpegTVr126` (Max-Age 7200s) |
| Method | **POST** untuk hampir semua endpoint (termasuk getTahun/getBulan) |
| Content-Type request | **`multipart/form-data`** (app pakai `FormData`); server juga terima `x-www-form-urlencoded` |
| Format response | selalu **JSON** |
| Header | `Accept: application/json`, `User-Agent: okhttp/4.x` |

> Data dikirim sebagai **FormData (multipart)** di semua endpoint — dibuktikan di kode (`new FormData(); append(...)`). Backend baca `$_POST`, jadi JSON diabaikan.

---

## 2. Field Identitas Perangkat (login)

Login **tidak** mengirim `device_id`/fingerprint. Yang dikirim:

| Field | Isi | Sumber (react-native-device-info) |
|---|---|---|
| `token` | **FCM token** (push, dari Google Firebase) | `messaging().getToken()` |
| `token_device` | brand, mis. `realme` | `getBrand()` |
| `token_versi` | versi OS Android, mis. `10` | `getSystemVersion()` |

- `getFingerprint()`/`getUniqueId()`/`getDeviceId()` **tidak dipakai** di login.
- **Device binding** dikunci ke **FCM `token`** (satu-satunya nilai unik per-instalasi).

### 2.1 Di mana FCM token disimpan?

FCM token **diterbitkan oleh server Google (FCM)**, dan Google memegang catatan **otoritatif**-nya (pemetaan `token → instance app → device`) untuk me-routing push. Di HP hanya ada **salinan cache**.

```
Server Google FCM   ← catatan OTORITATIF (untuk routing push)
      │  (saat pertama / refresh)
      ▼
Firebase SDK + Google Play Services di HP  ← CACHE token
      │
      └─ getToken() baca dari cache ini (cepat, tanpa network)
```

| Lokasi | Token ada? | Bisa dibaca? |
|---|---|---|
| Server Google FCM | ✅ otoritatif | Hanya pemilik project Firebase (server key / Firebase console) |
| Cache di HP (data privat app / Google Play Services) | ✅ salinan | Perlu **root** |
| Saat app mengirim ke server TVRI (field `token`) | ✅ sesaat | **Intercept** traffic |

Catatan: lokasi persis cache lokal bergantung versi Firebase SDK & Google Play Services (bisa di storage GMS dan/atau SharedPreferences/datastore Firebase Installations di data privat app). Intinya: **otoritatif di Google, cache di HP** — bukan nilai statik yang bisa dirakit manual.

---

## 3. AUTENTIKASI

### 3.1 Login ✅
```
POST user/Presensi/login_new        (multipart/form-data)
```
| Field | Isi |
|---|---|
| `username` | NIP ASN |
| `password` | **plaintext** |
| `token` | FCM token |
| `token_device` | `getBrand()` (brand) |
| `token_versi` | `getSystemVersion()` (versi OS) |

**Response:**
```jsonc
{"status":201,"keterangan":"Username dan Password salah "}    // kredensial salah
{"status":201,"keterangan":"Username & Password Salah"}        // password dikirim ter-encode (format salah)
{"status":201,"keterangan":"Gagal Login ! Akun anda sudah terdaftar pada device yang lain ... kode berikut #XXXXX"}  // binding
{"status":200, ...data pegawai... }                            // sukses
```
Sukses → app simpan `response.data` ke AsyncStorage `@sesi`; server set cookie `simpegTVr126`.

### 3.2 Ganti Password ✅
```
POST user/Presensi/ubah_password     (multipart/form-data)
```
| Field | Isi |
|---|---|
| `pegawai_id` | NIP |
| `password` | password baru |
| `repassword` | konfirmasi (nama field: **`repassword`**) |

Validasi app: min 8 karakter, mengandung huruf besar, kecil, angka, karakter khusus.

### 3.3 Logout ✅
```
POST user/Presensi/logout
```

### 3.4 Login lama (JANGAN pakai)
```
POST front_end/Login/auth_pegawai_app   → 500 Error
```

---

### 3.5 Session & Identitas `pegawai_id` (Temuan)

**Session (cookie):**
- Login sukses → server mengirim `Set-Cookie: simpegTVr126=...; Max-Age=7200` (berlaku ~2 jam).
- App (okhttp) menyimpan cookie ini dan **mengirimnya otomatis di header `Cookie`** pada setiap request berikutnya → inilah yang mengautentikasi endpoint presensi/data.
- Field data (pegawai_id, GPS, foto) ada di **body**; session ada di **header `Cookie`** — dua tempat berbeda.

**Penyimpanan data login:**
- Response login `data` disimpan app ke AsyncStorage **`@sesi`** (`decompiled.js:231658`).
- Semua endpoint mengambil `pegawai_id` dari `@sesi` via `getData()` (`decompiled.js:231402, 231435`) — jadi nilainya = `response.data.pegawai_id` dari login.

**`pegawai_id` = NIP atau ID internal? — ⚠️ BELUM PASTI**
- Nilai `pegawai_id` berasal dari response login, **bukan** diketik user.
- Indikasi (bukan bukti) bahwa ini mungkin ID internal, bukan NIP: sistem memakai field `username`/`nip` DAN `pegawai_id` secara terpisah; response `detail_pegawai` mengembalikan `nip` sebagai field tersendiri (selain `pegawai_id`).
- Kepastian hanya dari melihat isi `@sesi` / response login.

**Perilaku `detail_pegawai`:**
- Tanpa cookie session valid → `{"status":200,"data":null}` (teruji).
- Struktur `data` (dengan session): `nama_pegawai`, `nip`, `foto`, `url_foto`, `lokasi_id`, `lokasi_name`, `is_blokabsen`, `is_close`, `is_updatepass` (`decompiled.js:248715-248766`).
- Jika dikirim `pegawai_id` yang salah (mis. NIP padahal server mengharap ID internal), kemungkinan juga mengembalikan `null` meski session valid.

**Catatan keamanan:** cookie `simpegTVr126` = kredensial sesi. Siapa pun yang memilikinya bisa bertindak sebagai user sampai kadaluarsa (~2 jam) — perlakukan seperti password, jangan dibagikan.

---

## 4. PRESENSI — Tahap Persiapan (sebelum ambil foto)

### 4.1 Cek status WFO/WFH ✅
```
POST user/Presensi/status_wf
```
Field: `pegawai_id`, `lokasi_id` → response `data` (opsi status kerja).

### 4.2 Ambil shift / jam kerja ✅
```
POST user/Presensi/shift
```
Field: `pegawai_id`, `lokasi_id` → response `data` (jam kerja).

### 4.3 Cek waktu server + lokasi ✅
```
POST user/Presensi/get_time
```
| Field | Isi |
|---|---|
| `lat` | latitude |
| `long` | longitude |
| `status_wf` | status kerja terpilih |
| `pegawai_id` | NIP |
| `lokasi_id` | ID lokasi |

### 4.4 Cek eligibility masuk ✅
```
POST user/Presensi/cek_masuk
```
Field: `pegawai_id`, `lokasi_id`. (Hanya cek — **tidak** mengirim foto.)

---

## 5. PRESENSI — Submit Absen (dengan foto)

### 5.1 Absen MASUK ✅
```
POST user/Presensi/saveData          (multipart/form-data)
```
| Field | Isi |
|---|---|
| `starting` | jam masuk lengkap (`set_jam`) |
| `present` | tanggal (`set_jam` 11 char pertama) |
| `starting_lat` | latitude |
| `starting_lng` | longitude |
| `status_wf` | status kerja |
| `no` | NIP (`pegawai_id`) |
| `user_id` | NIP (`pegawai_id`) |
| `shift_id` | jam kerja (`set_jam_kerja`) |
| `starting_photo` | **file foto** `{name, type, uri}` (bukan base64) |

### 5.2 Absen PULANG ✅
```
POST user/Presensi/saveDataPulang    (multipart/form-data)
```
| Field | Isi |
|---|---|
| `finish` | jam pulang (`set_jam`) |
| `id` | ID presensi (`id_p`) |
| `finish_lat` | latitude |
| `finish_lng` | longitude |
| `no` | NIP |
| `user_id` | NIP |
| `finish_photo` | **file foto** `{name, type, uri}` |

> Foto dikirim sebagai **file multipart** (objek `{name, type, uri}`), bukan string base64.

---

## 6. RIWAYAT & PROFIL

### 6.1 Detail / Profil Pegawai ✅
```
POST user/Presensi/detail_pegawai
```
Field: `pegawai_id` → response `data` (nama_pegawai, dll.).

### 6.2 Daftar Tahun ✅
```
POST user/Presensi/getTahun
```
Field: `pegawai_id` → `{"status":200,"data":[{"id":2022,"name":2022},...2026]}`
(juga jalan sebagai GET tanpa param — server lenient.)

### 6.3 Daftar Bulan ✅
```
POST user/Presensi/getbulan
```
Field: `pegawai_id` → `{"status":200,"data":[{"id":"01","name":"Januari"},...]}`

### 6.4 Rekap Presensi ✅
```
POST user/Presensi/rekap_presensi
```
Field: `pegawai_id`, `tahun`, `bulan` → rekap kehadiran.
Validasi app: "Tahun dan Bulan tidak boleh kosong".

### 6.5 Detail Presensi per ID ✅
```
POST user/Presensi/presensiById
```
Field: **`id`** (= idAbsen, BUKAN pegawai_id) → detail 1 record absen.

---

## 7. INFO & NOTIFIKASI

### 7.1 Info / Pengumuman ✅
```
GET user/Presensi/info
```
Response: `{"status":200,"foto":[{"id","title","image"},...]}`

### 7.2 Slider beranda ✅
```
POST user/Presensi/slider
```

### 7.3 Notifikasi SIMPEG ✅ — pegawai_id di URL
```
POST user/Api/notif_simpeg/<pegawai_id>
```
Response: `{"status_code":200,"total":N}`

### 7.4 Notifikasi Surat ✅ — pegawai_id di URL
```
POST user/Api/notif_surat/<pegawai_id>
```
Response: `{"status_code":404,"status":"NOT_FOUND","total":"0","message":"Tidak Ada Data Surat","data":null}`

### 7.5 Webview Modul ✅ — query string (dimuat di WebView)
```
GET user/Api/webview?app=<modul>&pegawai_id=<nip>
```
`app` = nama modul (cuti, gaji, surat, dinas luar, dll.).

### 7.6 Detail front_end ✅
```
GET front_end/front_end/detail/
```

---

## 8. Ringkasan Tabel Endpoint

| # | Method | Endpoint | Field / Param |
|---|---|---|---|
| 1 | POST | `user/Presensi/login_new` | username, password, token, token_device, token_versi |
| 2 | POST | `user/Presensi/ubah_password` | pegawai_id, password, repassword |
| 3 | POST | `user/Presensi/logout` | (sesi) |
| 4 | POST | `user/Presensi/status_wf` | pegawai_id, lokasi_id |
| 5 | POST | `user/Presensi/shift` | pegawai_id, lokasi_id |
| 6 | POST | `user/Presensi/get_time` | lat, long, status_wf, pegawai_id, lokasi_id |
| 7 | POST | `user/Presensi/cek_masuk` | pegawai_id, lokasi_id |
| 8 | POST | `user/Presensi/saveData` | starting, present, starting_lat, starting_lng, status_wf, no, user_id, shift_id, starting_photo |
| 9 | POST | `user/Presensi/saveDataPulang` | finish, id, finish_lat, finish_lng, no, user_id, finish_photo |
| 10 | POST | `user/Presensi/detail_pegawai` | pegawai_id |
| 11 | POST | `user/Presensi/getTahun` | pegawai_id |
| 12 | POST | `user/Presensi/getbulan` | pegawai_id |
| 13 | POST | `user/Presensi/rekap_presensi` | pegawai_id, tahun, bulan |
| 14 | POST | `user/Presensi/presensiById` | id (idAbsen) |
| 15 | GET | `user/Presensi/info` | — |
| 16 | POST | `user/Presensi/slider` | — |
| 17 | POST | `user/Api/notif_simpeg/<pegawai_id>` | pegawai_id (URL) |
| 18 | POST | `user/Api/notif_surat/<pegawai_id>` | pegawai_id (URL) |
| 19 | GET | `user/Api/webview?app=&pegawai_id=` | app, pegawai_id (query) |
| 20 | GET | `front_end/front_end/detail/` | — |

---

## 9. Alur Presensi Lengkap

```
LOGIN (login_new) → simpan @sesi + cookie
   ↓
detail_pegawai (profil)  +  notif_simpeg / notif_surat
   ↓  [mulai presensi]
status_wf → shift → get_time (validasi lokasi + waktu + status)
   ↓
cek_masuk (cek eligibility di lokasi)
   ↓  [buka kamera + ML Kit liveness]
saveData (MASUK)  /  saveDataPulang (PULANG)
   → kirim foto (file) + lat/lng + jam + shift
```

---

## 10. Mekanisme Keamanan

| Proteksi | Detail |
|---|---|
| Session cookie | `simpegTVr126` (2 jam) |
| Device binding | via **FCM token**; login device lain ditolak (kode ke Admin SIMPEG) |
| Anti Fake GPS | deteksi mock location → blokir |
| Geofencing | wajib dalam radius (`lat`/`long` dicek di `get_time`) |
| Liveness wajah | Google ML Kit lokal (`assets/models_bundled/`) |
| Password | plaintext via HTTPS; min 8 char kompleks saat ganti |

---

## 11. Metodologi

- APK diekstrak dari XAPK → `com.layanan.apk`.
- Bundle `assets/index.android.bundle` = Hermes bytecode v96.
- Didekompilasi dengan `hermes-dec` → `decompiled.js` (±16 MB).
- Field tiap endpoint dibaca dari konstruksi `FormData.append(...)` di kode (ref baris di `decompiled.js`: login 231565, saveData 331258, saveDataPulang 354635, status_wf 331614, shift 331705, get_time 331796, cek_masuk 249171, rekap 337644, getTahun 337548, getbulan 337483, presensiById 354435, ubah_password 352705, detail_pegawai 248650, notif 249006/249095, webview 330664).
- Endpoint publik diverifikasi live (getTahun, getBulan, info, notif, rekap).

## 12. Lampiran — Setup ADB via Kabel USB (Windows)

Untuk inspeksi app sendiri & persiapan intercept traffic (debugging).

### Langkah 1 — Unduh ADB (platform-tools)
1. Buka `https://developer.android.com/tools/releases/platform-tools`
2. Unduh **SDK Platform-Tools for Windows**
3. Ekstrak ke folder, mis. `C:\platform-tools`

### Langkah 2 — Aktifkan USB Debugging di HP
1. **Setelan → Tentang Ponsel**
2. Ketuk **Nomor bentukan (Build number)** 7× → mode pengembang aktif
3. **Setelan → Opsi Pengembang** → aktifkan **USB Debugging**

### Langkah 3 — Colok & otorisasi
1. Colok HP ke PC via USB (kabel **data**, bukan charge-only)
2. Mode USB di HP → **Transfer File (MTP)**
3. Popup **"Izinkan USB debugging?"** → centang **"Selalu izinkan dari komputer ini"** → **OK**

### Langkah 4 — Verifikasi
```powershell
cd C:\platform-tools
.\adb devices
```
Hasil benar:
```
List of devices attached
ABC123XYZ       device
```
- `unauthorized` → cek popup izin di HP
- kosong → ganti kabel data, atau install driver USB OEM (realme/OPPO)

### Langkah 5 — Perintah inspeksi berguna
```powershell
.\adb devices                                    # cek device
.\adb shell                                       # shell HP
.\adb logcat                                       # log sistem (debugging)
.\adb shell pm list packages | findstr layanan    # cek package app
.\adb shell dumpsys package com.layanan            # permission & komponen app
.\adb install aplikasi.apk                          # pasang APK
```

### Catatan
- ADB Wireless (tanpa kabel) juga tersedia di Android 10: Opsi Pengembang → Wireless debugging (perlu pairing pertama via kabel di sebagian perangkat).
- Untuk intercept HTTPS app ini perlu bypass trust sertifikat (root atau patch APK) — lihat catatan di Bagian 10 soal proteksi.

---

*Dokumentasi untuk pemahaman API atas akun milik sendiri. Endpoint yang mengubah data (absensi, password) tidak dieksekusi selama pengujian.*
