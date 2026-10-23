Nama: Fadhil Zahiruddin Munawwar 
NIM: 20251020034
Prodi: Teknik Informatika 

# PayClock: Absensi dan Gaji Karyawan Damken Print

Mini app CLI berbahasa PHP untuk mencatat absensi, menghitung lembur, dan memproses gaji bulanan karyawan Damken Print.
Mata kuliah Pemrograman Berorientasi Objek (TI033015), Pertemuan 4: Encapsulation.
Fondasinya adalah analisis berbasis objek dari Pertemuan 2 (class `Karyawan`, `Absensi`, `SlipGaji`).

## Cara menjalankan

Butuh PHP 8.1 ke atas. Jalankan perintah satu per satu di terminal (jangan ditempel sekaligus).

1. Masuk ke folder proyek. Di Windows dengan XAMPP, arahkan juga PATH ke PHP-nya:

```
cd C:\xampp\htdocs\payclock
$env:Path += ";C:\xampp\php"
```

2. Cek versi PHP:

```
php -v
```

3. Jalankan pengujian (hasil yang benar: `30 lulus, 0 gagal`):

```
php tests/run.php
```

4. Buat data contoh:

```
php bin/seed-contoh.php
```

5. Jalankan aplikasi dengan data contoh, lalu ketik angka menu dan Enter (`0` untuk keluar):

```
php bin/payclock.php --data=data/contoh.json
```

6. Untuk data sendiri, jalankan tanpa opsi (data tersimpan di `data/payclock.json`):

```
php bin/payclock.php
```
```

Nama dan gaji pada data contoh hanyalah contoh, bukan data asli Damken Print.

## Pemetaan kebutuhan ke kode

| FR | Kebutuhan | Menu | Class yang bertanggung jawab |
| --- | --- | --- | --- |
| FR1 | Catat jam masuk dan pulang | 5, 6 | `Absensi` |
| FR2 | Kelola karyawan, gaji pokok, status | 2, 3, 4 | `Karyawan` |
| FR3 | Total jam kerja per hari dan periode | 7 | `Absensi`, `PayClock::rekapBulan` |
| FR4 | Jam lembur dan bonus | 8 | `Absensi`, `SlipGaji` |
| FR5 | Gaji karyawan aktif per bulan | 8 | `SlipGaji` |
| FR6 | Lengkapi absensi yang belum pulang | 6 | `Absensi`, `PayClock` |
| FR7 | Tandai gaji sudah dibayar | 9 | `SlipGaji`, `PayClock` |
| FR8 | Rekap dan slip milik karyawan | 7, 10 | `PayClock`, `SlipGaji` |

## Struktur proyek

| File | Fungsi |
| --- | --- |
| `src/Karyawan.php` | Identitas, gaji pokok, status aktif |
| `src/Absensi.php` | Satu catatan kehadiran per karyawan per hari |
| `src/SlipGaji.php` | Slip satu karyawan satu bulan (draft ke dibayar) |
| `src/PayClock.php` | Memegang daftar objek dan aturan lintas objek |
| `src/Penyimpanan.php` | Simpan dan muat JSON |
| `src/Waktu.php` | Helper validasi dan format tanggal, jam, rupiah |
| `bin/payclock.php` | Menu CLI (hanya input dan tampilan, tanpa rumus gaji) |
| `tests/run.php` | Pengujian |
| `bukti/` | Hasil pengujian dan transkrip demo CLI |

## Penjelasan desain: apa yang berubah dan kenapa

**Sebelum.** Analisis Pertemuan 2 hanya mendaftar atribut dan method. Jika ditulis apa adanya dengan property public,
siapa pun bisa menulis `$karyawan->gajiPokok = -5`, `$slip->totalGaji = 1`, atau mengosongkan `jamPulang` pada slip
yang sudah dibayar. Rumus gaji juga mudah bocor ke kode menu.

**Sesudah.** Seluruh state penting private. Perubahan hanya lewat method yang menjaga aturan, dengan alur
luar objek, public method, validasi, lalu state internal.

| Class | State private | Aturan yang dijaga |
| --- | --- | --- |
| `Karyawan` | id, nama, gajiPokok, aktif | Nama 2-60 karakter; gaji pokok > 0 dan maks Rp50 juta (pengaman salah ketik); aktifkan/nonaktifkan yang sudah berstatus sama ditolak |
| `Absensi` | karyawanId, tanggal, menitMasuk, menitPulang | Format tanggal dan jam valid; pulang harus setelah masuk; pulang tidak bisa dicatat dua kali; menghitung absensi yang belum pulang ditolak |
| `SlipGaji` | semua field, constructor private | Hanya karyawan aktif; absensi harus milik karyawan dan bulan itu, tanpa duplikat, dan semua sudah pulang; slip dibayar tidak bisa dihitung ulang atau dibayar dua kali |
| `PayClock` | daftar karyawan, absensi, slip | Satu absen masuk per karyawan per hari; hanya karyawan aktif yang bisa absen; bulan yang sudah dibayar terkunci |

Aturan yang menyangkut satu objek ada di objek itu. Aturan yang menyangkut banyak objek (misalnya "satu absen per hari")
ada di `PayClock`, karena hanya pemilik daftar yang bisa memeriksanya.

### Keputusan yang menutup celah analisis Pertemuan 2

1. **Snapshot gaji pokok.** `SlipGaji` menyimpan `gajiPokok` saat dihitung. Tanpa itu, rincian slip bulan lalu tidak bisa
   direkonstruksi setelah gaji pokok diubah. Ini diuji pada skenario "Snapshot".
2. **Status karyawan.** Ditambah `aktifkan()` dan `nonaktifkan()` karena FR2 menuntut status bisa dikelola, tetapi
   analisis hanya punya `isAktif()`. Tidak ada setter status mentah.
3. **Status absensi tidak disimpan.** "Lengkap atau belum" dihitung dari ada tidaknya jam pulang, sehingga tidak mungkin
   ada status "lengkap" dengan jam pulang kosong.
4. **Siapa memegang daftar absensi.** `SlipGaji::hitung()` menerima daftar absensi sebagai parameter; daftar dipegang
   `PayClock`. `SlipGaji` jadi mudah diuji tanpa objek lain, dan `Karyawan` tidak kelebihan tanggung jawab.
   `PayClock::absensiBulan()` mengembalikan salinan (clone) agar catatan asli tidak bisa diubah lewat jalan pintas.
5. **Slip yang sudah dibayar.** Tidak bisa dihitung ulang, tidak bisa dibayar dua kali, dan absensi bulan itu terkunci.
   Memproses gaji ulang melewati slip yang sudah dibayar.
6. **Uang dan waktu.** Durasi disimpan sebagai menit (int) dan rupiah sebagai int, bukan float, agar tidak ada galat
   pembulatan.
7. **Exception, bukan diam-diam mengabaikan.** Contoh `setJudul()` di slide hanya `return` saat input kosong. Di sini
   setiap pelanggaran aturan melempar exception dengan pesan yang menyebut apa yang salah dan nilainya, sehingga pemanggil
   tidak mengira operasi berhasil. Perhitungan gaji dilakukan lebih dulu, state baru diubah jika semuanya lolos.
8. **Constructor private pada `SlipGaji`.** Slip hanya bisa dibuat lewat `SlipGaji::buat()` (menghitung langsung) atau
   `SlipGaji::dariArray()` (memuat data tersimpan), jadi tidak ada slip setengah jadi.

### Asumsi bisnis yang perlu dikonfirmasi pemilik

- **Lembur dihitung per menit (prorata), dibulatkan ke bawah ke rupiah.** Contoh: pulang 17:20 menghasilkan Rp16.666.
  Alternatifnya hanya jam penuh; ubah di `Absensi::hitungMenitLembur()` dan `SlipGaji::hitung()`.
- **Ketidakhadiran tidak memotong gaji.** Sesuai model analisis: gaji pokok tetap ditambah bonus lembur. Jam kerja
  hanya untuk rekap. Jika karyawan yang tidak masuk harus dipotong, itu requirement baru.
- **Tidak ada shift lintas tengah malam.** Jam pulang harus setelah jam masuk pada hari yang sama.
- **Hari Minggu dan hari libur tidak dibedakan.** Semua menit setelah 17:00 dihitung lembur.
- **Batas bawah gaji pokok (UMR) belum dipasang.** Nilainya berubah tiap tahun. Jika tarif lembur atau UMR perlu
  riwayat perubahan, saat itulah `AturanGaji` layak menjadi class (sesuai syarat di analisis).
- **Belum ada autentikasi.** Siapa pun yang memegang aplikasi bisa memilih ID karyawan mana pun.

## Skenario pengujian

`php tests/run.php` menjalankan 30 skenario; hasil lengkap ada di `bukti/hasil-pengujian.txt` (**30 lulus, 0 gagal**).
Pengujian juga diperiksa dengan merusak aturan secara sengaja pada salinan sementara; setiap perusakan terdeteksi.
Empat kelompok utama (minimal 4 skenario sesuai tugas):

| Jenis | Contoh skenario | Hasil |
| --- | --- | --- |
| Normal | Gaji pokok 3.000.000 + lembur 90 menit | Total gaji 3.075.000 |
| Batas | Pulang tepat 17:00; masuk 18:00 pulang 20:00; lembur 20 menit | Lembur 0; 120 menit; bonus 16.666 |
| Input salah | Jam 25:00; tanggal 2026-02-30; gaji pokok 0; pulang sebelum masuk | `InvalidArgumentException` |
| Aturan bisnis | Absensi belum pulang; slip dibayar dua kali; absen di bulan terkunci | `DomainException` |
| Akses langsung | `$karyawan->gajiPokok = -5`, `$slip->totalGaji = 1` | `Error: Cannot access private property` |

`bukti/demo-cli.txt` memuat satu sesi CLI lengkap pada data contoh: proses gaji dengan satu karyawan gagal karena belum
pulang, melengkapi absensi, memproses ulang, membayar, lalu mencoba membayar lagi dan menambah absen di bulan terkunci.

## Catatan untuk pertemuan berikutnya

Kode ini fondasi untuk Pertemuan 5. Calon pengembangan: antarmuka untuk penyimpanan (JSON ke database),
`AturanGaji` sebagai class bila tarif perlu riwayat, dan peran pengguna (Pemilik dan Karyawan) dengan hak akses berbeda.

payclock
├── src
│   ├── Karyawan.php      ← class Karyawan
│   ├── Absensi.php       ← class Absensi
│   ├── SlipGaji.php      ← class SlipGaji
│   ├── PayClock.php      ← pemegang daftar dan aturan lintas objek
│   ├── Penyimpanan.php   ← simpan/muat JSON
│   └── Waktu.php         ← helper tanggal, jam, rupiah
├── bin
│   ├── payclock.php      ← menu aplikasi (CLI)
│   └── seed-contoh.php   ← pembuat data contoh
├── tests
│   └── run.php           ← pengujian
├── bukti                 ← hasil pengujian dan transkrip demo
├── data                  ← file data (dibuat saat aplikasi jalan)
├── bootstrap.php
└── README.md
