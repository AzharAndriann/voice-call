# Panduan Testing Frontend CipherVoice

Dokumen ini berisi cara menjalankan dan mengetes aplikasi CipherVoice setelah frontend di-deploy ke Vercel. Testing dilakukan menggunakan dua browser atau dua perangkat supaya alur User A dan User B lebih mudah dilihat.

## 1. Persiapan

Sebelum mulai, pastikan:

- Backend C sudah berjalan dan dapat diakses dari internet.
- Endpoint backend menggunakan WebSocket Secure (`wss://`) karena website Vercel menggunakan HTTPS.
- Browser sudah diberi izin menggunakan mikrofon.
- User A dan User B memakai passphrase yang sama.

Contoh endpoint:

```text
wss://alamat-backend/websocket
```

## 2. Konfigurasi Vercel

Pada dashboard Vercel, buka:

```text
Project → Settings → Environment Variables
```

Tambahkan variabel berikut:

```text
Name  : VITE_WS_URL
Value : wss://alamat-backend/websocket
```

Pilih environment `Production`, `Preview`, dan `Development` jika semuanya ingin menggunakan backend yang sama. Setelah variabel ditambahkan atau diubah, lakukan **Redeploy**. Variabel Vite dibaca saat proses build, jadi perubahan tidak langsung aktif tanpa build ulang.

Untuk menjalankan secara lokal, isi file `.env`:

```env
VITE_WS_URL=ws://localhost:8000/websocket
```

Kemudian jalankan:

```bash
npm install
npm run dev
```

## 3. Cek Koneksi WebSocket

1. Buka website CipherVoice.
2. Aplikasi otomatis mencoba terhubung ke endpoint dari `VITE_WS_URL`.
3. Perhatikan badge status pada bagian atas halaman.

Hasil yang diharapkan:

```text
CONNECTING → CONNECTED
```

Jika masih `DISCONNECTED`, buka tab **Kunci & Konfigurasi** dan periksa endpoint yang tampil. Tombol **Reconnect** dapat digunakan untuk mencoba koneksi ulang.

## 4. Testing Create dan Join Room

Gunakan dua browser berbeda, contohnya Chrome biasa untuk User A dan Incognito untuk User B. Bisa juga menggunakan dua laptop atau HP yang berbeda.

### User A sebagai host

1. Buka website CipherVoice.
2. Pastikan status sudah `CONNECTED`.
3. Klik **Buat Room Baru (6 Digit)**.
4. Catat atau salin kode room yang muncul.

Hasil yang diharapkan:

- Status berubah menjadi `IN_ROOM`.
- Label room menampilkan `HOST`.
- Kode room enam digit muncul, misalnya `000001`.
- Halaman menampilkan `Menunggu Lawan Bicara...`.

### User B sebagai guest

1. Buka website pada browser atau perangkat kedua.
2. Pastikan status sudah `CONNECTED`.
3. Masukkan kode milik User A.
4. Klik **Gabung**.

Hasil yang diharapkan:

- User B berstatus `IN_ROOM` dan memiliki label `GUEST`.
- User B menerima pesan berhasil bergabung.
- User A menerima notifikasi bahwa peer sudah bergabung.
- Kedua halaman menampilkan `Peer Terhubung`.

## 5. Menyamakan Kunci Enkripsi

1. Pada kedua user, buka tab **Kunci & Konfigurasi**.
2. Pilih **Passphrase / Kata Sandi Room**.
3. Masukkan passphrase yang sama pada kedua user.
4. Klik **Terapkan**.

Contoh untuk testing:

```text
KunciSuaraRahasiaKampus2026
```

Jika key berbeda, packet tetap dapat diteruskan server, tetapi proses dekripsi pada penerima akan gagal karena authentication tag tidak valid.

## 6. Testing Voice Call Dua Arah

1. Setelah User B bergabung, klik **Mulai Panggilan Suara (Host)** pada User A.
2. Izinkan akses mikrofon jika browser meminta permission.
3. Bicara melalui mikrofon User A.
4. Periksa apakah suara diterima pada perangkat User B.
5. Coba bicara dari User B untuk menguji arah sebaliknya.

Hasil yang diharapkan:

- Status panggilan berubah menjadi `CALL ACTIVE`.
- Waveform **Suara Anda** bergerak saat mikrofon menerima suara.
- Waveform **Lawan Bicara** bergerak saat packet audio diterima.
- Nilai `TERKIRIM` dan `DITERIMA` terus bertambah.
- Nilai `DITOLAK/DROP` tetap nol jika key dan packet valid.

Gunakan tombol mikrofon untuk mencoba fitur mute dan tombol speaker untuk mencoba fitur deafen. Klik **Akhiri Panggilan** setelah testing selesai.

> Gunakan headset saat mengetes dua perangkat yang berdekatan supaya tidak terjadi suara feedback atau gema.

## 7. Testing Packet Inspector

1. Mulai panggilan atau kirim chat terenkripsi.
2. Buka tab **Packet Inspector**.
3. Tunggu sampai packet pertama muncul.

Packet yang ditampilkan memiliki format:

```text
[sequence 4 byte][nonce 12 byte][ciphertext][tag 16 byte]
```

Yang perlu diperiksa:

- Sequence number berubah sesuai urutan packet.
- Nonce berukuran 12 byte.
- Ciphertext berisi data terenkripsi, bukan plaintext.
- Authentication tag berukuran 16 byte.
- Tombol **Salin Hex** berhasil menyalin detail packet.

## 8. Testing E2E Chat

1. Pastikan kedua user sudah berada di room yang sama.
2. Buka tab **E2E Chat**.
3. Ketik pesan dari User A, lalu klik **Kirim**.
4. Lakukan pengujian arah sebaliknya dari User B.

Hasil yang diharapkan:

- Pesan tampil pada kedua user.
- Tampilan pesan memperlihatkan potongan ciphertext.
- Isi pesan hanya dapat didekripsi jika kedua user memakai key yang sama.

## 9. Testing Tamper Lab

Tamper Lab dapat digunakan tanpa melakukan panggilan. Buka tab **Tamper Lab (Tahap 7)**, lalu jalankan pengujian berikut:

1. **Uji Paket Normal**  
   Hasil yang diharapkan: dekripsi berhasil dan plaintext asli muncul.

2. **Tamper Ciphertext**  
   Hasil yang diharapkan: packet ditolak karena satu byte ciphertext diubah.

3. **Tamper Auth Tag**  
   Hasil yang diharapkan: packet ditolak karena authentication tag tidak valid.

4. **Paket Terpotong (<32B)**  
   Hasil yang diharapkan: packet abnormal ditolak tanpa membuat aplikasi crash.

## 10. Testing Leave dan Disconnect

1. Klik **Tinggalkan Room** pada salah satu user.
2. Periksa halaman user yang lain.

Hasil yang diharapkan:

- User yang keluar kembali ke status `CONNECTED`.
- User lain menerima notifikasi bahwa peer meninggalkan room.
- Panggilan berhenti jika sedang aktif.

## 11. Checklist Hasil Testing

Gunakan checklist berikut sebelum demo atau pengumpulan:

- [ ] Website Vercel dapat dibuka.
- [ ] Status WebSocket berubah menjadi `CONNECTED`.
- [ ] User A berhasil membuat room.
- [ ] User B berhasil bergabung menggunakan kode room.
- [ ] Notifikasi `peer_joined` diterima User A.
- [ ] Kedua user memakai passphrase yang sama.
- [ ] Audio dapat dikirim dari User A ke User B.
- [ ] Audio dapat dikirim dari User B ke User A.
- [ ] Packet Inspector menampilkan sequence, nonce, ciphertext, dan tag.
- [ ] Chat terenkripsi berhasil dikirim dua arah.
- [ ] Tamper ciphertext berhasil ditolak.
- [ ] Tamper tag berhasil ditolak.
- [ ] Packet terpotong berhasil ditolak.
- [ ] Fitur leave room dan peer disconnect berjalan.

## 12. Troubleshooting

### Status selalu DISCONNECTED

- Pastikan backend sedang berjalan.
- Pastikan `VITE_WS_URL` sudah dimasukkan pada Vercel.
- Pastikan endpoint menggunakan `wss://`, bukan `ws://`, untuk website HTTPS.
- Lakukan redeploy setelah mengubah environment variable.
- Periksa Console dan Network → WS pada browser DevTools.

### Room tidak dapat dibuat

- Pastikan status sudah `CONNECTED` sebelum menekan tombol create room.
- Periksa log backend untuk memastikan request `create_room` diterima.

### User B tidak dapat bergabung

- Periksa kembali kode room enam digit.
- Pastikan room belum diisi user kedua.
- Pastikan kedua user terhubung ke backend yang sama.

### Audio tidak terdengar

- Pastikan izin mikrofon diberikan.
- Pastikan kedua user menggunakan key atau passphrase yang sama.
- Periksa apakah counter packet `TERKIRIM` dan `DITERIMA` bertambah.
- Gunakan headset dan periksa volume browser/perangkat.
- Beberapa browser memblokir audio otomatis; klik halaman terlebih dahulu sebelum memulai panggilan.

### Packet banyak ditolak

- Pastikan key kedua user sama.
- Pastikan frontend kedua user menggunakan versi aplikasi yang sama.
- Periksa apakah koneksi WebSocket sering terputus.

## Catatan

Server hanya bertugas mengelola room dan meneruskan packet binary. Proses enkripsi dilakukan pada frontend pengirim, sedangkan verifikasi tag dan dekripsi dilakukan pada frontend penerima.
