# Dokumentasi Arsitektur Spine-Leaf untuk Datacenter Modern

## Daftar Isi
1. Pendahuluan
2. Mengapa Beralih dari Arsitektur Tradisional
3. Konsep Dasar Spine-Leaf
4. Komponen Utama
5. Topologi dan Desain
6. Underlay Network
7. Overlay Network (VXLAN & EVPN)
8. Perhitungan Oversubscription dan Scalability
9. Implementasi Konfigurasi
10. High Availability dan Redundansi
11. Monitoring dan Troubleshooting
12. Best Practices
13. Kesimpulan
14. Referensi

---

## 1. Pendahuluan

Arsitektur **Spine-Leaf** adalah desain jaringan datacenter dua-tingkat (two-tier) yang dikembangkan untuk mengatasi keterbatasan arsitektur tradisional tiga-tingkat (core-aggregation-access) dalam menghadapi pola trafik modern yang didominasi oleh komunikasi **east-west** (server-ke-server) dibanding **north-south** (client-ke-server).

Arsitektur ini pertama kali dipopulerkan oleh Charles Clos pada era telekomunikasi (dikenal sebagai **Clos Network**), kemudian diadaptasi untuk jaringan datacenter modern karena kemampuannya memberikan bandwidth yang konsisten, latency yang predictable, dan skalabilitas horizontal yang mudah.

## 2. Mengapa Beralih dari Arsitektur Tradisional

### Arsitektur 3-Tier (Core - Aggregation - Access)

```
        [Core]
       /      \
 [Aggregation] [Aggregation]
   /    \         /    \
[Access][Access][Access][Access]
```

**Kelemahan:**
- Traffic antar server pada rack berbeda harus melewati banyak hop (naik ke aggregation/core lalu turun kembali) → latency tinggi.
- Menggunakan Spanning Tree Protocol (STP) yang memblokir link redundan → bandwidth terbuang percuma (link idle).
- Sulit untuk scale-out karena bottleneck di layer core/aggregation.
- Tidak ideal untuk trafik east-west yang mendominasi aplikasi modern (microservices, big data, virtualisasi, storage terdistribusi).

### Arsitektur Spine-Leaf

**Keunggulan:**
- Setiap Leaf terhubung ke **semua** Spine → jumlah hop antar leaf selalu konsisten (predictable latency).
- Menggunakan **ECMP (Equal-Cost Multi-Path)** sehingga semua link aktif digunakan secara bersamaan (tidak ada blocking seperti STP).
- Scale-out mudah: tambah kapasitas dengan menambah switch Spine (bandwidth) atau switch Leaf (port density).
- Cocok untuk trafik east-west yang tinggi.

## 3. Konsep Dasar Spine-Leaf

Prinsip utama: **setiap Leaf switch wajib terhubung ke setiap Spine switch**, tetapi **Spine tidak boleh terhubung ke Spine lain**, dan **Leaf tidak boleh terhubung ke Leaf lain** (non-blocking full-mesh antara dua layer).

```
        [Spine1]   [Spine2]   [Spine3]   [Spine4]
         |  |  |  |   |  |  |  |   |  |  |  |   |  |  |  |
       [Leaf1]   [Leaf2]   [Leaf3]   [Leaf4]
        |||       |||       |||       |||
     Servers    Servers   Servers   Servers
```

Karakteristik penting:
- **Non-blocking**: setiap path antar leaf memiliki cost yang sama.
- **Predictable latency**: maksimal 3 hop (Leaf-Spine-Leaf) dalam topologi 2-tier.
- **Horizontal scaling**: tambah Spine untuk bandwidth, tambah Leaf untuk port density server.

## 4. Komponen Utama

| Komponen | Fungsi |
|---|---|
| **Spine Switch** | Backbone jaringan, hanya menghubungkan Leaf switch, tidak terhubung langsung ke server. Tugasnya murni forwarding trafik antar Leaf dengan performa tinggi. |
| **Leaf Switch** | Titik koneksi untuk server, storage, dan perangkat endpoint lainnya (Top-of-Rack/ToR). Setiap Leaf wajib uplink ke semua Spine. |
| **Border Leaf** | Leaf khusus yang menghubungkan fabric datacenter ke jaringan eksternal (WAN, internet, DC lain). |
| **Super-Spine** | Layer tambahan (opsional) untuk menghubungkan beberapa Spine-Leaf pod dalam datacenter besar (3-tier Clos / 5-stage Clos). |

## 5. Topologi dan Desain

### 5.1 Desain 2-Tier (Standar)
Cocok untuk datacenter skala kecil-menengah (< 1 pod, ratusan hingga ribuan server).

- Jumlah Spine umumnya genap: 2, 4, 8 unit (tergantung kebutuhan bandwidth & redundansi).
- Jumlah Leaf tergantung jumlah rack dan port density Spine.

**Rumus kapasitas maksimum Leaf** yang bisa didukung oleh N buah Spine dengan P port per Spine:

```
Jumlah maksimum Leaf = P (port Spine) 
(karena tiap Spine mengalokasikan 1 port untuk tiap Leaf)
```

### 5.2 Desain 3-Tier (Super-Spine / 5-Stage Clos)
Digunakan pada datacenter skala besar (hyperscale) yang memiliki banyak **pod** (setiap pod adalah satu unit Spine-Leaf).

```
                [Super-Spine]
              /       |        \
        [Spine-Pod1][Spine-Pod2][Spine-Pod3]
          |    |       |    |       |    |
        [Leaf][Leaf] [Leaf][Leaf] [Leaf][Leaf]
```

Digunakan ketika jumlah port di Spine sudah tidak cukup untuk menampung seluruh Leaf dalam satu tingkat.

### 5.3 Integrasi Firewall: FortiGate (Internet Gateway) dan Palo Alto (Segmentasi Client-Server)

Fabric Spine-Leaf sendiri **tidak melakukan inspeksi keamanan** — ia murni forwarding L2/L3 berkecepatan tinggi. Karena itu, firewall harus ditempatkan secara eksplisit di titik-titik strategis dalam desain. Dalam desain ini digunakan dua firewall dengan peran berbeda:

| Firewall | Peran | Posisi dalam topologi |
|---|---|---|
| **FortiGate** | Internet Edge Gateway — NAT, VPN, IPS/IDS, filtering trafik north-south dari/ke internet | Di depan **Border Leaf**, antara ISP/Internet dan fabric datacenter |
| **Palo Alto** | Segmentasi keamanan client-ke-server (east-west/north-south internal) — App-ID, User-ID, microsegmentation | Inline antara jaringan akses (campus/BYOD/WiFi/IoT) dan **Server Leaf**, sebelum trafik client menyentuh server |

**Mengapa dipisah menjadi dua firewall dengan peran berbeda?**
- **FortiGate** dioptimalkan untuk fungsi gateway internet: NAT/PAT, site-to-site & remote-access VPN, web filtering, dan IPS terhadap ancaman dari luar. Ia menjadi satu-satunya pintu keluar/masuk fabric ke internet — semua trafik north-south wajib melewatinya.
- **Palo Alto** difokuskan sebagai firewall segmentasi internal dengan kemampuan **App-ID** dan **User-ID**, sehingga bisa membedakan aplikasi/user spesifik (bukan cuma port/IP) saat client (laptop BYOD, AP wireless, gateway IoT) mengakses server di datacenter. Ini penting karena ancaman terbesar pada jaringan modern sering datang dari **internal** (device BYOD/IoT yang sudah disusupi), bukan hanya dari internet.
- Memisahkan tanggung jawab ini juga mengurangi beban satu firewall tunggal dan membatasi *blast radius* — jika satu segmen (misal IoT) disusupi, Palo Alto mencegah pergerakan lateral ke server sebelum ancaman itu bisa keluar lewat FortiGate ke internet.

**Cara integrasi ke fabric (mode routed/Layer 3):**
- Kedua firewall dijalankan dalam **mode routed (L3)**, terhubung sebagai next-hop pada Border Leaf melalui VRF/VLAN dedicated, bukan disisipkan secara transparan di jalur data switch-to-switch.
- **FortiGate**: satu interface menghadap ISP/internet (WAN), satu interface menghadap Border Leaf (LAN side fabric). Static default route atau BGP peering opsional ke ISP.
- **Palo Alto**: satu zone/interface menghadap sisi client (Distribution switch campus — BYOD/WiFi/IoT), satu zone/interface menghadap sisi server (Leaf server/Server VRF). Semua trafik client→server di-route melalui Palo Alto sebagai default gateway L3 untuk subnet client, sehingga tidak ada jalur pintas yang melewatkan firewall.
- Untuk HA, kedua firewall sebaiknya dipasang sebagai **cluster aktif-pasif atau aktif-aktif** (lihat Bagian 10).

## 6. Underlay Network

Underlay adalah jaringan fisik/IP yang menghubungkan seluruh Spine dan Leaf. Tujuannya adalah menyediakan **IP reachability** antar seluruh perangkat fabric dengan performa ECMP penuh.

**Protokol yang umum digunakan:**
- **eBGP** (paling umum & direkomendasikan oleh RFC 7938 – "Use of BGP for Routing in Large-Scale Data Centers")
- OSPF (alternatif, lebih sederhana namun kurang scalable dibanding BGP)
- IS-IS (jarang digunakan tapi valid)

**Mengapa eBGP lebih disukai?**
- Setiap switch (Spine & Leaf) diberi **AS number** unik → troubleshooting loop lebih mudah.
- Native support untuk ECMP.
- Convergence lebih stabil pada skala besar dibanding OSPF/IS-IS.
- Policy control granular (route-map, prefix-list) untuk traffic engineering.

**Skema penomoran AS (contoh umum):**
- Setiap Leaf → AS unik (misal 65001, 65002, dst.)
- Setiap Spine → AS unik atau AS yang sama untuk semua Spine (tergantung desain)

## 7. Overlay Network (VXLAN & EVPN)

Underlay hanya menyediakan IP reachability dasar. Untuk mendukung **multi-tenancy**, **Layer 2 extension antar rack**, dan **workload mobility** (VM/container berpindah rack tanpa ganti IP), digunakan teknologi **overlay**.

### 7.1 VXLAN (Virtual Extensible LAN)
- Melakukan enkapsulasi frame Ethernet (Layer 2) ke dalam paket UDP (Layer 3) → dikenal sebagai **MAC-in-UDP**.
- Menggunakan **VNI (VXLAN Network Identifier)** 24-bit → mendukung hingga 16 juta segmen L2 (jauh melebihi VLAN yang dibatasi 4096 ID).
- Endpoint enkapsulasi disebut **VTEP (VXLAN Tunnel Endpoint)**, biasanya berada di Leaf switch.

### 7.2 EVPN (Ethernet VPN)
- Digunakan sebagai **control-plane** untuk VXLAN, menggantikan mekanisme "flood-and-learn" tradisional.
- Menggunakan **MP-BGP** untuk distribusi informasi MAC dan IP address antar VTEP → mengurangi flooding BUM (Broadcast, Unknown-unicast, Multicast) traffic.
- Mendukung fitur lanjutan: **Anycast Gateway** (default gateway yang sama di semua Leaf), **Multi-homing** (ESI/LACP dual-homing server ke 2 Leaf berbeda), dan **Integrated Routing and Bridging (IRB)**.

**Diagram konsep Underlay vs Overlay:**

```
Overlay  : VXLAN/EVPN (Virtual Network, VNI, Tenant Segmentation)
              ↑ (encapsulated di atas)
Underlay : BGP/OSPF IP Fabric (Physical Reachability, ECMP)
```

## 8. Perhitungan Oversubscription dan Scalability

**Oversubscription ratio** adalah perbandingan antara total bandwidth downlink (ke server) dengan total bandwidth uplink (ke Spine) pada satu Leaf switch.

```
Oversubscription Ratio = Total Bandwidth Downlink : Total Bandwidth Uplink
```

**Contoh perhitungan:**
- Leaf switch memiliki 48 port 25GbE (downlink ke server) dan 6 port 100GbE (uplink ke Spine).
- Total downlink = 48 x 25 Gbps = 1200 Gbps
- Total uplink = 6 x 100 Gbps = 600 Gbps
- Oversubscription ratio = 1200:600 = **2:1**

Rasio ideal untuk workload umum: **3:1** hingga **1:1** tergantung kebutuhan (semakin kecil rasio semakin non-blocking, tapi biaya makin mahal).

**Tips desain kapasitas:**
- Untuk workload yang sangat sensitif terhadap latency (HPC, storage all-flash, AI/ML training cluster) → gunakan rasio 1:1 (non-blocking).
- Untuk workload umum enterprise → rasio 3:1 sudah cukup ekonomis.

## 9. Implementasi Konfigurasi

Berikut contoh implementasi dasar underlay eBGP dan overlay VXLAN EVPN pada perangkat berbasis **Cisco NX-OS** (konsep serupa berlaku di Arista EOS, Juniper Junos, atau platform berbasis FRRouting/Cumulus Linux).

### 9.1 Contoh Konfigurasi Underlay eBGP (Leaf Switch)

```
feature bgp
feature interface-vlan
feature vn-segment-vlan-based
feature nv overlay

! Konfigurasi interface uplink ke Spine
interface Ethernet1/1
  description Uplink-to-Spine1
  no switchport
  ip address 10.0.1.1/31
  no shutdown

interface Ethernet1/2
  description Uplink-to-Spine2
  no switchport
  ip address 10.0.1.3/31
  no shutdown

! Konfigurasi BGP underlay
router bgp 65001
  router-id 1.1.1.1
  address-family ipv4 unicast
    maximum-paths 4
  neighbor 10.0.1.0 remote-as 65100
    address-family ipv4 unicast
      route-map ALLOW-ALL in
      route-map ALLOW-ALL out
  neighbor 10.0.1.2 remote-as 65100
    address-family ipv4 unicast
      route-map ALLOW-ALL in
      route-map ALLOW-ALL out
```

### 9.2 Contoh Konfigurasi Spine (BGP Route Reflector untuk Overlay)

```
router bgp 65100
  router-id 2.2.2.2
  address-family ipv4 unicast
    maximum-paths 8
  neighbor 10.0.1.1 remote-as 65001
    address-family ipv4 unicast
  neighbor 10.0.2.1 remote-as 65002
    address-family ipv4 unicast

  ! Spine bertindak sebagai Route Reflector untuk EVPN address-family
  address-family l2vpn evpn
    retain route-target all
  neighbor 10.0.1.1
    address-family l2vpn evpn
      route-reflector-client
      send-community extended
  neighbor 10.0.2.1
    address-family l2vpn evpn
      route-reflector-client
      send-community extended
```

### 9.3 Contoh Konfigurasi VXLAN EVPN (Leaf sebagai VTEP)

```
! Aktifkan NVE interface sebagai VTEP
interface nve1
  no shutdown
  source-interface loopback1
  host-reachability protocol bgp
  member vni 10001
    ingress-replication protocol bgp
  member vni 10002
    ingress-replication protocol bgp

! Mapping VLAN ke VNI
vlan 100
  vn-segment 10001
vlan 200
  vn-segment 10002

! Konfigurasi Anycast Gateway (IRB) untuk VLAN 100
interface Vlan100
  no shutdown
  ip address 172.16.10.1/24
  fabric forwarding mode anycast-gateway

! EVPN VNI advertisement di BGP
router bgp 65001
  address-family l2vpn evpn
    advertise-pip
  vlan-configuration 100
    rd auto
    route-target import auto
    route-target export auto
```

### 9.4 Verifikasi Dasar

```
! Cek status BGP underlay
show bgp ipv4 unicast summary

! Cek status EVPN neighbor
show bgp l2vpn evpn summary

! Cek status VTEP dan VNI
show nve peers
show nve vni

! Cek tabel MAC yang dipelajari via EVPN
show l2route evpn mac all
```

> **Catatan:** Sintaks di atas menggunakan gaya Cisco NX-OS sebagai referensi konsep. Vendor lain seperti Arista (EOS), Juniper (Junos/Junos Evolved), dan platform open-source seperti Cumulus Linux / SONiC memiliki sintaks berbeda namun konsep underlay-overlay-nya identik.

### 9.5 Contoh Konfigurasi FortiGate (Internet Gateway)

Konfigurasi dasar interface WAN/LAN dan policy NAT keluar (internet edge), menggunakan CLI FortiOS:

```
config system interface
    edit "wan1"
        set ip 203.0.113.2/30
        set allowaccess ping
        set role wan
    next
    edit "lan-fabric"
        set ip 10.100.0.1/24
        set allowaccess ping https ssh
        set role lan
    next
end

! Default route ke ISP
config router static
    edit 1
        set gateway 203.0.113.1
        set device "wan1"
    next
end

! Policy NAT untuk trafik keluar dari fabric ke internet
config firewall policy
    edit 1
        set name "Fabric-to-Internet"
        set srcintf "lan-fabric"
        set dstintf "wan1"
        set srcaddr "Fabric-Subnet"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
        set nat enable
        set utm-status enable
        set ips-sensor "default"
        set av-profile "default"
    next
end
```

### 9.6 Contoh Konfigurasi Palo Alto (Segmentasi Client-Server)

Konfigurasi zone dan security policy yang membatasi akses client (BYOD/WiFi/IoT) ke server berdasarkan aplikasi spesifik, menggunakan sintaks CLI PAN-OS:

```
# Definisi zone
set zone client-zone network layer3 ethernet1/1
set zone server-zone network layer3 ethernet1/2

# IP interface tiap zone
set network interface ethernet1/1 layer3 ip 10.50.0.1/24
set network interface ethernet1/2 layer3 ip 10.60.0.1/24

# Address object
set address "Client-BYOD-WiFi" ip-netmask 10.50.10.0/24
set address "Client-IoT" ip-netmask 10.50.20.0/24
set address "Server-App-Tier" ip-netmask 10.60.10.0/24

# Security policy: hanya izinkan aplikasi spesifik dari client ke server
set rulebase security rules "Client-to-Server-App" from client-zone to server-zone \
    source "Client-BYOD-WiFi" destination "Server-App-Tier" \
    application [ ssl web-browsing ms-rdp ] \
    service application-default action allow \
    log-end yes profile-setting group "default-security-profiles"

# Blokir eksplisit trafik dari segmen IoT ke server kecuali port yang dibutuhkan
set rulebase security rules "IoT-to-Server-Restricted" from client-zone to server-zone \
    source "Client-IoT" destination "Server-App-Tier" \
    application [ mqtt ] \
    service application-default action allow \
    log-end yes

# Deny-all default (eksplisit, best practice)
set rulebase security rules "Deny-All-Else" from client-zone to server-zone \
    source any destination any application any service any action deny log-end yes
```

> **Catatan:** Palo Alto ditempatkan sebagai gateway L3 untuk subnet client (10.50.0.0/16) sehingga seluruh trafik client menuju server **wajib** melewati firewall ini — tidak ada static route alternatif yang memotong jalur inspeksi.

## 10. High Availability dan Redundansi

- **Multi-homing server**: server dihubungkan ke 2 Leaf berbeda menggunakan **MLAG** (Multi-Chassis LAG) atau **EVPN ESI Multi-homing**, sehingga jika satu Leaf down, server tetap terhubung via Leaf lainnya.
- **Redundansi Spine**: minimal 2 Spine untuk menghindari single point of failure; idealnya 4 atau lebih untuk beban traffic besar.
- **BFD (Bidirectional Forwarding Detection)**: dipasang di atas BGP untuk mempercepat deteksi kegagalan link (sub-second failover) dibanding default BGP hold-timer.
- **Border Leaf redundant**: gunakan minimal 2 Border Leaf untuk koneksi keluar (WAN/DCI) agar tidak ada single point of failure saat menghubungkan ke luar fabric.
- **FortiGate HA**: gunakan minimal 2 unit dalam **FGCP cluster** (mode aktif-pasif atau aktif-aktif) agar internet gateway tidak menjadi single point of failure.
- **Palo Alto HA**: gunakan **HA pair** (aktif-pasif dengan session sync, atau aktif-aktif untuk throughput lebih tinggi) agar inspeksi client-to-server tetap berjalan saat salah satu unit down — jangan sampai kegagalan firewall justru memutus akses client ke seluruh server.

## 11. Monitoring dan Troubleshooting

Aspek yang perlu dipantau secara rutin:
- **BGP session state** (underlay dan overlay EVPN) — pastikan semua neighbor dalam status *Established*.
- **Utilisasi link Spine-Leaf** — untuk mendeteksi potensi bottleneck/hotspot traffic.
- **VTEP reachability** — pastikan semua VTEP saling ping via underlay loopback.
- **MAC/ARP table consistency** — bandingkan tabel MAC yang dipelajari via EVPN dengan kondisi aktual.
- **Telemetry streaming** (gNMI/gRPC) direkomendasikan dibanding SNMP tradisional untuk visibility real-time pada skala besar.

Tools yang umum digunakan: **NetBox** (source of truth/IPAM), **Prometheus + Grafana** (metrics), **Ansible/Nornir** (automation & config management), **ThousandEyes / Kentik** (traffic visibility).

## 12. Best Practices

1. **Gunakan automation** (Ansible, Nornir, atau vendor-specific seperti Cisco NDFC/Arista CVP) untuk deployment dan konsistensi konfigurasi — hindari konfigurasi manual pada skala besar.
2. **Standarisasi penomoran IP dan AS number** sejak awal desain (gunakan IPAM tool) agar mudah di-scale.
3. **Rencanakan oversubscription ratio** sesuai kebutuhan workload sebelum membeli hardware.
4. **Pisahkan underlay dan overlay secara jelas** — underlay cukup sederhana (hanya loopback & p2p link), kompleksitas policy diletakkan di overlay (EVPN).
5. **Gunakan BFD** untuk mempercepat convergence saat terjadi failure.
6. **Desain Border Leaf terpisah** dari Leaf yang menghubungkan server, agar traffic eksternal tidak mengganggu traffic internal fabric.
7. **Dokumentasikan topologi** menggunakan source-of-truth (NetBox/Nautobot) agar mudah diaudit dan diotomatisasi.
8. **Uji failover** secara berkala (link down, node down) untuk memastikan redundansi bekerja sesuai desain.

## 13. Kesimpulan

Arsitektur Spine-Leaf merupakan fondasi standar industri untuk datacenter modern karena mampu menyediakan **bandwidth tinggi, latency rendah dan predictable, skalabilitas horizontal, serta redundansi yang baik** — jauh lebih unggul dibanding arsitektur 3-tier tradisional untuk menangani trafik east-west yang mendominasi workload modern seperti virtualisasi, container, big data, dan AI/ML.

Kombinasi **BGP underlay** dan **VXLAN EVPN overlay** menjadi pendekatan paling umum dan direkomendasikan (mengacu pada RFC 7938 dan RFC 8365) untuk membangun fabric yang scalable, vendor-agnostic secara konsep, dan siap mendukung kebutuhan multi-tenancy datacenter modern.

## 14. Referensi

- RFC 7938 — Use of BGP for Routing in Large-Scale Data Centers
- RFC 8365 — A Network Virtualization Overlay Solution Using EVPN
- RFC 7348 — Virtual eXtensible Local Area Network (VXLAN)
- Dokumentasi resmi Cisco NX-OS VXLAN EVPN Configuration Guide
- Dokumentasi resmi Arista EOS VXLAN Configuration Guide
