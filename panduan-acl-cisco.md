# Panduan Lengkap Access Control List (ACL) — Cisco IOS

> Catatan pribadi hasil praktik langsung di Packet Tracer, termasuk kesalahan yang pernah dialami supaya tidak terulang.

---

## 1. Konsep Dasar

**ACL** adalah daftar aturan yang dipasang di router untuk **mengizinkan (`permit`)** atau **memblokir (`deny`)** traffic berdasarkan kriteria tertentu (source, destination, protokol, port).

### Jenis ACL
| Jenis | Nomor | Bisa Filter Berdasarkan |
|---|---|---|
| Standard ACL | 1–99 | Source IP saja (kasar) |
| Extended ACL | 100–199 | Source IP, Destination IP, Protokol, Port (presisi) |

### 3 Aturan Emas ACL
1. **Implicit deny** — kalau traffic tidak cocok satupun rule, otomatis **diblokir** (tidak terlihat sebagai baris, tapi selalu ada di akhir).
2. **Top-down, berhenti di match pertama** — urutan rule sangat menentukan. Rule lebih spesifik ditaruh duluan, rule umum (`permit ip any any`) di paling akhir.
3. **Harus di-apply ke interface** — bikin rule saja tidak cukup, harus di-apply dengan arah `in` (masuk ke router) atau `out` (keluar dari router).

### ACL itu STATELESS (penting!)
ACL classic **tidak mengenali** bahwa sebuah paket adalah "balasan" dari koneksi yang sudah diizinkan. Setiap paket dicek independen berdasarkan source-destination-protokolnya sendiri.

**Contoh kasus nyata yang pernah dialami:**
Rule `deny icmp <Mahasiswa> <Dosen>` ternyata membuat **Dosen juga tidak bisa ping ke Mahasiswa** — padahal rule hanya menyasar arah Mahasiswa→Dosen. Penyebabnya: waktu Dosen ping ke Mahasiswa, PC Mahasiswa membalas dengan **ICMP Echo Reply** (source=Mahasiswa, destination=Dosen) — ini match rule `deny` yang sama walau isinya cuma "balasan".

**Solusi:** tambahkan keyword `echo` di akhir rule ICMP supaya hanya menyasar **Echo Request** (permintaan awal), bukan **Echo Reply** (balasan):
```
access-list 100 deny icmp <source> <wildcard> <destination> <wildcard> echo
```

---

## 2. Wildcard Mask

Wildcard mask adalah **kebalikan** dari subnet mask (bit 1 dan 0 ditukar).

**Rumus cepat:** `255 - (nilai oktet subnet mask)`

| Prefix | Subnet Mask (oktet terakhir) | Wildcard Mask |
|---|---|---|
| /24 | 0 | 255 |
| /27 | 224 | 31 |
| /28 | 240 | 15 |
| /29 | 248 | 7 |
| /30 | 252 | 3 |

**Cara baca:** wildcard `0.0.0.31` di `192.168.20.0` artinya "3 oktet pertama harus persis sama, oktet terakhir bebas 0–31".

**Kasus khusus:**
- 1 host spesifik → gunakan keyword `host` (otomatis wildcard `0.0.0.0`), contoh: `host 192.168.20.50`
- Semua IP → gunakan `any` (tanpa alamat, tanpa wildcard)

**⚠️ Kesalahan umum:** Wildcard **BUKAN** dihitung dari "network address ke broadcast address" (misal `47 - 32 = 15` itu kebetulan sama, tapi logikanya salah). Selalu pakai rumus `255 - subnet mask`, supaya tidak keliru di kasus lain (misal /29 network `.48` broadcast `.55`, kalau dihitung `55-48=7` kebetulan sama juga — tapi jangan diandalkan sebagai rumus).

---

## 3. Skeleton Command Extended ACL

```
access-list [NOMOR] [permit/deny] [PROTOKOL] [SOURCE] [SOURCE-WILDCARD] [DESTINATION] [DESTINATION-WILDCARD] [PORT/opsional]
```

| Slot | Pilihan |
|---|---|
| PROTOKOL | `ip` (semua), `tcp`, `udp`, `icmp` |
| SOURCE/DESTINATION | network+wildcard, `host x.x.x.x`, atau `any` |
| PORT | hanya untuk tcp/udp, contoh `eq 80` |

**Contoh-contoh:**
```
access-list 100 deny ip 192.168.20.0 0.0.0.31 host 192.168.20.50
access-list 100 deny icmp 192.168.20.0 0.0.0.31 192.168.20.32 0.0.0.15 echo
access-list 100 permit tcp any host 192.168.20.49 eq 80
access-list 100 permit ip any any
```

---

## 4. Command Reference — Membuat, Melihat, Mengubah, Menghapus

### Masuk mode konfigurasi
```
enable
configure terminal
```

### Membuat ACL (baris baru otomatis ditambah ke bawah)
```
access-list 100 deny ip 192.168.20.0 0.0.0.31 host 192.168.20.50
access-list 100 permit ip any any
```
> IOS otomatis memberi nomor urut internal (biasanya kelipatan 10) untuk tiap baris.

### Apply ACL ke interface
```
interface FastEthernet0/0
ip access-group 100 in
exit
```
- `in` = cek traffic yang **masuk** ke router dari interface itu (paling umum, dipasang dekat sumber traffic)
- `out` = cek traffic yang **keluar** dari router lewat interface itu

### Melihat isi ACL
```
show access-lists
show access-lists 100
```
Menampilkan semua rule beserta **match counter** (berapa kali rule itu kena).

### Melihat apakah ACL sudah ter-apply ke interface
```
show ip interface FastEthernet0/0
```
Cari baris `Incoming access list is 100` atau `Outgoing access list is 100`.

### Menghapus SATU baris rule tertentu
Tulis ulang persis command-nya dengan `no` di depan:
```
no access-list 100 deny ip 192.168.20.0 0.0.0.31 host 192.168.20.50
```

### Menghapus SELURUH ACL (semua baris nomor itu)
```
no access-list 100
```
**⚠️ PENTING:** Command `ip access-group 100 in` di interface **TIDAK otomatis terhapus** kalau ACL-nya dihapus. Kalau kamu bikin ulang ACL dengan nomor yang SAMA (100), otomatis nyambung lagi ke interface tanpa perlu apply ulang. Tapi kalau ACL nomor itu kosong/tidak ada, interface akan berperilaku **permit all** (tanpa pembatasan).

### Melepas ACL dari interface (tanpa menghapus ACL-nya)
```
interface FastEthernet0/0
no ip access-group 100 in
exit
```

### Menyisipkan rule DI TENGAH urutan yang sudah ada
Kalau ada nomor urut yang masih kosong (misal ada gap 10, 30, sisa 20 kosong):
```
access-list 100 20 permit icmp host 192.168.20.5 host 192.168.20.49
```
Kalau tidak ada gap tersedia, cara paling aman: hapus semua (`no access-list 100`) lalu tulis ulang semua baris **dengan urutan yang benar** dari awal.

---

## 5. Prosedur Debugging ACL (checklist)

Ketika ping/koneksi gagal dan dicurigai karena ACL, urutan cek:

1. **Cek koneksi fisik dulu** — pastikan bukan masalah kabel/switch (test ping dalam 1 subnet yang sama, yang tidak lewat ACL sama sekali).
2. **Cek IP address & default gateway di end device** (`ipconfig`) — pastikan valid, tidak `0.0.0.0`, tidak duplikat.
3. **Cek status interface router** (`show ip interface brief`) — pastikan `up/up`, ada IP yang benar.
4. **Cek isi ACL** (`show access-lists 100`) — pastikan urutan rule benar, `permit ip any any` di paling bawah.
5. **Cek ACL sudah ter-apply ke interface yang benar** (`show ip interface <nama>`) — interface yang dipasang harus yang **terhubung langsung ke sumber traffic** yang mau difilter.
6. **Test ping bertahap**, jangan langsung lintas-banyak-segmen:
   - PC → gateway sendiri
   - PC → PC lain (subnet sama)
   - PC → server/segmen lain (lintas ACL)
7. **Bandingkan match counter sebelum-sesudah test** — kalau counter di rule yang diharapkan tidak naik padahal traffic gagal, berarti yang bekerja itu **implicit deny** (rule tak terlihat), bukan rule yang kamu tulis.
8. **Ingat sifat stateless ACL untuk ICMP** — kalau salah satu arah tiba-tiba ikut gagal padahal rule cuma untuk 1 arah, curigai efek reply/request seperti kasus di atas.

---

## 6. Cara Membaca Output Ping — Jangan Tertukar!

| Output | Artinya |
|---|---|
| `Reply from <IP TUJUAN>: bytes=...` | Berhasil, balasan asli dari device tujuan |
| `Reply from <IP GATEWAY>: Destination host unreachable` | **GAGAL** — router aktif menolak (biasanya karena ACL atau tidak ada rute). Perhatikan: IP yang "reply" itu IP ROUTER, bukan tujuan! |
| `Request timed out` | **GAGAL** — tidak ada balasan sama sekali (masalah dasar: routing, gateway, fisik — BUKAN ACL, karena ACL biasanya mengirim pesan unreachable, bukan diam saja) |

---

## 7. Contoh Kasus Lengkap (Studi Kasus Kampus)

**Skenario:**
- Mahasiswa BOLEH akses Server Akademik, TIDAK BOLEH akses Server Nilai
- Mahasiswa TIDAK BOLEH memulai ping ke Network Dosen (tapi Dosen tetap bisa ping ke Mahasiswa)

**ACL final yang benar (urutan penting!):**
```
access-list 100 deny ip 192.168.20.0 0.0.0.31 host 192.168.20.49
access-list 100 deny icmp 192.168.20.0 0.0.0.31 192.168.20.32 0.0.0.15 echo
access-list 100 permit ip any any
```

**Apply:**
```
interface FastEthernet0/0
ip access-group 100 in
```

**Hasil tervalidasi:**
- Mahasiswa → Server Nilai: Blocked ✓
- Mahasiswa → Server Akademik: Allowed ✓
- Mahasiswa → memulai ping ke Dosen: Blocked ✓
- Dosen → ping ke Mahasiswa (dan balasannya): Allowed ✓

---

## 8. Hal yang Masih Perlu Diingat / Sering Lupa
- [ ] `no shutdown` di interface (default-nya mati)
- [ ] `permit ip any any` HARUS di paling bawah
- [ ] Hapus ACL tidak otomatis melepas dari interface
- [ ] Wildcard mask dihitung dari `255 - subnet mask`, bukan dari range host
- [ ] ICMP itu stateless — gunakan `echo` kalau hanya mau blokir satu arah (request), bukan balasannya (reply)
- [ ] `show access-lists` untuk cek match counter — cara tercepat verifikasi rule mana yang benar-benar bekerja
