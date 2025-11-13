# Laporan Tugas 1: Desain Jaringan VLSM & CIDR - Yayasan ARA

Laporan ini mendokumentasikan proses perencanaan, perhitungan, dan implementasi jaringan untuk Yayasan Pendidikan ARA berdasarkan soal yang diberikan.

**Data Utama:**
* **Base Network Unik:** `10.66.0.0` (Berdasarkan `10.(NRP mod 256).0.0`)

---

## Fase 1: Perencanaan Subnetting (Tabel VLSM)

**Tujuan:** Memecah (subnetting) *Base Network* `10.66.0.0` menjadi 7 jaringan (subnet) yang lebih kecil dan efisien menggunakan **VLSM (Variable Length Subnet Mask)**.

### 1.1. Logika Perhitungan

Perhitungan dilakukan dengan mengurutkan kebutuhan host dari terbesar ke terkecil, dengan satu pengecualian strategis:

> **Keputusan Desain Penting:** Jaringan `Server & Admin (6 host)` dialokasikan **sebelum** `Bidang Pengawas (18 host)`. Ini dilakukan agar semua 5 jaringan Kantor Pusat (Sekretariat, Kurikulum, Guru, Sarpras, Server) berada dalam satu blok IP yang berurutan (`10.66.0.0` s/d `10.66.3.199`). Keputusan ini krusial untuk **Fase 2 (CIDR)**.

### 1.2. Rumus dan Metode Kalkulasi

Setiap baris pada tabel VLSM dihitung sebagai berikut:

* **`Prefix` (/xx):**
    1.  Menghitung jumlah bit host ($n$) yang diperlukan: $2^n - 2 \ge \text{Kebutuhan Host}$.
    2.  Menghitung prefix: $\text{Prefix} = 32 - n$.
    * *Contoh (Sekretariat, 380 host):* $2^9 - 2 = 510$ (cukup). Maka $n=9$. Prefix = $32 - 9 = 23$.

* **`Subnet Mask`:** Representasi desimal dari Prefix.
    * *Contoh (/23):* `11111111.11111111.11111110.00000000` -> `255.255.254.0`

* **`Network Address`:** Alamat IP pertama di subnet, yang dimulai dari alamat yang valid setelah subnet sebelumnya berakhir.
    * *Contoh (Kurikulum):* Jaringan Sekretariat (`/23`) memakai 512 alamat (dari `10.66.0.0` s/d `10.66.1.255`). Maka, jaringan Kurikulum dimulai di `10.66.2.0`.

* **`Broadcast`:** Alamat IP terakhir di subnet.
    * *Contoh (Kurikulum, /24):* Memakai 256 alamat, dari `...2.0` s/d `...2.255`. Alamat broadcast-nya adalah `10.66.2.255`.

* **`Range Host Valid`:** Rentang IP yang dapat digunakan oleh perangkat (host).
    * Rumus: `(Network Address + 1)` s/d `(Broadcast - 1)`.
    * *Contoh (Kurikulum):* `10.66.2.1` - `10.66.2.254`.

* **`Gateway`:** IP "pintu keluar" untuk setiap LAN, yang akan dikonfigurasi pada interface router. Sesuai konvensi, IP valid pertama dari *range host* digunakan.
    * *Contoh (Kurikulum):* `10.66.2.1`.

### 1.3. Tabel VLSM Final

| Lokasi | Ruang / Jaringan | Jml Host | Network Address | Prefix | Subnet Mask | Range Host Valid | Broadcast | Gateway |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Pusat** | Sekretariat | 380 | `10.66.0.0` | /23 | `255.255.254.0` | `10.66.0.1` - `10.66.1.254` | `10.66.1.255` | `10.66.0.1` |
| **Pusat** | Bidang Kurikulum | 220 | `10.66.2.0` | /24 | `255.255.255.0` | `10.66.2.1` - `10.66.2.254` | `10.66.2.255` | `10.66.2.1` |
| **Pusat** | Bidang Guru & Tendik | 95 | `10.66.3.0` | /25 | `255.255.255.128` | `10.66.3.1` - `10.66.3.126` | `10.66.3.127` | `10.66.3.1` |
| **Pusat** | Bidang Sarana Prasarana | 45 | `10.66.3.128` | /26 | `255.255.255.192` | `10.66.3.129` - `10.66.3.190` | `10.66.3.191` | `10.66.3.129` |
| **Pusat** | Server & Admin | 6 | `10.66.3.192` | /29 | `255.255.255.248` | `10.66.3.193` - `10.66.3.198` | `10.66.3.199` | `10.66.3.193` |
| **Cabang**| Bidang Pengawas Sekolah | 18 | `10.66.3.224` | /27 | `255.255.255.224` | `10.66.3.225` - `10.66.3.254` | `10.66.3.255` | `10.66.3.225` |
| **WAN** | WAN (Pusat-Cabang) | 2 | `10.66.4.0` | /30 | `255.255.255.252` | `10.66.4.1` - `10.66.4.2` | `10.66.4.3` | `10.66.4.1` |

---

## Fase 2: Perencanaan Routing (Tabel CIDR)

**Tujuan:** Melakukan **Aggregasi Rute (Supernetting)** untuk semua 5 jaringan Kantor Pusat. Ini adalah kebalikan dari VLSM.

**Mengapa?** Agar `Router_Cabang` tidak perlu menghafal 5 rute berbeda untuk menjangkau Kantor Pusat. Cukup 1 rute ringkasan (supernet) saja, sehingga *routing table* menjadi lebih efisien.

### 2.1. Proses Agregasi (Visualisasi)

Kita perlu mencari prefix terkecil yang mencakup semua 5 jaringan Kantor Pusat:
1.  `10.66.0.0/23` (Mencakup `10.66.0.x` dan `10.66.1.x`)
2.  `10.66.2.0/24`
3.  `10.66.3.0/25`
4.  `10.66.3.128/26`
5.  `10.66.3.192/29`

Semua jaringan ini berada dalam rentang `10.66.0.0` s/d `10.66.3.199`.

Dengan melihat oktet ketiga dalam biner:
* `10.66.0.x` -> `000000`**`00`**
* `10.66.1.x` -> `000000`**`01`**
* `10.66.2.x` -> `000000`**`10`**
* `10.66.3.x` -> `000000`**`11`**

Bit yang sama (common prefix) di oktet ketiga adalah 6 bit pertama: **`000000`**.

Total bit prefix = 8 (oktet 1) + 8 (oktet 2) + 6 (oktet 3) = **22 bit**.
Maka, Supernet-nya adalah **`10.66.0.0/22`**.

### 2.2. Tabel Hasil CIDR (Ringkasan Rute)

Tabel ini adalah daftar rute final yang akan digunakan dalam konfigurasi.

| Deskripsi | Network | Mask | Prefix | Range Host | Broadcast |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Supernet Kantor Pusat** | `10.66.0.0` | `255.255.252.0` | /22 | `10.66.0.1` - `10.66.3.254` | `10.66.3.255` |
| Jaringan Kantor Cabang | `10.66.3.224` | `255.255.255.224` | /27 | `10.66.3.225` - `10.66.3.254` | `10.66.3.255` |
| Jaringan WAN | `10.66.4.0` | `255.255.255.252` | /30 | `10.66.4.1` - `10.66.4.2` | `10.66.4.3` |

---

## Fase 3: Implementasi Packet Tracer

Perencanaan dari tabel VLSM dan CIDR di atas diwujudkan dalam topologi Packet Tracer.

* **Topologi:** Dibuat desain `Router-on-a-Stick` (modifikasi), di mana 1 `Router_Pusat` menangani 5 LAN (via 5 interface fisik) dan 1 `Router_Cabang` menangani 1 LAN. Kedua router terhubung via link WAN Serial. Desain ini 100% cocok dengan 7 subnet yang dihitung di Tabel VLSM.

* **Konfigurasi Host (PC):** Setiap PC di 6 LAN dikonfigurasi secara statis dengan:
    1.  `IP Address` (diambil dari kolom "Range Host Valid").
    2.  `Subnet Mask` (sesuai prefix jaringan).
    3.  `Default Gateway` (diambil dari kolom "Gateway").

* **Konfigurasi Router (IP Interfaces):** Setiap interface router yang terhubung ke LAN atau WAN dikonfigurasi dengan IP yang sesuai:
    * Interface LAN (misal `G0/0/0` Sekretariat) diberi IP "Gateway" dari Tabel VLSM (`10.66.0.1`).
    * Interface WAN (misal `Se0/2/0` di Pusat) diberi IP pertama dari range WAN (`10.66.4.1`).

* **Konfigurasi Router (Static Routing):** Ini adalah implementasi dari Tabel CIDR.
    * **Di `Router_Cabang`:** Ditambahkan satu rute statis (Supernet) ke Kantor Pusat.
        ```bash
        ip route 10.66.0.0 255.255.252.0 10.66.4.1
        ```
    * **Di `Router_Pusat`:** Ditambahkan satu rute statis (spesifik) ke Kantor Cabang.
        ```bash
        ip route 10.66.3.224 255.255.255.224 10.66.4.2
        ```
