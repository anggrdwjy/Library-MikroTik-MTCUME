# MTCUME - Panduan Implementasi dan Operasi

## Daftar Isi
- [Modul 1: PPP - Fondasi Manajemen User](https://github.com/anggrdwjy/Library-MikroTik-MTCUME/tree/main#modul-1-ppp---fondasi-manajemen-user)
- [Modul 2: PPTP dan L2TP - Tunnel PPP Antar Site](https://github.com/anggrdwjy/Library-MikroTik-MTCUME/tree/main#modul-2-pptp-dan-l2tp---tunnel-ppp-antar-site)
- [Modul 3: PPPoE - Layanan Broadband ISP](https://github.com/anggrdwjy/Library-MikroTik-MTCUME/tree/main#modul-3-pppoe---layanan-broadband-isp)
- [Modul 4: Bridging - Memperpanjang Layer 2 Melalui Tunnel](https://github.com/anggrdwjy/Library-MikroTik-MTCUME/tree/main#modul-4-bridging---memperpanjang-layer-2-melalui-tunnel)
- [Modul 5: IPsec - Tunnel Terenkripsi Antar Router](https://github.com/anggrdwjy/Library-MikroTik-MTCUME/tree/main#modul-5-ipsec---tunnel-terenkripsi-antar-router)
- [Modul 6: Hotspot - Manajemen User Akses WiFi dan LAN](https://github.com/anggrdwjy/Library-MikroTik-MTCUME/tree/main#modul-6-hotspot---manajemen-user-akses-wifi-dan-lan)
- [Modul 7: RADIUS - Manajemen User Terpusat](https://github.com/anggrdwjy/Library-MikroTik-MTCUME/tree/main#modul-7-radius---manajemen-user-terpusat)

## Modul 1: PPP - Fondasi Manajemen User

### Tujuan Implementasi

Membangun fondasi PPP yang digunakan oleh PPTP, L2TP, dan PPPoE. Apabila profile, pool, dan secret tidak tertata dengan rapi, seluruh modul berikutnya akan sulit dioperasikan.

### Studi Kasus

ISP lokal melayani 200 pelanggan rumahan. Terdapat dua paket: Gold 10 Mbps dan Silver 5 Mbps. Setiap pelanggan hanya diizinkan untuk melakukan login dalam satu sesi. Koneksi harus diputus secara otomatis setelah 1 hari dan memaksa autentikasi ulang. TCP MSS harus disesuaikan agar browsing tidak terputus-putus.

### Konsep Kunci

- **PPP Profile:** Template yang mengatur local-address, remote-address, rate-limit, encryption, session-timeout, TCP MSS, dan filter.
- **PPP Secret:** Akun user. Menunjuk ke profile. Dapat mengesampingkan local-address, remote-address, dan routes.
- **IP Pool:** Range alamat yang dibagikan ke client PPP. Gunakan `next-pool` apabila satu pool telah habis.

### Perbandingan RouterOS v6 vs v7

| Aspek | RouterOS v6 (6.49.x) | RouterOS v7 (7.x) |
| --- | --- | --- |
| Sintaks dasar PPP | `/ppp profile`, `/ppp secret`, `/ip pool` | Tidak terdapat perubahan besar |
| Rate-limit | `rate-limit=10M/10M` di profile | Tidak terdapat perbedaan mendasar, tetapi queue handling lebih konsisten dengan Cake / FQ-CoDel |
| Encryption | `use-encryption=yes / required` | Tidak terdapat perbedaan, tetap didukung untuk PPTP / PPPoE |
| Only-one | `only-one=yes` di profile | Sama |
| Session-timeout | Didukung di profile | Sama |
| Monitoring | `/ppp active print` | Sama, tambahan kolom di WinBox v7 lebih lengkap |
| Catatan operasi | Stabil dan banyak digunakan di ISP lama | Direkomendasikan untuk instalasi baru, pastikan firmware dan package sama di semua router |

### Konfigurasi

#### Langkah 1 - Pembuatan IP Pool

```routeros
/ip pool add name=pool-gold ranges=10.10.10.10-10.10.10.200
/ip pool add name=pool-silver ranges=10.10.20.10-10.10.20.200
/ip pool add name=pool-mgmt ranges=10.99.99.50-10.99.99.100
```

#### Langkah 2 - Pembuatan PPP Profile

```routeros
/ppp profile add name=profile-gold \
  local-address=10.10.10.1 \
  remote-address=pool-gold \
  rate-limit=10M/10M \
  change-tcp-mss=yes \
  use-encryption=required \
  only-one=yes \
  session-timeout=1d \
  dns-server=8.8.8.8,1.1.1.1

/ppp profile add name=profile-silver \
  local-address=10.10.20.1 \
  remote-address=pool-silver \
  rate-limit=5M/5M \
  change-tcp-mss=yes \
  use-encryption=required \
  only-one=yes \
  session-timeout=1d \
  dns-server=8.8.8.8,1.1.1.1

/ppp profile add name=profile-default \
  local-address=10.10.10.1 \
  remote-address=pool-silver \
  change-tcp-mss=yes \
  only-one=yes
```

#### Langkah 3 - Pembuatan PPP Secret

```routeros
/ppp secret add name=pelanggan001 password=PasswordStrong001 \
  service=any profile=profile-gold \
  comment="Gold - Jl. Merdeka 12"

# Contoh user dengan IP statis dan route khusus
/ppp secret add name=pelanggan002 password=PasswordStrong002 \
  service=pppoe profile=profile-gold \
  local-address=10.10.10.1 remote-address=10.10.10.50 \
  routes="192.168.50.0/24 10.10.10.50 1" \
  comment="Gold - IP Statis - Kantor Cabang"

# User untuk teknisi / admin
/ppp secret add name=teknisi01 password=PasswordStrong003 \
  service=any profile=profile-default \
  comment="Teknisi Lapangan"
```

#### Langkah 4 - Batasi Service yang Diizinkan untuk Login (Opsional, Keamanan)

```routeros
# Apabila secret hanya untuk PPPoE, jangan biarkan login melalui PPTP / L2TP
/ppp secret set [find name=pelanggan001] service=pppoe
```

### Verifikasi

```routeros
/ip pool print
/ppp profile print
/ppp secret print
/ppp active print
# Periksa siapa yang sedang online, IP, uptime, dan profile yang digunakan
/ppp active print where name=pelanggan001
```

### Operasi dan Troubleshooting

- **User tidak dapat melakukan login, padahal password benar:** Periksa `service` di secret. Jika `service=pppoe` tetapi client dial melalui PPTP, login ditolak. Periksa juga `only-one=yes` - jika sesi lama masih menggantung, terputus dulu.
- **Pelanggan dapat IP salah:** Periksa `remote-address` di profile dan `pool` yang dirujuk. Apabila secret memiliki `remote-address` statis, itu akan mengalahkan pool.
- **Browsing lambat / situs tertentu tidak terbuka:** Biasanya MSS. Pastikan `change-tcp-mss=yes` di profile. Di PPPoE, MTU/MRU 1480 atau 1492 sering lebih aman daripada 1500.
- **Sesi tidak terputus:** Periksa `session-timeout` dan `idle-timeout` di profile. Untuk memutuskan paksa: `/ppp active remove [find name=pelanggan001]`
- **Operasi harian:** Pantaulah `/ppp active print count-only`, `/ip pool used print`, dan log PPP di `/log print where topics~"ppp"`.

## Modul 2: PPTP dan L2TP - Tunnel PPP Antar Site

### Tujuan Implementasi

Membangun tunnel PPP untuk menghubungkan router cabang ke pusat. PPTP digunakan untuk kompatibilitas lama, L2TP digunakan sebagai standar yang lebih aman, terutama L2TP/IPsec.

### Studi Kasus

Satu cabang di kota lain harus terhubung ke kantor pusat untuk akses server internal 192.168.88.0/24. Cabang hanya memiliki alamat IP publik dinamis. Koneksi harus otomatis tersambung kembali jika putus. Terdapat dua kebutuhan:

- **A:** Tunnel darurat cepat (PPTP) untuk teknisi lapangan.
- **B:** Tunnel permanen cabang (L2TP/IPsec) untuk operasional.

### Konsep Kunci

- **PPTP:** Mudah diimplementasikan, tetapi enkripsi MPPE lemah. Jangan digunakan untuk data sensitif. Hanya untuk legacy atau darurat.
- **L2TP:** Tidak terdapat enkripsi sendiri. Harus dibungkus IPsec agar aman. Lebih stabil untuk NAT.
- **Dial-on-demand:** Client hanya dial saat terdapat traffic. Untuk cabang permanen, nonaktifkan dial-on-demand.
- **Static interface vs dynamic:** Interface PPTP/L2TP dinamis dibuat otomatis saat client tersambung. Static interface digunakan jika diperlukan monitoring, queue, atau firewall yang konsisten.

### Perbandingan RouterOS v6 vs v7

| Aspek | RouterOS v6 (6.49.x) | RouterOS v7 (7.x) |
| --- | --- | --- |
| PPTP Server / Client | Tersedia penuh | Tersedia, tetapi dianggap legacy dan tidak direkomendasikan |
| L2TP Server / Client | Tersedia, `use-ipsec` pada server | Sama, tetapi integrasi IPsec lebih ketat |
| L2TP + IPsec Secret | `ipsec-secret` pada server L2TP | Sama |
| Authentication | pap, chap, mschap1, mschap2 | Sama, mschap2 direkomendasikan untuk PPTP/L2TP |
| NAT-Traversal | Otomatis jika IPsec digunakan | Sama, lebih stabil di v7 |
| Rekomendasi | PPTP masih sering ditemui | Untuk baru, pilih L2TP/IPsec atau WireGuard / IPsec murni |

### Konfigurasi - PPTP Server pada R-CORE

```routeros
# Profile khusus PPTP
/ppp profile add name=profile-pptp \
  local-address=10.0.0.1 remote-address=pool-pptp \
  rate-limit=20M/20M change-tcp-mss=yes \
  use-encryption=required only-one=yes

/ip pool add name=pool-pptp ranges=10.0.0.10-10.0.0.50

# Aktifkan PPTP Server
/interface pptp-server server set enabled=yes \
  default-profile=profile-pptp \
  authentication=mschap2 \
  keepalive-timeout=30

# Secret untuk teknisi / cabang PPTP
/ppp secret add name=cabang-pptp password=PasswordStrongPPTP \
  service=pptp profile=profile-pptp

# Static interface (opsional, agar mudah di-firewall dan di-queue)
/interface pptp-server add name=pptp-cabang user=cabang-pptp
```

### Konfigurasi - PPTP Client pada R-BRANCH

```routeros
/interface pptp-client add name=pptp-to-core \
  connect-to=203.0.113.2 user=cabang-pptp password=PasswordStrongPPTP \
  profile=profile-pptp \
  add-default-route=no dial-on-demand=no \
  allow=mschap2 disabled=no

# Route ke LAN pusat melalui tunnel
/ip route add dst-address=192.168.88.0/24 gateway=pptp-to-core distance=1 comment="Melalui PPTP Core"

# Route pada R-CORE balik ke LAN cabang
# Jalankan pada R-CORE:
/ip route add dst-address=192.168.10.0/24 gateway=10.0.0.10 distance=1 comment="Melalui PPTP Branch"
```

### Konfigurasi - L2TP/IPsec Server pada R-CORE (Rekomendasi Produksi)

```routeros
/ppp profile add name=profile-l2tp \
  local-address=10.0.1.1 remote-address=pool-l2tp \
  change-tcp-mss=yes use-encryption=yes only-one=yes

/ip pool add name=pool-l2tp ranges=10.0.1.10-10.0.1.50

/interface l2tp-server server set enabled=yes \
  default-profile=profile-l2tp \
  authentication=mschap2 \
  use-ipsec=required \
  ipsec-secret=IpsecSecretStrong123 \
  keepalive-timeout=30

/ppp secret add name=cabang-l2tp password=PasswordStrongL2TP \
  service=l2tp profile=profile-l2tp
```

### Konfigurasi - L2TP/IPsec Client pada R-BRANCH

```routeros
/interface l2tp-client add name=l2tp-to-core \
  connect-to=203.0.113.2 user=cabang-l2tp password=PasswordStrongL2TP \
  profile=profile-l2tp \
  use-ipsec=yes ipsec-secret=IpsecSecretStrong123 \
  add-default-route=no dial-on-demand=no disabled=no

/ip route add dst-address=192.168.88.0/24 gateway=l2tp-to-core distance=1 comment="Melalui L2TP Core"
```

### Verifikasi

```routeros
/interface pptp-server print
/interface l2tp-server print
/interface pptp-client print
/interface l2tp-client print
/ppp active print where service=l2tp
/ping 10.0.1.10 src-address=10.0.1.1
/ping 192.168.88.1 src-address=192.168.10.1
```

### Operasi dan Troubleshooting

- **L2TP tidak tersambung, PPTP tersambung:** Biasanya IPsec blocking. Periksa firewall UDP 500, 4500, dan protokol IPsec (ESP) pada kedua sisi. Di L2TP/IPsec, pastikan `ipsec-secret` sama persis.
- **Tunnel tersambung tetapi tidak dapat ping LAN:** Periksa route pada kedua sisi. Tunnel PPP hanya memberi route ke IP tunnel, bukan ke LAN. Tambahkan route statis ke subnet LAN lawan.
- **Tunnel sering terputus:** Periksa `keepalive-timeout`, kualitas link, dan NAT timeout. Untuk cabang permanen, set `dial-on-demand=no` dan tambahkan Netwatch untuk auto-reconnect.
- **PPTP tersambung tetapi lambat:** Periksa MTU/MRU. Turunkan ke 1400-1450 untuk PPTP.
- **Keamanan:** Jangan membuka PPTP (TCP 1723) ke internet tanpa firewall whitelist. Batasi hanya dari IP cabang yang dikenal.

## Modul 3: PPPoE - Layanan Broadband ISP

### Tujuan Implementasi

Membangun PPPoE Server yang aman, scalable, dan mudah dioperasikan untuk pelanggan FTTH / broadband. Ini modul inti untuk ISP.

### Studi Kasus

ISP melayani 500 pelanggan FTTH melalui OLT. Setiap OLT terhubung melalui VLAN ke R-CORE. Pelanggan dial PPPoE dari ONT / router rumah. Terdapat paket Gold 20 Mbps, Silver 10 Mbps, dan Bronze 5 Mbps. Perlu ECMP untuk load balancing ke dua upstream. Server PPPoE tidak diizinkan untuk memiliki alamat IP pada interface PPPoE agar tidak dapat dipindai secara langsung.

### Konsep Kunci

- **PPPoE Server:** Mendengarkan pada interface (biasanya VLAN). Satu server per VLAN / per OLT.
- **PPPoE Client:** Router pelanggan yang dial ke server.
- **Service-Name:** Nama layanan yang diiklankan server. Client dapat memilih service-name tertentu.
- **One-Session-Per-Host:** Mencegah satu MAC dial banyak sesi.
- **ECMP:** Equal Cost Multi Path untuk membagi traffic ke dua gateway PPPoE / upstream.

### Perbandingan RouterOS v6 vs v7

| Aspek | RouterOS v6 (6.49.x) | RouterOS v7 (7.x) |
| --- | --- | --- |
| PPPoE Server | `/interface pppoe-server server` | Sama |
| PPPoE Client | `/interface pppoe-client` | Sama |
| VLAN + PPPoE | Didukung penuh | Sama, performa VLAN + PPPoE lebih baik di v7 |
| ECMP | `check-gateway=ping` di route | Sama, tambahan dukungan ECMP dengan routing-table dan PCC yang lebih stabil |
| MTU / MRU | Default 1480, sering diubah ke 1492 | Sama, tetapi v7 lebih ketat soal MTU di PPPoE + VLAN |
| RADIUS | Integrasi melalui `/ppp aaa` | Sama, User Manager v5 lebih stabil di v7 |
| Catatan | Sangat umum di ISP lama | Direkomendasikan untuk instalasi baru |

### Konfigurasi - PPPoE Server pada R-CORE

#### Langkah 1 - VLAN per OLT

```routeros
/interface vlan add name=vlan100-olt1 vlan-id=100 interface=ether2 comment="OLT1 - PPPoE"
/interface vlan add name=vlan101-olt2 vlan-id=101 interface=ether2 comment="OLT2 - PPPoE"

# JANGAN berikan IP ke VLAN PPPoE. Ini untuk keamanan server PPPoE.
# Biarkan tanpa IP address.
```

#### Langkah 2 - Profile PPPoE

```routeros
/ppp profile add name=profile-pppoe-gold \
  local-address=10.10.10.1 remote-address=pool-gold \
  rate-limit=20M/20M \
  change-tcp-mss=yes use-encryption=required \
  only-one=yes session-timeout=1d \
  dns-server=8.8.8.8,1.1.1.1 \
  comment="PPPoE Gold 20M"

# Untuk PPPoE, tambahkan juga incoming/outgoing filter jika perlu
# Contoh: blokir akses antar pelanggan PPPoE (client isolation)
/ppp profile set profile-pppoe-gold \
  incoming-filter=pppoe-in outgoing-filter=pppoe-out

/ip firewall filter add chain=forward action=drop \
  src-address=10.10.10.0/24 dst-address=10.10.10.0/24 \
  comment="Isolasi antar pelanggan PPPoE" place-before=0
```

#### Langkah 3 - PPPoE Server

```routeros
/interface pppoe-server server add \
  service-name=ISP-PPPoE \
  interface=vlan100-olt1 \
  default-profile=profile-pppoe-gold \
  authentication=pap,chap,mschap2 \
  one-session-per-host=yes \
  disabled=no \
  comment="PPPoE OLT1"

# Tambahkan server kedua untuk OLT2
/interface pppoe-server server add \
  service-name=ISP-PPPoE \
  interface=vlan101-olt2 \
  default-profile=profile-pppoe-gold \
  authentication=pap,chap,mschap2 \
  one-session-per-host=yes \
  disabled=no
```

#### Langkah 4 - Secret Pelanggan PPPoE

```routeros
/ppp secret add name=pelanggan101 password=PasswordCustomer101 \
  service=pppoe profile=profile-pppoe-gold \
  comment="PPPoE Gold - OLT1 - Pelanggan 101"

/ppp secret add name=pelanggan102 password=PasswordCustomer102 \
  service=pppoe profile=profile-pppoe-gold \
  comment="PPPoE Gold - OLT1 - Pelanggan 102"
```

#### Langkah 5 - ECMP untuk Dua Upstream

```routeros
# Contoh: dua interface PPPoE upstream atau dua gateway WAN
# Metode 1 - ECMP sederhana dengan check-gateway
/ip route add dst-address=0.0.0.0/0 gateway=203.0.113.1,203.0.113.5 \
  check-gateway=ping distance=1 comment="ECMP Upstream ISP"

# Metode 2 - ECMP untuk interface PPPoE client (jika R-CORE juga sebagai client)
/interface pppoe-client add name=pppoe-up1 interface=ether1 \
  user=uplink1 password=UplinkKuat1 profile=default \
  add-default-route=no disabled=no
/interface pppoe-client add name=pppoe-up2 interface=ether1 \
  user=uplink2 password=UplinkKuat2 profile=default \
  add-default-route=no disabled=no

/ip route add dst-address=0.0.0.0/0 gateway=pppoe-up1,pppoe-up2 \
  check-gateway=ping distance=1 comment="ECMP PPPoE Upstream"
```

### Konfigurasi - PPPoE Client di Router Pelanggan

```routeros
/interface pppoe-client add name=pppoe-out \
  interface=ether1 user=pelanggan101 password=PasswordCustomer101 \
  profile=default service-name=ISP-PPPoE \
  add-default-route=yes use-peer-dns=yes \
  dial-on-demand=no disabled=no

# Pastikan NAT untuk LAN pelanggan
/ip firewall nat add chain=srcnat out-interface=pppoe-out action=masquerade comment="NAT Pelanggan"
```

### Verifikasi

```routeros
/interface pppoe-server server print
/interface pppoe-server print
/ppp active print where service=pppoe
/ip route print where gateway~"pppoe"
/ping 8.8.8.8 src-address=10.10.10.10
# Dari sisi pelanggan: periksa IP, gateway, DNS
/ip address print where interface=pppoe-out
```

### Operasi dan Troubleshooting

- **PPPoE tidak dapat dial, log PADI timeout:** Periksa VLAN, service-name, dan apakah server PPPoE `disabled=no`. Periksa juga kabel / OLT. Gunakan Torch di VLAN untuk melihat PADI/PADO.
- **Dapat melakukan dial tetapi tidak memperoleh akses internet:** Periksa NAT, firewall forward, dan route default. Periksa `local-address` di profile - jika salah, pelanggan tidak memiliki gateway yang benar.
- **Pelanggan sering terputus:** Periksa `session-timeout`, kualitas fiber, dan `keepalive`. Periksa log: `/log print where topics~"pppoe"`
- **Satu pelanggan dapat dial berkali-kali:** Pastikan `one-session-per-host=yes` di PPPoE Server dan `only-one=yes` di profile.
- **ECMP tidak seimbang:** ECMP RouterOS membagi berdasarkan connection, bukan per paket. Untuk pembagian lebih merata, kombinasikan dengan PCC di mangle.
- **Keamanan:** Jangan memberikan alamat IP pada interface VLAN PPPoE. Batasi akses WinBox / API hanya dari VLAN manajemen.

## Modul 4: Bridging - Memperpanjang Layer 2 Melalui Tunnel

### Tujuan Implementasi

Menghubungkan dua site di Layer 2 seolah-olah satu switch, melalui tunnel. Ini digunakan untuk CCTV, VLAN yang sama, atau aplikasi yang memerlukan broadcast / DHCP di dua lokasi.

### Studi Kasus

Terdapat dua gedung: Gedung A (pusat) dan Gedung B (cabang). CCTV di Gedung B harus satu subnet dengan NVR di Gedung A: 192.168.20.0/24. Tidak dapat menarik kabel secara langsung. Solusi yang diterapkan adalah membangun tunnel L2TP, lalu jembatani Layer 2 melalui EoIP, VPLS, atau BCP. Pilihlah salah satu sesuai kebutuhan.

### Konsep Kunci

- **EoIP:** Ethernet over IP. Membungkus frame Ethernet di dalam IP. Paling mudah, tidak diperlukan MPLS.
- **VPLS:** Virtual Private LAN Service. Diperlukan MPLS / LDP. Lebih scalable untuk banyak site.
- **BCP:** Bridge Control Protocol. Mem-bridge interface PPP langsung. Cocok untuk L2TP yang sudah ada.
- **Multilink / MLPPP:** Menggabungkan beberapa link PPP menjadi satu link logis yang lebih besar dan redundan.
- **MRRU:** Max Received Reconstructed Unit. Harus sama pada kedua sisi untuk multilink.

### Perbandingan RouterOS v6 vs v7

| Aspek | RouterOS v6 (6.49.x) | RouterOS v7 (7.x) |
| --- | --- | --- |
| EoIP | `/interface eoip` | Sama |
| VPLS | `/interface vpls`, diperlukan MPLS + LDP | Sama, tetapi konfigurasi MPLS / LDP di v7 lebih ketat, periksa `mpls ldp` dan `mpls interface` |
| BCP | melalui PPP Profile `bridge` + L2TP `use-bcp` | Sama |
| MLPPP / MRRU | Didukung, `mrru` di profile dan client | Sama, pastikan MTU/MRRU konsisten |
| Bridge | `/interface bridge`, `bridge port` | Sama, v7 memiliki bridge VLAN filtering yang lebih baik |
| Performa | Memadai untuk CCTV berskala kecil | Lebih baik untuk throughput besar |

### Konfigurasi A - L2TP + EoIP (Paling Umum, Tanpa MPLS)

#### Pada R-CORE

```routeros
# 1. L2TP Tunnel sudah dibuat di Modul 2, pastikan aktif
# 2. EoIP Tunnel di atas L2TP (atau langsung melalui IP)
/interface eoip add name=eoip-cabang \
  remote-address=10.0.1.10 tunnel-id=10 \
  keepalive=10s,10 \
  disabled=no comment="EoIP ke Cabang"

# 3. Bridge untuk CCTV
/interface bridge add name=bridge-cctv protocol-mode=none comment="Bridge CCTV"
/interface bridge port add bridge=bridge-cctv interface=eoip-cabang
/interface bridge port add bridge=bridge-cctv interface=ether5 comment="Port CCTV Lokal"

/ip address add address=192.168.20.1/24 interface=bridge-cctv network=192.168.20.0
```

#### Pada R-BRANCH

```routeros
/interface eoip add name=eoip-core \
  remote-address=10.0.1.1 tunnel-id=10 \
  keepalive=10s,10 disabled=no

/interface bridge add name=bridge-cctv protocol-mode=none
/interface bridge port add bridge=bridge-cctv interface=eoip-core
/interface bridge port add bridge=bridge-cctv interface=ether2

/ip address add address=192.168.20.2/24 interface=bridge-cctv network=192.168.20.0
```

### Konfigurasi B - L2TP + VPLS (Untuk Banyak Site, Diperlukan MPLS)

```routeros
# Di kedua router, aktifkan MPLS dan LDP terlebih dahulu
# R-CORE:
/mpls interface add interface=ether1 mpls-mtu=1508
/mpls ldp set enabled=yes lsr-id=10.0.0.1 transport-address=10.0.0.1
/mpls ldp interface add interface=ether1

# R-BRANCH:
/mpls interface add interface=ether1 mpls-mtu=1508
/mpls ldp set enabled=yes lsr-id=10.0.0.2 transport-address=10.0.0.2
/mpls ldp interface add interface=ether1

# VPLS pada R-CORE:
/interface vpls add name=vpls-cabang \
  remote-peer=10.0.0.2 vpls-id=100:10 \
  cisco-style=yes cisco-style-id=100 \
  disabled=no comment="VPLS ke Cabang"

/interface bridge add name=bridge-vpls
/interface bridge port add bridge=bridge-vpls interface=vpls-cabang
/interface bridge port add bridge=bridge-vpls interface=ether5

# VPLS pada R-BRANCH: mirror, ubah remote-peer ke 10.0.0.1
```

### Konfigurasi C - L2TP + BCP (Bridge PPP Langsung)

```routeros
# Pada R-CORE: Buatlah bridge dan arahkan PPP Profile ke bridge tersebut
/interface bridge add name=bridge-bcp
/interface bridge port add bridge=bridge-bcp interface=ether5

/ppp profile add name=profile-bcp \
  local-address=192.168.20.1 remote-address=192.168.20.10 \
  bridge=bridge-bcp bridge-learning=yes \
  change-tcp-mss=yes

# Aktifkan BCP di L2TP Server
/interface l2tp-server server set use-bcp=yes default-profile=profile-bcp

# Pada R-BRANCH: L2TP Client dengan BCP
/interface l2tp-client add name=l2tp-bcp \
  connect-to=203.0.113.2 user=cabang-l2tp password=PasswordStrongL2TP \
  profile=profile-bcp use-ipsec=yes ipsec-secret=IpsecSecretStrong123 \
  add-default-route=no disabled=no
```

### Konfigurasi D - Multilink / MLPPP (Gabung Dua Link)

```routeros
# Di PPP Profile, aktifkan MRRU
/ppp profile set profile-pppoe-gold mrru=1500

# Di Client PPPoE dengan dua interface fisik
/interface pppoe-client add name=pppoe-mlppp \
  interface=ether1,ether2 \
  user=pelanggan101 password=PasswordCustomer101 \
  profile=profile-pppoe-gold \
  mrru=1500 max-mtu=1500 max-mru=1500 \
  add-default-route=yes disabled=no

# Nonaktifkan mangle MSS jika menggunakan MLPPP, agar tidak konflik
# Periksa dan nonaktifkan rule MSS lama jika perlu:
/ip firewall mangle print where comment~"MSS"
```

### Verifikasi

```routeros
/interface eoip print
/interface vpls print
/interface bridge print
/interface bridge port print
/interface bridge host print where bridge=bridge-cctv
# Ping antar site di subnet yang sama
/ping 192.168.20.2 src-address=192.168.20.1
# Periksa MAC yang melalui bridge
/interface bridge host print
```

### Operasi dan Troubleshooting

- **EoIP tersambung tetapi tidak dapat ping:** Periksa `tunnel-id` harus sama pada kedua sisi, dan `remote-address` harus IP yang dapat dijangkau. Periksa firewall: EoIP gunakan protokol GRE (IP 47). Izinkan melalui firewall jika diblokir.
- **Loop / broadcast storm:** Jangan masukkan dua interface yang sama ke bridge tanpa `protocol-mode` atau loop protection. Untuk EoIP, set `protocol-mode=none` di bridge CCTV jika tidak terdapat loop, atau `rstp` jika terdapat redundansi.
- **VPLS tidak up:** Periksa LDP neighbor: `/mpls ldp neighbor print`. Periksa `vpls-id` dan `remote-peer` harus mirror. Periksa MTU MPLS minimal 1508.
- **BCP tidak bridge:** Periksa profile PPP `bridge` sudah diisi dan L2TP server `use-bcp=yes`. BCP hanya jalan jika kedua sisi setuju.
- **MLPPP tidak menggabung:** Periksa `mrru` sama pada server dan client, dan MTU konsisten. Apabila satu link putus, MLPPP harus tetap berjalan dengan kecepatan satu link.

## Modul 5: IPsec - Tunnel Terenkripsi Antar Router

### Tujuan Implementasi

Membangun VPN site-to-site yang terenkripsi dan aman, termasuk saat kedua sisi berada di balik NAT.

### Studi Kasus

Kantor pusat (192.168.88.0/24) dan cabang (192.168.10.0/24) harus saling akses server internal. Koneksi melewati internet publik. Terdapat NAT pada kedua sisi. Diperlukan IPsec dengan IKEv2, AES-256, SHA-256, PFS, dan DPD agar tunnel otomatis pulih jika putus.

### Konsep Kunci

- **Peer:** Siapa lawan bicara IPsec, bagaimana autentikasi, dan parameter IKE.
- **Identity:** Metode autentikasi (pre-shared-key, certificate) dan secret.
- **Proposal:** Algoritma enkripsi dan autentikasi untuk fase 2 (IPsec SA).
- **Policy:** Traffic apa yang harus dienkripsi (src-address ke dst-address), dan action tunnel atau drop.
- **Installed-SA:** Security Association yang sedang aktif. Dapat di-flush jika terhenti.
- **NAT-Traversal:** Agar IPsec dapat melalui NAT. Menggunakan UDP 4500.
- **DPD:** Dead Peer Detection. Mendeteksi peer mati dan membangun ulang tunnel.

### Perbandingan RouterOS v6 vs v7

| Aspek | RouterOS v6 (6.49.x) | RouterOS v7 (7.x) |
| --- | --- | --- |
| Struktur Peer | `/ip ipsec peer` berisi banyak parameter langsung | Dipisah: `/ip ipsec profile`, `/ip ipsec peer`, `/ip ipsec identity` |
| Proposal | `/ip ipsec proposal` | Sama, tetapi parameter `pfs-group` lebih eksplisit |
| Policy | `/ip ipsec policy` | Sama |
| NAT-Traversal | Di peer, `nat-traversal=yes` | Di profile, `nat-traversal=yes` |
| DPD | Di peer | Di profile |
| Certificate | Didukung, manajemen melalui `/certificate` | Sama, tetapi v7 memiliki dukungan certificate dan EAP yang lebih baik |
| IKEv2 | Didukung | Didukung, direkomendasikan di v7 |
| Rekomendasi | Gunakan IKEv2 jika memungkinkan | Wajib gunakan IKEv2 untuk instalasi baru |

### Konfigurasi - RouterOS v7 (Rekomendasi)

#### Pada R-CORE (v7)

```routeros
# 1. Profile IPsec
/ip ipsec profile add name=profile-ike2 \
  hash-algorithm=sha256 enc-algorithm=aes-256 \
  dh-group=modp2048 lifetime=8h \
  proposal-check=obey nat-traversal=yes \
  dpd-interval=2m dpd-maximum-failures=5

# 2. Peer
/ip ipsec peer add name=peer-branch \
  address=203.0.113.10/32 \
  profile=profile-ike2 \
  exchange-mode=ike2 send-initial-contact=yes

# 3. Identity (Pre-Shared-Key)
/ip ipsec identity add peer=peer-branch \
  auth-method=pre-shared-key secret=IpsecSecretStrong123 \
  generate-policy=port-strict

# 4. Proposal
/ip ipsec proposal add name=proposal-ike2 \
  auth-algorithms=sha256 enc-algorithms=aes-256-cbc \
  lifetime=8h pfs-group=modp2048

# 5. Policy
/ip ipsec policy add src-address=192.168.88.0/24 dst-address=192.168.10.0/24 \
  peer=peer-branch tunnel=yes sa-src-address=203.0.113.2 \
  sa-dst-address=203.0.113.10 proposal=proposal-ike2 \
  action=encrypt level=require

# 6. NAT Bypass - Traffic IPsec jangan di-NAT
/ip firewall nat add chain=srcnat \
  src-address=192.168.88.0/24 dst-address=192.168.10.0/24 \
  action=accept place-before=0 comment="Bypass NAT untuk IPsec"

# 7. Firewall - Izinkan IPsec
/ip firewall filter add chain=input protocol=udp port=500,4500 action=accept comment="IPsec IKE + NAT-T" place-before=0
/ip firewall filter add chain=input protocol=ipsec-esp action=accept comment="IPsec ESP" place-before=0
```

#### Pada R-BRANCH (v7) - Mirror

```routeros
/ip ipsec profile add name=profile-ike2 \
  hash-algorithm=sha256 enc-algorithm=aes-256 \
  dh-group=modp2048 lifetime=8h \
  proposal-check=obey nat-traversal=yes \
  dpd-interval=2m dpd-maximum-failures=5

/ip ipsec peer add name=peer-core \
  address=203.0.113.2/32 profile=profile-ike2 \
  exchange-mode=ike2 send-initial-contact=yes

/ip ipsec identity add peer=peer-core \
  auth-method=pre-shared-key secret=IpsecSecretStrong123 \
  generate-policy=port-strict

/ip ipsec proposal add name=proposal-ike2 \
  auth-algorithms=sha256 enc-algorithms=aes-256-cbc \
  lifetime=8h pfs-group=modp2048

/ip ipsec policy add src-address=192.168.10.0/24 dst-address=192.168.88.0/24 \
  peer=peer-core tunnel=yes sa-src-address=203.0.113.10 \
  sa-dst-address=203.0.113.2 proposal=proposal-ike2 \
  action=encrypt level=require

/ip firewall nat add chain=srcnat \
  src-address=192.168.10.0/24 dst-address=192.168.88.0/24 \
  action=accept place-before=0 comment="Bypass NAT untuk IPsec"

/ip firewall filter add chain=input protocol=udp port=500,4500 action=accept comment="IPsec IKE + NAT-T" place-before=0
/ip firewall filter add chain=input protocol=ipsec-esp action=accept comment="IPsec ESP" place-before=0
```

### Konfigurasi - RouterOS v6 (Untuk Router Lama)

```routeros
# v6: Peer langsung berisi banyak parameter, tidak terdapat profile terpisah
/ip ipsec peer add address=203.0.113.10/32 \
  auth-method=pre-shared-key secret=IpsecSecretStrong123 \
  exchange-mode=ike2 generate-policy=port-strict \
  send-initial-contact=yes nat-traversal=yes \
  dpd-interval=2m dpd-maximum-failures=5 \
  hash-algorithm=sha256 enc-algorithm=aes-256 dh-group=modp2048 lifetime=8h

/ip ipsec proposal add name=proposal-ike2 \
  auth-algorithms=sha256 enc-algorithms=aes-256-cbc \
  lifetime=8h pfs-group=modp2048

/ip ipsec policy add src-address=192.168.88.0/24 dst-address=192.168.10.0/24 \
  peer=203.0.113.10/32 tunnel=yes \
  sa-src-address=203.0.113.2 sa-dst-address=203.0.113.10 \
  proposal=proposal-ike2 action=encrypt level=require
```

### Verifikasi

```routeros
/ip ipsec peer print
/ip ipsec identity print
/ip ipsec policy print
/ip ipsec installed-sa print
/ip ipsec active-peers print

# Ping harus melalui tunnel
/ping 192.168.10.1 src-address=192.168.88.1
# Periksa apakah SA terbentuk dan terdapat bytes terenkripsi
/ip ipsec installed-sa print where src-address=203.0.113.2
```

### Operasi dan Troubleshooting

- **Peer tidak terbentuk, log "no proposal chosen":** Proposal tidak cocok. Samakan `enc-algorithm`, `hash-algorithm`, dan `dh-group` pada kedua sisi.
- **Peer terbentuk tetapi policy tidak aktif / SA 0:** Periksa policy `src-address` dan `dst-address` harus mirror. Periksa NAT bypass harus di atas rule masquerade.
- **IPsec jalan tetapi lambat:** Periksa PFS dan lifetime. PFS modp2048 lebih berat tetapi lebih aman. Apabila router kecil, pertimbangkan modp1024 atau AES-128 untuk site kecil.
- **Tunnel putus setelah idle:** Periksa DPD. Pastikan firewall tidak memblokir UDP 4500 setelah idle. Gunakan `dpd-interval=2m`.
- **SA terhenti setelah ubah konfigurasi:** Flush SA: `/ip ipsec installed-sa flush` lalu ping lagi untuk memicu pembentukan SA baru.
- **Operasi:** Pantaulah `/ip ipsec active-peers print`, `/ip ipsec installed-sa print`, dan log IPsec. Untuk failover, buatlah dua peer ke dua IP publik cabang apabila tersedia.

## Modul 6: Hotspot - Manajemen User Akses WiFi dan LAN

### Tujuan Implementasi

Membangun portal login untuk user WiFi / LAN, dengan voucher, limit kecepatan, kuota, dan pemantauan user aktif. Cocok untuk cafe, sekolah, hotel, area publik, dan kantor.

### Studi Kasus

Sebuah sekolah dan area publik diperlukan WiFi untuk 300 user bersamaan. Terdapat tiga tipe user:

- **Voucher harian** untuk tamu, kecepatan 5 Mbps, masa aktif 2 jam.
- **Siswa** login dengan username, kecepatan 10 Mbps, kuota 2 GB per hari.
- **Perangkat bypass** seperti printer dan CCTV, tanpa melakukan login, melalui IP Binding.

Terdapat walled garden untuk domain sekolah agar dapat diakses tanpa melakukan login. Login harus melalui HTTPS dan Cookie agar user tidak login berulang.

### Konsep Kunci

- **Hotspot Profile:** Mengatur hotspot-address, dns-name, login-by, html-directory, dan use-radius.
- **Hotspot User Profile:** Template untuk rate-limit, shared-users, session-timeout, keepalive, dan packet-mark.
- **Hotspot User:** Akun voucher atau user tetap.
- **Host Table vs Active Table:** Host adalah perangkat yang terdeteksi, Active adalah yang sudah login.
- **Walled Garden:** Domain / IP yang dapat diakses tanpa melakukan login.
- **IP Binding:** Mem-bypass perangkat tertentu dari Hotspot.

### Perbandingan RouterOS v6 vs v7

| Aspek | RouterOS v6 (6.49.x) | RouterOS v7 (7.x) |
| --- | --- | --- |
| Hotspot Setup | `/ip hotspot setup` dan `/ip hotspot profile` | Sama |
| Login-by | `http-chap`, `http-pap`, `cookie`, `mac`, `trial`, `https` | Sama |
| HTML Directory | `hotspot` default, dapat custom | Sama, custom page tetap di `/flash/hotspot` atau `/hotspot` |
| User Profile | `/ip hotspot user profile` | Sama |
| Walled Garden | `/ip hotspot walled-garden` dan `walled-garden ip` | Sama |
| IP Binding | `/ip hotspot ip-binding` | Sama |
| RADIUS | `use-radius=yes` di profile | Sama, tetapi User Manager v5 di v7 lebih stabil |
| HTTPS Login | Diperlukan certificate | Sama, v7 lebih mudah generate certificate melalui Let's Encrypt / self-signed |
| Catatan | Banyak digunakan di hotspot lama | Direkomendasikan untuk instalasi baru |

### Konfigurasi

#### Langkah 1 - IP Pool dan Interface

```routeros
/ip pool add name=hs-pool ranges=192.168.88.10-192.168.88.200
/ip address add address=192.168.88.1/24 interface=ether3 network=192.168.88.0 comment="Hotspot Gateway"
```

#### Langkah 2 - Hotspot Profile

```routeros
/ip hotspot profile add name=hs-profile \
  hotspot-address=192.168.88.1 \
  dns-name=login.sekolah.local \
  html-directory=hotspot \
  login-by=cookie,http-chap,https,trial \
  trial-user-profile=profile-trial \
  use-radius=yes \
  trial-uptime-limit=30m trial-uptime-reset=1d \
  comment="Hotspot Sekolah"
```

#### Langkah 3 - Hotspot Server

```routeros
/ip hotspot add name=hotspot1 \
  interface=ether3 profile=hs-profile \
  address-pool=hs-pool \
  idle-timeout=5m keepalive-timeout=5m \
  addresses-per-mac=1 disabled=no
```

#### Langkah 4 - Hotspot User Profile

```routeros
# Voucher tamu
/ip hotspot user profile add name=profile-voucher \
  shared-users=1 rate-limit=5M/5M \
  session-timeout=2h keepalive-timeout=5m \
  address-list="" \
  comment="Voucher Tamu 5M 2 Jam"

# Siswa
/ip hotspot user profile add name=profile-siswa \
  shared-users=2 rate-limit=10M/10M \
  session-timeout=8h keepalive-timeout=5m \
  incoming-packet-mark="" outgoing-packet-mark="" \
  comment="Siswa 10M"

# Trial
/ip hotspot user profile add name=profile-trial \
  shared-users=1 rate-limit=2M/2M \
  session-timeout=30m keepalive-timeout=5m \
  comment="Trial 30 Menit"
```

#### Langkah 5 - Hotspot User

```routeros
# Voucher tamu (buatlah banyak voucher dengan script atau bulk)
/ip hotspot user add name=voucher-001 password=voucher001 \
  profile=profile-voucher limit-uptime=2h \
  comment="Voucher Tamu - 12 Nov"

# Siswa
/ip hotspot user add name=siswa-001 password=SiswaKuat001 \
  profile=profile-siswa limit-uptime=8h \
  limit-bytes-in=1000000000 limit-bytes-out=1000000000 \
  comment="Siswa - Kelas 10A"

# Reset limit harian melalui scheduler (jalankan tengah malam)
/system scheduler add name=reset-hotspot-limit \
  start-time=00:00:00 interval=1d \
  on-event="/ip hotspot user reset-counters" \
  comment="Reset counter Hotspot harian"
```

#### Langkah 6 - Walled Garden dan Bypass

```routeros
# Walled Garden - domain sekolah dapat diakses tanpa melakukan login
/ip hotspot walled-garden add dst-host=*.sekolah.local action=allow comment="Domain Sekolah"
/ip hotspot walled-garden add dst-host=*.kemdikbud.go.id action=allow comment="Domain Edukasi"
/ip hotspot walled-garden ip add dst-address=8.8.8.8 action=allow comment="DNS Publik untuk Walled Garden"

# IP Binding - Bypass printer, CCTV, AP
/ip hotspot ip-binding add mac-address=AA:BB:CC:DD:EE:01 \
  address=192.168.88.50 type=bypassed comment="Printer Perpustakaan"
/ip hotspot ip-binding add mac-address=AA:BB:CC:DD:EE:02 \
  address=192.168.88.51 type=bypassed comment="CCTV Lobby"
```

#### Langkah 7 - Kustomisasi Halaman Login (Opsional)

```routeros
# Upload file HTML custom ke /hotspot melalui FTP / WinBox Files
# Edit login.html, status.html, dan logout.html
# Tambahkan logo, nama sekolah, dan cara beli voucher

# Contoh: Ubah halaman login ke folder custom
/ip hotspot profile set hs-profile html-directory=hotspot-sekolah
```

### Verifikasi

```routeros
/ip hotspot print
/ip hotspot profile print
/ip hotspot user print
/ip hotspot user profile print
/ip hotspot host print
/ip hotspot active print
/ip hotspot walled-garden print
/ip hotspot ip-binding print
```

### Operasi dan Troubleshooting

- **User tidak diarahkan ke halaman login:** Periksa DNS. Hotspot memerlukan DNS yang dapat resolve `dns-name`. Periksa firewall: Hotspot memerlukan port 80 dan 443 terbuka ke hotspot-address. Periksa `address-pool` tidak habis.
- **Login berhasil tetapi tidak terdapat internet:** Periksa NAT, firewall forward, dan `hotspot-address`. Periksa juga `use-radius` - jika RADIUS down dan `use-radius=yes`, login lokal mungkin tetap berjalan tergantung setting, tetapi lebih aman periksa RADIUS.
- **User sering logout sendiri:** Periksa `keepalive-timeout` dan `idle-timeout`. Apabila AP sering roaming, naikkan `keepalive-timeout` ke 10m.
- **Voucher habis tetapi masih dapat melakukan login:** Periksa `limit-uptime` dan `limit-bytes`. Jalankan `/ip hotspot user print` untuk melihat `uptime` dan `bytes`. Gunakan scheduler reset counter.
- **Walled Garden tidak berjalan:** Periksa urutan walled garden dan DNS. Walled garden berbasis dst-host memerlukan resolusi DNS. Untuk IP, gunakan `walled-garden ip`.
- **Operasi harian:** Pantaulah `/ip hotspot active print count-only`, `/ip hotspot host print count-only`, dan jumlah voucher terpakai. Buatlah laporan mingguan: user aktif pada jam sibuk, voucher terlaris, dan perangkat bypass.

## Modul 7: RADIUS - Manajemen User Terpusat

### Tujuan Implementasi

Memindahkan manajemen user dari router lokal (PPP Secret, Hotspot User) ke server RADIUS terpusat. Satu server mengelola banyak NAS (router PPPoE, Hotspot, L2TP). Ini fondasi untuk ISP dan jaringan kampus yang besar.

### Studi Kasus

ISP memiliki 3 NAS: R-CORE (PPPoE), R-HOTSPOT (Hotspot), dan R-BRANCH (L2TP). Seluruh user harus dikelola dari satu User Manager di 10.99.99.10. User PPPoE dan Hotspot dapat gunakan akun yang sama. Admin dapat terputus user dari server tanpa melakukan login ke tiap router. Terdapat profile Gold dan Silver yang berlaku di semua NAS.

### Konsep Kunci

- **RADIUS Client (NAS):** Router yang meminta autentikasi ke server RADIUS.
- **RADIUS Server:** Server yang memverifikasi user, memberi profile, dan mencatat accounting.
- **User Manager:** Aplikasi RADIUS bawaan MikroTik. Di v6 namanya User Manager v4, di v7 namanya User Manager v5.
- **RADIUS Incoming / CoA:** Port 3799 untuk terputus atau change-of-authorization dari server ke NAS.
- **AAA:** Authentication, Authorization, Accounting. Authentication periksa password, Authorization periksa profile / rate-limit, Accounting catat durasi dan bytes.

### Perbandingan RouterOS v6 vs v7

| Aspek | RouterOS v6 (6.49.x) | RouterOS v7 (7.x) |
| --- | --- | --- |
| User Manager Package | Package terpisah `user-manager`, diinstall di CHR / RB | Package terpisah `user-manager`, versi 5 |
| Sintaks User Manager | `/tool user-manager ...` | `/user-manager ...` (tanpa `/tool`) |
| Database | SQLite, di `/user-manager` | Sama, tetapi lebih stabil dan terdapat perbaikan performa |
| RADIUS Client | `/radius add ...` | Sama |
| RADIUS Incoming | `/radius incoming set accept=yes port=3799` | Sama |
| PPP AAA | `/ppp aaa set use-radius=yes accounting=yes` | Sama |
| Hotspot RADIUS | `use-radius=yes` di Hotspot Profile | Sama |
| Web UI User Manager | `http://10.99.99.10/userman` | `http://10.99.99.10/um` atau `/userman` tergantung versi, periksa di dokumentasi v7 |
| Migrasi | Backup User Manager v4 harus diexport | Migrasi v4 ke v5 diperlukan export / import manual, jangan langsung upgrade tanpa backup |
| Rekomendasi | Untuk router lama yang masih v6 | Untuk instalasi baru, gunakan v7 + User Manager v5 |

### Konfigurasi - User Manager Server (RouterOS v7)

#### Langkah 1 - Install dan Aktifkan User Manager

```routeros
# Di server User Manager (CHR atau RB yang jadi server):
# Pastikan package user-manager sudah terinstall dan enabled
/system package print where name~"user-manager"

# Pembuatan database dan user admin pertama (melalui CLI v7)
/user-manager user add username=admin-um password=AdminUmKuat123 \
  comment="Admin User Manager"

# Pembuatan Router (NAS) yang diizinkan untuk bertanya ke server
/user-manager router add name=R-CORE \
  ip-address=10.99.99.1 shared-secret=RADIUSSecretSangatKuat123 \
  comment="NAS PPPoE Core"

/user-manager router add name=R-HOTSPOT \
  ip-address=10.99.99.2 shared-secret=RADIUSSecretSangatKuat123 \
  comment="NAS Hotspot"

/user-manager router add name=R-BRANCH \
  ip-address=10.99.99.3 shared-secret=RADIUSSecretSangatKuat123 \
  comment="NAS L2TP Branch"
```

#### Langkah 2 - Profile dan Limitation pada User Manager

```routeros
# Di v7, profile dan limitation dibuat melalui CLI / Web UI
# Contoh melalui CLI v7:
/user-manager profile add name=profile-gold-um \
  comment="Gold 20M - User Manager"

/user-manager limitation add name=limit-gold \
  profile=profile-gold-um \
  rate-limit=20M/20M \
  uptime-limit=1d \
  comment="Limit Gold"

# User
/user-manager user add username=pelanggan101 password=PasswordCustomer101 \
  profile=profile-gold-um \
  comment="PPPoE + Hotspot - Gold"
```

> **Catatan v6:** Di RouterOS v6, sintaksnya adalah `/tool user-manager ...`. Contoh: `/tool user-manager router add ...`, `/tool user-manager user add ...`, `/tool user-manager user profile add ...`. Konsepnya sama, hanya path CLI yang berbeda.

### Konfigurasi - RADIUS Client pada NAS (R-CORE, R-HOTSPOT, R-BRANCH)

```routeros
# Di setiap NAS, tambahkan RADIUS Client
/radius add service=ppp,hotspot \
  address=10.99.99.10 secret=RADIUSSecretSangatKuat123 \
  timeout=3s authentication-port=1812 accounting-port=1813 \
  comment="RADIUS User Manager"

# Aktifkan RADIUS Incoming agar server dapat terputus user
/radius incoming set accept=yes port=3799

# PPP AAA - Gunakan RADIUS untuk PPP
/ppp aaa set use-radius=yes accounting=yes interim-update=5m \
  default-profile=profile-default

# Hotspot Profile - Gunakan RADIUS untuk Hotspot
/ip hotspot profile set hs-profile use-radius=yes

# Periksa RADIUS
/radius print
/radius incoming print
/ppp aaa print
```

### Konfigurasi - RADIUS untuk Service Tertentu Saja

```routeros
# Jika hanya PPPoE yang gunakan RADIUS, Hotspot tetap lokal:
/radius add service=ppp address=10.99.99.10 secret=RADIUSSecretSangatKuat123

# Jika hanya Hotspot yang gunakan RADIUS:
/radius add service=hotspot address=10.99.99.10 secret=RADIUSSecretSangatKuat123

# Jika hanya login router yang gunakan RADIUS (untuk admin):
/radius add service=login address=10.99.99.10 secret=RADIUSSecretSangatKuat123
/user aaa set use-radius=yes accounting=yes
```

### Verifikasi

```routeros
# Di NAS:
/radius print
/radius monitor 0
# Periksa request berhasil / timeout
/radius incoming print

# Di Server User Manager (v7):
/user-manager router print
/user-manager user print
/user-manager session print
# Lihat sesi aktif, uptime, dan bytes

# Test login PPPoE / Hotspot dengan user dari User Manager
/ppp active print where name=pelanggan101
/ip hotspot active print where user=pelanggan101

# Terputus paksa dari server (CoA / RADIUS Incoming)
# melalui User Manager Web UI: pilih session -> Terputus
# melalui CLI pada NAS jika perlu:
/ppp active remove [find name=pelanggan101]
```

### Operasi dan Troubleshooting

- **Login gagal, log "RADIUS timeout":** Periksa konektivitas NAS ke server RADIUS: `/ping 10.99.99.10`. Periksa firewall UDP 1812, 1813, dan 3799. Periksa `secret` sama persis pada NAS dan pada User Manager router.
- **Login berhasil tetapi profile salah:** Periksa profile pada User Manager. Apabila NAS memiliki profile lokal dengan nama sama, RADIUS dapat tertimpa. Pastikan `default-profile` di `/ppp aaa` tidak mengalahkan profile RADIUS.
- **Accounting tidak tercatat:** Periksa `accounting=yes` di `/ppp aaa` dan `interim-update`. Periksa port accounting 1813 terbuka. Periksa database User Manager tidak penuh.
- **RADIUS Incoming tidak berjalan:** Periksa `/radius incoming print` - harus `accept=yes`. Periksa firewall UDP 3799 pada NAS. Periksa User Manager dapat reach NAS.
- **User Manager berjalan lambat setelah ribuan user:** Di v7, pisahkan User Manager ke CHR khusus, jangan gabung dengan PPPoE Server yang sibuk. Backup database rutin dan bersihkan session lama.
- **Operasi harian:** Pantaulah `/user-manager session print count-only`, `/radius monitor`, dan log RADIUS. Buatlah backup User Manager setiap hari: `/user-manager database save` atau export melalui Web UI.
- **Keamanan:** Jangan membuka port RADIUS 1812/1813/3799 ke internet. Hanya dari IP NAS yang dikenal. Gunakan secret yang panjang dan ubah berkala.
  

## Lampiran A: Perintah Verifikasi Cepat

### Semua Modul

```routeros
# PPP
/ppp active print
/ppp secret print

# PPPoE
/interface pppoe-server print
/ppp active print where service=pppoe

# Tunnel
/interface pptp-client print
/interface l2tp-client print
/interface eoip print
/interface vpls print

# IPsec
/ip ipsec active-peers print
/ip ipsec installed-sa print

# Hotspot
/ip hotspot active print
/ip hotspot host print

# RADIUS
/radius print
/radius monitor 0
/user-manager session print
```

### Backup Sebelum Perubahan

```routeros
# Backup biner + export konfigurasi
/system backup save name=backup-sebelum-perubahan
/export file=export-sebelum-perubahan

# Backup User Manager (v7)
/user-manager database save name=um-backup

# Backup User Manager (v6)
/tool user-manager database save name=um-backup-v6
```

## Lampiran B: Troubleshooting Umum Antar Modul

| Gejala | Kemungkinan Penyebab | Periksa |
| --- | --- | --- |
| PPP / PPPoE tidak dapat melakukan login | Secret salah, service salah, profile salah | `/ppp secret print`, `/log print where topics~"ppp"` |
| Tunnel L2TP tersambung tetapi tidak terdapat traffic | Route belum ditambahkan | `/ip route print`, `/ppp active print` |
| PPPoE dapat dial tetapi tidak terdapat internet | NAT / firewall / DNS | `/ip firewall nat print`, `/ip dns print` |
| Bridging EoIP tidak dapat ping | Tunnel-ID beda, GRE diblokir | `/interface eoip print`, firewall GRE |
| IPsec peer tidak terbentuk | Proposal tidak cocok, NAT-T diblokir | `/ip ipsec peer print`, `/log print where topics~"ipsec"` |
| Hotspot tidak redirect | DNS / pool habis / firewall | `/ip hotspot print`, `/ip pool print` |
| RADIUS timeout | Secret beda, port diblokir | `/radius print`, `/ping 10.99.99.10` |
| User Manager berjalan lambat | Database penuh, server gabung dengan PPPoE | Pisahkan server, bersihkan session lama |

