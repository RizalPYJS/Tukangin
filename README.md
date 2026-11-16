# Tukangin — panduan struktur proyek

Tukangin adalah website yang mempertemukan tukang dan penyedia jasa lokal (teknisi AC, tukang listrik, servis HP/laptop, pijat panggilan, dll.) dengan pelanggan, dimulai dari Kota Bandung.

Dokumen ini menjelaskan isi setiap file supaya kamu, atau AI lain yang membantu merapikan, bisa mengubah website tanpa merusak bagian lain.

---

## 1. Teknologi

| Bagian | Yang dipakai | Catatan |
|---|---|---|
| Tampilan | Satu file `index.html` (HTML + CSS + JavaScript biasa) | Tanpa framework, tanpa proses build |
| Database & login | **Supabase** (PostgreSQL + Auth + Storage) | Dipanggil lewat `supabase-js` v2 dari CDN jsDelivr |
| Hosting | **GitHub Pages** | Hanya file statis; tidak ada kode server |
| Keamanan data | **Row Level Security (RLS)** di database | Akses diatur di database, bukan di tampilan |

---

## 2. Daftar file

```
tukangin/
├── index.html                    ← seluruh website (tampilan + logika)
├── README.md                     ← dokumen ini
└── supabase/
    ├── 1-struktur-database.sql   ← tabel, aturan akses, fungsi, tempat foto, 5 penyedia awal
    ├── 2-calon-mitra.sql         ← 101 calon mitra dari Google Maps (khusus admin)
    └── 3-jadikan-admin.sql       ← menjadikan akun tertentu sebagai admin
```

File SQL hanya dijalankan sekali di Supabase → SQL Editor. Yang diunggah ke GitHub cukup `index.html` (dan README ini bila mau).

---

## 3. Isi `index.html`, dari atas ke bawah

| Bagian | Isi |
|---|---|
| `<head>` | Judul, font Google (Plus Jakarta Sans, IBM Plex Mono), semua CSS, dan `<script>` supabase-js |
| CSS → `:root` | **Token warna** (`--bg`, `--fg`, `--accent`, warna per kategori `--c-ac`, dst.) untuk mode terang dan gelap. Ubah warna di sini saja |
| CSS lainnya | Navigasi, beranda (hero, kolase, kartu layanan), kartu penyedia, peta, form, panel admin, footer |
| `<header>`, `<main>`, `<footer>` | Kerangka halaman. Isi `<main>` dan `<footer>` dibuat oleh JavaScript |
| `/* reference data */` | `CATS` (kategori jasa), `ICON` (ikon SVG), `KEC` (kecamatan + koordinat), `OSTEP` (status pesanan), `PSTAT` (status penyedia), `PROVS`/`KOTA_JABAR` (pilihan lokasi), `LIVE` (wilayah yang sudah aktif), `SLOTS` (jam pemesanan) |
| `/* state */` | Objek `S`, tempat semua data yang sedang tampil disimpan |
| `/* derived */` | Fungsi bantu: rating, jarak, filter pencarian |
| `/* koneksi database (Supabase) */` | `SUPABASE_URL`, `SUPABASE_ANON_KEY` (publishable key), fungsi pembaca data (`loadPublic`, `loadMine`, `loadAdmin`, `reload`), sesi (`applySession`, `ensureSession`), dan **semua penulisan data di objek `api`** |
| `/* nav & routing */` | Menu atas, footer, perpindahan halaman (`go`), render ulang (`render`, `refresh`) |
| `/* shared pieces */` | Komponen yang dipakai ulang: kartu penyedia, chip kategori, blok WhatsApp |
| `/* views */` | Setiap halaman adalah `VIEWS.<nama>` dengan fungsi `html()` (membuat tampilan) dan `after()` (memasang event form) |
| `/* admin */` | Panel admin dan tab-tabnya |
| `/* events */` | Penanganan klik, ketik, dan perubahan pilihan untuk seluruh halaman |
| `/* boot */` | Menyalakan website: membuat koneksi Supabase, membaca sesi, memuat data, memuat ulang tiap 30 detik |

### Daftar halaman (`VIEWS`)

| Nama | Isi | Siapa |
|---|---|---|
| `beranda` | Hero "#TukangDekatmu", pencarian, kolase foto, cara kerja, kategori layanan | Semua |
| `cari` | Daftar penyedia + filter kategori, lokasi, urutan, "Buka 24 jam", "Mitra terkonfirmasi" | Semua |
| `peta` | Peta sebaran penyedia (SVG dari koordinat) | Semua |
| `profil` | Detail penyedia, link Google Maps & rute, ulasan | Semua |
| `pesan` | Form pemesanan | Semua (sesi otomatis dibuat) |
| `pesanan` | Pesanan milik perangkat ini + status | Pelanggan |
| `ulas` | Form ulasan setelah pesanan selesai | Pelanggan |
| `mitra` | Form pendaftaran penyedia jasa | Semua |
| `masuk` | Login admin (dibuka lewat alamat `#admin`) | Admin |
| `admin` | Tab: Pesanan, Pendaftaran, Calon mitra, Penyedia, Foto, Pengaturan | Admin |
| `tambah` | Tambah / edit data penyedia | Admin |

---

## 4. Database (Supabase)

| Tabel | Isi | Siapa yang bisa membaca | Siapa yang bisa mengubah |
|---|---|---|---|
| `providers` | Penyedia jasa yang tampil di website | Semua (kecuali yang `nonaktif`) | Admin |
| `provider_contacts` | Nomor telepon penyedia | Admin | Admin |
| `orders` | Pesanan | Pemilik pesanan & admin | Pelanggan membuat (status harus `baru`); admin mengubah |
| `reviews` | Ulasan | Semua | Pelanggan, hanya untuk pesanannya yang `selesai` |
| `applications` | Pendaftaran mitra | Pendaftar & admin | Pendaftar membuat; admin mengubah |
| `prospects` | 101 calon mitra dari Google Maps | Admin | Admin |
| `settings` | Nomor WA, email, Instagram, LinkedIn, foto kategori | Semua | Admin |
| `admins` | Daftar akun admin | Diri sendiri | Lewat SQL saja |

Fungsi database:
- `is_admin()` → `true` bila akun yang login adalah admin.
- `cancel_order(id)` → pelanggan membatalkan pesanannya sendiri selama statusnya masih `baru`.

Tempat foto: bucket Storage `foto` (publik untuk dilihat, hanya admin yang mengunggah).

Login:
- **Admin**: email + password (dibuat di Supabase → Authentication → Users).
- **Pelanggan**: tanpa daftar. Website membuat sesi anonim otomatis saat pertama memesan (fitur "Allow anonymous sign-ins" harus menyala).

---

## 5. Yang harus tetap sinkron

Kalau salah satu diubah, pasangannya juga harus diubah:

| Di `index.html` | Di database |
|---|---|
| Fungsi `mapProv`, `mapOrder`, `mapApp`, `mapRev`, `mapProsp`, `provRow`, dan objek `api` (nama kolom seperti `provider_id`, `g_rating`, `admin_note`) | Nama kolom di tabel |
| `OSTEP` (`baru`, `dikonfirmasi`, `dikerjakan`, `selesai`, `batal`) | `check` pada `orders.status` |
| `PSTAT` (`aktif`, `belum_bergabung`, `nonaktif`) | `check` pada `providers.status` |
| `PST` (status calon mitra) | `check` pada `prospects.cstatus` |
| `CATS` (id kategori: `ac`, `listrik`, `hp`, …) | Isi kolom `cat` di `providers`, `applications`, `prospects` |

---

## 6. Aturan penting

1. **Jangan pernah** menaruh *secret key* (`sb_secret_...`) atau *service_role key* di `index.html`. Yang boleh hanya *publishable key* (`sb_publishable_...`).
2. **Jangan mematikan atau menghapus Row Level Security** dan policy di database. Keamanan data bergantung pada itu, bukan pada tampilan website.
3. **Mengubah database = jalankan SQL baru**, misalnya `4-tambah-kolom-harga.sql` berisi `alter table ... add column ...`. Menjalankan ulang file 1 tidak menambah kolom baru, karena memakai `create table if not exists`.
4. Website ini statis (GitHub Pages). Tidak ada kode server; semua logika berjalan di browser dan dijaga oleh aturan database.
5. Uji di **Chrome** dari alamat GitHub Pages, bukan di pratinjau aplikasi Claude (pratinjau memblokir koneksi ke Supabase).

---

## 7. Kalau ingin dipecah menjadi beberapa file

Satu file paling mudah dikelola dari HP. Kalau nanti dirapikan, struktur yang disarankan:

```
tukangin/
├── index.html            ← kerangka HTML saja
├── css/style.css         ← isi <style>
├── js/config.js          ← SUPABASE_URL, SUPABASE_ANON_KEY
├── js/data.js            ← reference data (CATS, KEC, PROVS, …)
├── js/db.js              ← koneksi Supabase, load*, api, writeError
├── js/ui.js              ← nav, footer, render, komponen bersama
├── js/views/*.js         ← satu file per halaman (beranda.js, cari.js, admin.js, …)
├── js/app.js             ← events + boot
└── supabase/*.sql
```

Urutan `<script>` di `index.html` harus: supabase-js → config → data → db → ui → views → app.

---

## 8. Contoh perintah untuk AI lain

> Ini proyek website Tukangin. Baca README.md dulu. Saya lampirkan index.html dan supabase/1-struktur-database.sql.
> Tugas: [jelaskan perubahan yang kamu mau].
> Aturan: jangan ubah nama tabel atau kolom tanpa membuat file SQL migrasi baru; jangan menaruh secret key; jangan mematikan RLS; semua penulisan data lewat objek `api`; tetap satu file `index.html` yang bisa diunggah ke GitHub Pages.
