# PouCode Baileys — Laporan Audit & Persiapan Rilis npm

Dokumen ini mencatat hasil audit terhadap ZIP `Baileys-main_5` (fork PouCode).
Dokumen ini TIDAK ikut dipublikasikan ke npm (tidak tercantum di `files` pada package.json).

## 1. Perubahan yang diterapkan

| # | File | Masalah | Perbaikan | Dampak kompatibilitas |
|---|------|---------|-----------|-----------------------|
| 1 | `lib/Socket/messages-send.js` | `sendUnifiedResponse` memakai `options.noSelfSync`, tetapi `options` tidak ada di scope → `ReferenceError` setiap pemanggilan. | Menambahkan parameter opsional `options = {}` (pola sama dengan `sendRichMessage`). | Aditif. Pemanggil lama dengan 3 argumen tetap berfungsi. |
| 2 | `lib/Store/make-cache-manager-store.js` + `package.json` `cache-manager` | Dependensi `^7.2.8` menarik v7, yang tidak lagi mengekspor `caching()` (hanya `createCache`). Pemanggilan `caching(store)` akan gagal. | Pin ke `^5.7.6`; v5 mengekspor `caching(store)` dengan signature yang sama dengan kode. | Hanya memengaruhi `makeCacheManagerStore`. |
| 3 | `lib/Store/make-cache-manager-store.js` | v5 memakai `ttl` dalam **milidetik**; kode menyimpan `creds` dengan `ttl = 63115200` (dimaksudkan 2 tahun dalam detik) → kredensial kedaluwarsa dalam ~17 jam. | `ttl = 63115200 * 1000`. | Perilaku sesuai niat awal (2 tahun). |
| 4 | `package.json` `dependencies` | `long` dipakai di `lib/Socket/messages-recv.js` tetapi tidak dideklarasikan (hanya transitif dari protobufjs). | Menambahkan `"long": "^5.0.0"` (rentang yang dipakai protobufjs 7.5.6). | Aditif. |
| 5 | `package.json` `peerDependenciesMeta` | `sharp` adalah peer dependency **wajib**; npm akan memaksa instal `sharp` (native) ke setiap konsumen. Kode memuatnya secara dinamis dan opsional. | Menandai `sharp` sebagai `optional`. | Konsumen tanpa `sharp` tidak lagi gagal instal. |
| 6 | `package.json` `types` | Menunjuk ke `lib/index.d.ts` yang tidak ada. | Field dihapus. Typings untuk API utama BELUM tersedia (lihat bagian 4). | Tidak ada perubahan runtime. |
| 7 | `README.md` | Contoh memakai `fetchLatestWAWebVersion` (nama API sebenarnya `fetchLatestWaWebVersion`) dan menganggap nilai kembaliannya string (sebenarnya `{ version, isLatest }`). | Nama dan pemakaian diperbaiki; contoh pairing dibungkus fungsi async. | Dokumentasi saja. |
| 8 | `README.md` | Contoh QR memakai `printQRInTerminal: true`, opsi itu sudah deprecated dan tidak mencetak QR lagi. | Contoh diganti ke event `connection.update` (`qr`). | Dokumentasi saja. |
| 9 | `README.md` | Install "not published on npm"; `require()` pada paket ESM-only memerlukan Node 22.12+/20.19+. | Menambahkan instruksi npm (setelah publish), contoh `import`, dan catatan versi Node. | Dokumentasi saja. |
| 10 | 95 file di `lib/` | Komentar `//# sourceMappingURL=*.js.map` menunjuk ke file `.map` yang tidak ada. | Komentar dihapus (hanya baris komentar di akhir file). | Tidak ada perubahan perilaku. |
| 11 | `test/smoke.test.mjs`, script `test:smoke` | Tidak ada test sama sekali. | Smoke test dengan `node:test` bawaan (tanpa dependensi baru, tidak ikut tarball). | Aditif. |

Semua fitur khusus fork (`noSelfSync`, relay/send helpers, PN/LID, newsletter, album, interactive, poll/quiz, pairing code, Browsers.poucode, dll.) dipertahankan tanpa perubahan.

## 2. Masalah yang DITEMUKAN tetapi TIDAK diubah (perlu keputusan)

- **A. `trustInteropContact` tidak didefinisikan** (`lib/Socket/messages-send.js`, sekitar baris 894). Dipanggil untuk setiap pengiriman ke JID `@interop` setelah pesan terkirim, sehingga pengiriman ke kontak interop akan melempar `ReferenceError`. Tidak ada implementasi di repo; maksud "trust" tidak dapat dipastikan dari kode. Pilihan: (a) kirimkan implementasi asli jika ada, atau (b) hapus pemanggilan. Keputusan ada pada pemilik fork.
- **B. Konfigurasi lint/test/build belum berfungsi**: `lint` menunjuk ke `src/` yang tidak ada; tidak ada `tsconfig.json`, `jest.config`, `eslint.config`, atau test selain smoke test. `npm test` (jest) akan gagal karena tidak ada test.
- **C. Typings TypeScript belum tersedia** untuk API utama (hanya `WAProto/index.d.ts`). Membuat `.d.ts` berarti menambah pipeline build; belum dilakukan.
- **D. Dependensi `libsignal` berasal dari GitHub** (`github:tenka-san/libsignal-node`). Repo tersebut bisa dibaca publik (paket `libsignal` v2.0.1, file `src/curve.js`, `src/crypto.js`, `src/protobufs.js` ada). Konsumen tetap membutuhkan akses GitHub saat instal. Pertimbangkan menerbitkan fork ini ke npm.
- **E. `engines.node` `>=20.0.0`**: paket ESM-only; `require()` hanya didukung di Node 20.19+/22.12+. Mengubah `engines` menyangkut kompatibilitas, jadi tidak diubah.
- **F. Versi `1.0.0`**: tidak diubah. Jika `@poucode/baileys@1.0.0` sudah pernah terbit, publish akan ditolak; naikkan ke `1.0.1` atau lebih.
- **G. Tidak ada lockfile**; `packageManager` menyebut yarn@4.9.2 dan `resolutions` bersifat yarn-only (diabaikan npm).
- **H. Banner `console.log` saat import** (`lib/index.js`) mencetak branding dan link Telegram. Ini fitur fork, dipertahankan; perlu diketahui karena output akan tampil di log bot.
- **I. `packageManager`/script `build:*` dan `changelog:*`** tidak diuji (butuh tsc/typedoc/konfigurasi yang tidak ada).

## 3. Hasil verifikasi

| Pemeriksaan | Status | Keterangan |
|---|---|---|
| Sintaks semua `.js` (lib, WAProto, test) | Dijalankan — lulus | 110 file, 0 gagal (`node --check`). |
| ESLint core (no-undef, no-dupe-keys, no-const-assign, dll.) | Dijalankan — 1 error tersisa | Error tersisa = item A. Sebelumnya 2 error (`options` sudah diperbaiki). |
| Link ESM per file (ekspor/impor yang hilang) | Dijalankan — lulus | Semua 107 modul lib di-link tanpa error ekspor/impor hilang. |
| Pemeriksaan nama ekspor ambigu (`export *`) | Dijalankan — lulus | 0 konflik. |
| Import penuh `lib/index.js` | Dijalankan dengan **shim** | Dependensi asli (protobufjs, long, dll.) TIDAK bisa di-install di sandbox; shim hanya untuk harness. Membuktikan modul dapat dievaluasi, bukan kesesuaian API eksternal. |
| Smoke test (`test/smoke.test.mjs`) | Dijalankan dengan shim — 5/5 lulus | Belum dijalankan dengan dependensi asli. |
| `npm pack --dry-run` | Dijalankan — lulus | 116 file, ukuran ±1.4 MB; `test/` tidak ikut. |
| Verifikasi API dependensi eksternal | Dijalankan via registry/unpkg | `cache-manager` v5.7.6 (`caching(store)`, ttl ms), v7.2.9 (tidak ada `caching`), `whatsapp-rust-bridge` 0.5.4 (semua named import ada), `protobufjs` 7.5.6 (`long ^5.0.0`), `libsignal` src (ada). |
| `npm install` / jest / tsc | **Tidak dapat dijalankan** | Registry npm dan GitHub diblokir kebijakan egress sandbox (HTTP 403). |
| Cek nama `@poucode/baileys` di registry | **Belum dapat dipastikan** | Pengecekan via registry mengembalikan error (kemungkinan 404 / belum terbit). Pastikan dengan `npm view @poucode/baileys`. |
| Koneksi WhatsApp sungguhan | **Tidak dilakukan** | Sesuai instruksi (tanpa akun). |

## 4. Panduan publish (ringkas)

1. `node -v` (harus 20+), lalu `cd Baileys-main`.
2. `npm install` (butuh akses ke registry npm dan GitHub).
3. `npm run test:smoke`
4. `npm pack --dry-run` — pastikan tidak ada file rahasia.
5. `npm view @poucode/baileys` — harus menunjukkan 404 jika belum terbit. Pastikan Anda memiliki scope/organisasi `poucode` di npm.
6. `npm login` (lakukan sendiri, termasuk OTP).
7. `npm publish --access public` (scoped package bersifat private secara default).
8. Verifikasi: `npm view @poucode/baileys version`, lalu instal di folder kosong dan jalankan `node -e "import('@poucode/baileys').then(m => console.log(typeof m.default))"`.
