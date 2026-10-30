# Laporan Kerentanan Keamanan — Aplikasi Presensi Elayanan TVRI (SIMPEG)

> **RAHASIA / PRIVAT — untuk pengelola SIMPEG TVRI.**
> Jangan dipublikasikan. Dokumen ini berisi dugaan kerentanan yang belum diperbaiki.
> Disusun untuk tujuan *responsible disclosure* (pelaporan agar diperbaiki), bukan eksploitasi.
> Tanggal: 30 Oktober 2026

---

## Ringkasan

Dari analisis aplikasi mobile **Elayanan TVRI** (`com.layanan` v3.1) dan interaksi dengan API `https://simpeg.tvri.go.id`, ditemukan beberapa dugaan kelemahan keamanan pada modul presensi. Laporan ini menjelaskan temuan, dampak, dan rekomendasi perbaikan. **Tidak** disertakan langkah eksploitasi detail maupun data pegawai mana pun.

---

## Temuan 1 — Broken Access Control / IDOR pada data presensi (Prioritas: TINGGI)

**Deskripsi:**
Endpoint presensi (mis. `cek_masuk`) tampak mengembalikan data presensi berdasarkan `pegawai_id` yang dikirim di request, tanpa memverifikasi bahwa `pegawai_id` tersebut milik sesi/pengguna yang sedang login. Akibatnya, mengganti nilai `pegawai_id` dapat menampilkan data presensi **pegawai lain**.

**Data yang berpotensi bocor:** jam masuk, status WFO/WFH, shift, tanggal, status pulang, ID record — milik pegawai lain.

**Dampak:** Pelanggaran privasi data kepegawaian. Jika `pegawai_id` diiterasi, data presensi seluruh pegawai dapat ditarik oleh pihak yang memiliki akses aplikasi.

**Penguat (eskalasi):** Response `cek_masuk` menyertakan `id` (UUID record presensi) milik pegawai yang di-query. `id` ini juga dipakai endpoint lain (`presensiById` untuk baca detail, `saveDataPulang` untuk update record pulang). Artinya, membocorkan `id` record pegawai lain berpotensi menaikkan celah dari **baca** menjadi **mereferensikan/mengubah** record milik orang lain — perlu diverifikasi apakah endpoint tersebut juga tidak memeriksa kepemilikan.

**Rekomendasi:**
- Terapkan **otorisasi per-objek**: setiap endpoint harus memverifikasi bahwa `pegawai_id`/record yang diminta milik pengguna pada sesi aktif (ambil identitas dari sesi server, bukan dari parameter request).
- Abaikan `pegawai_id` dari body untuk data milik sendiri; gunakan identitas sesi.

---

## Temuan 2 — Kontrol anti-kecurangan berjalan di sisi klien (Prioritas: TINGGI)

**Deskripsi:**
Validasi anti-kecurangan presensi tampak dilakukan di aplikasi (klien):
- Deteksi **Fake GPS / mock location**
- **Liveness** wajah (Google ML Kit)

Karena berjalan di klien, kontrol ini dapat dilewati bila penyerang memanggil API presensi langsung (di luar aplikasi) dengan foto & koordinat yang disiapkan sendiri.

**Dampak:** Presensi dapat dipalsukan tanpa benar-benar hadir di lokasi / tanpa verifikasi wajah nyata.

**Rekomendasi:**
- Lakukan validasi **di sisi server**: verifikasi lokasi (geofencing) di server (sudah terlihat ada perhitungan `jarak` di server — pastikan ini *enforcing*, bukan sekadar informatif).
- Pertimbangkan verifikasi wajah / liveness di server, atau tanda tangan integritas pada data presensi.
- Deteksi mock-location sebaiknya dikuatkan dengan sinyal server-side (mis. konsistensi lokasi, kecepatan perpindahan).

---

## Temuan 3 — Dugaan: stempel waktu presensi dapat dipengaruhi klien (Prioritas: TINGGI — perlu verifikasi)

**Deskripsi (DUGAAN, belum diverifikasi):**
Field tanggal/jam presensi (`present`, `starting`, `finish`) dibangun aplikasi dari waktu yang diperoleh klien. Perlu dipastikan apakah **server menimpa** nilai ini dengan waktu server, atau **mempercayai** nilai kiriman klien. Jika server mempercayai kiriman klien, maka presensi berpotensi **di-backdate** (mengisi kehadiran untuk tanggal/jam lampau).

> Catatan: dugaan ini **tidak** kami uji, karena memverifikasinya berarti membuat record presensi palsu. Mohon diverifikasi oleh tim internal di lingkungan uji.

**Dampak:** Manipulasi catatan kehadiran (mis. menutupi keterlambatan), berdampak ke integritas tunjangan.

**Rekomendasi:**
- Server **harus selalu** menetapkan `present`/`starting`/`finish` dari **waktu server**, mengabaikan nilai waktu dari klien.
- Tolak submit presensi yang tanggalnya tidak sama dengan tanggal server saat itu.

---

## Temuan 4 — Device binding mengandalkan FCM token (Prioritas: SEDANG)

**Deskripsi:**
Pengikatan 1 akun = 1 perangkat tampak bergantung pada FCM token. Ini membatasi login, namun endpoint data/presensi diautentikasi via session cookie — perlu dipastikan session juga terikat ke perangkat/binding, agar sesi yang bocor tidak bisa dipakai lintas perangkat.

**Rekomendasi:**
- Pastikan masa berlaku sesi wajar dan terikat konteks perangkat.
- Pertimbangkan rotasi/again-binding saat anomali perangkat terdeteksi.

---

## Catatan metodologi

- Temuan berasal dari analisis statik APK (dekompilasi bundle React Native/Hermes) dan interaksi wajar dengan endpoint.
- **Tidak ada data pegawai lain yang dikumpulkan, disimpan, atau disebarkan** dalam laporan ini.
- Pengujian yang bersifat merusak/memalsukan record **tidak dilakukan**.

## Kontak & tindak lanjut

Mohon konfirmasi penerimaan laporan ini dan perkiraan waktu perbaikan. Kami siap memberikan detail teknis tambahan secara privat kepada tim yang berwenang bila diperlukan.
