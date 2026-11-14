# WaBail 1.1.0 — Audit, Bug & Ringkasan Perubahan

Package `@poucode/baileys` · Brand WaBail · Developer PouCode · https://github.com/pou-code/WaBail

## Status pengujian (jujur)
- DILAKUKAN: `node --check` pada semua file `lib/**/*.js` (kecuali `newsletter.js`, tidak dibuka), pengecekan import relatif (0 import hilang), tes logika murni `jid-utils` dan `buildZipArchive`/`crc32` (valid via `unzip -t`).
- TIDAK DILAKUKAN: semua pengujian dengan WhatsApp sungguhan (koneksi, pairing code, reconnect, private chat, grup, media, poll/quiz/interactive, newsletter/channel, relay). Dependency npm tidak bisa dipasang di lingkungan ini (jaringan mati), jadi library belum pernah di-`import` penuh di sini. Jalankan `npm install && npm run test:smoke` lalu uji manual di nomor uji.

## Bug dan penyebab
1. Private chat tidak dibalas (grup normal). Jalur TERIMA (`decode-wa-message.js`, `jid-utils.js`, `handleMessage`, `process-message.js`) identik di kedua project, jadi bukan di parser. Perbedaan nyata ada di jalur KIRIM 1:1: wbails mengirim `cstoken` (HMAC dari NCT salt) untuk chat 1:1 yang belum punya `tctoken`, WaBail tidak. WaBail juga tidak menyimpan NCT salt dari app-state sync. Dugaan terkuat (BELUM dibuktikan di WhatsApp): balasan 1:1 ditolak server (kode 463) sedangkan grup memakai jalur sender-key sehingga lolos. Perbaikan: simpan `nct-salt` di `chats.js` dan lampirkan `cstoken` di `relayMessage` (bisa dimatikan dengan `{ cstoken: false }`).
2. `relayMessage` WaBail bisa "sukses" tanpa mengirim ke lawan bicara bila resolusi device kosong; versi WaBail sudah melempar 421 untuk kasus ini (dipertahankan). Jika log Anda menunjukkan `no recipient devices resolved`, penyebabnya ada di USync device query, bukan di bot.
3. `graphql.js` (wbails): `import Boom from '@hapi/boom'` lalu `new Boom(...)`. @hapi/boom CJS, jadi saat error GraphQL terjadi akan `TypeError: Boom is not a constructor`. Diperbaiki menjadi `import { Boom }`.
4. wbails mengganti arti opsi `participant` menjadi `participants` di jalur retry (`messages-recv.js`) dan `relayMessage`. Itu mengubah API; WaBail (`participant`) dipertahankan agar kompatibel.
5. Nama file USync tidak konsisten antar project (`UsyncLIDProtocol.js` vs `USyncLIDProtocol.js`) sehingga bisa gagal di filesystem case-sensitive. Disatukan ke `USync*`.
6. wbails `generateMessageID` memakai `randomBytes(9.5)` (ukuran non-integer); WaBail (`randomBytes(10)`) dipertahankan. wbails `cache-manager@^7` dengan TTL detik vs kode WaBail untuk `^5` dengan TTL ms; WaBail dipertahankan.

## Tidak di-port dengan sengaja
- `checkBan` (wbails `socket.js`): memanggil server pihak ketiga dengan header aneh dan mengirim nomor telepon; juga tidak meng-`await` `res.json()`.
- `autoFollow`, `Types/Mex.js` ID newsletter, `mex.js` wbails, `username.js` wbails (ID query berbeda, tidak bisa diverifikasi): dipertahankan versi WaBail agar newsletter dan fitur yang sudah jalan tidak berubah.
- Perbaikan retry/SKDM dan opsi `isSecret`/`protected` wbails: mengubah arti `participant`.

## Ringkasan perubahan
- Dasar: WaBail-main. Ditambahkan dari wbails: layer `privacy`, `interop`, `graphql`, `message-builder`; `WAUSync` (superset: business, picture, text status, sidelist, feature); `sticker.js`, sticker pack dan raw-proto pada `messages.js`; `crc32`/`buildZipArchive`/`resolveStickerBuffer`; `useSqliteAuthState`; `writeToFile`/`readFromFile` pada in-memory store; `stickerPackMessage` di `getMessageType`.
- `Socket/index.js`: komposisi layer lengkap. `package.json`: versi 1.1.0, deskripsi, keyword; dependency tetap milik WaBail.
- `lib/Socket/newsletter.js` milik WaBail tidak diedit, tidak dipindah, tidak diganti (mtime dan ukuran 10974 byte sama dengan arsip asli). Catatan: dua kali pencarian grep saya tanpa sengaja memindai isi file newsletter.js (WaBail sekali, wbails sekali, satu baris tampil); tidak ada isi yang dipakai dan tidak ada perubahan.
- Branding: `senderName` di `luxu.js` menjadi PouCode. Prefix ID pesan `XZCYYX-` masih ada di `generics.js`/`socket.js` dan belum diganti.
