# Week 1: Linux Networking & Infrastructure Simulation (Home Lab)

**Project Status:** ✅ Completed
**Role:** System Engineer Intern
**Stack:** Linux (Ubuntu Server), VirtualBox, Bash, Networking (TCP/IP), Nginx

## 📌 Project Overview
Proyek ini adalah simulasi infrastruktur jaringan *On-Premise* "Mini Branch Office" menggunakan environment virtual. Tujuan utamanya adalah membangun **Linux Gateway Router** dari nol yang menghubungkan *Private Subnet* (Isolated) ke *Public Internet*, serta mengekspos layanan internal (Web Server) ke jaringan luar secara aman.

Proyek ini mendemonstrasikan pemahaman mendalam tentang Layer 2 (Data Link), Layer 3 (Network), dan Layer 4 (Transport) pada model OSI.

## 🏗️ Architecture Topology

```mermaid
graph LR
    Internet((Internet/WAN)) <-->|NAT| Adapter1[enp0s3: DHCP]
    subgraph "vLab-Router (Gateway)"
        Adapter1
        Kernel[Kernel IP Forwarding]
        IPTables[IPTables NAT & DNAT]
        Adapter2[enp0s8: 192.168.10.1]
    end
    
    Adapter2 <-->|Internal Network| ClientInterface[enp0s3: 192.168.10.2]
    
    subgraph "vLab-Client (App Server)"
        ClientInterface
        Nginx[Nginx Web Server :80]
    end

    User[Laptop Host] -.->|Port 8080| Adapter1
    IPTables -.->|Forward :80| Nginx
Host Machine: Asus Vivobook (16GB RAM)

Hypervisor: VirtualBox 7.0

Router Spec: 1 vCPU, 1GB RAM, Ubuntu Server 22.04

Client Spec: 1 vCPU, 1GB RAM, Ubuntu Server 22.04

⚙️ Configuration & Implementation
1. Network Configuration (Netplan)
Menggunakan systemd-networkd sebagai backend renderer.

Router Configuration (/etc/netplan/01-netcfg.yaml): Router bertindak sebagai Gateway. Interface 1 terhubung ke NAT (Internet), Interface 2 menjadi gerbang untuk jaringan lokal.

YAML

network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:             # WAN Interface
      dhcp4: true
    enp0s8:             # LAN Interface
      dhcp4: no
      addresses:
        - 192.168.10.1/24
Client Configuration (/etc/netplan/01-netcfg.yaml): Client dikonfigurasi dengan IP Statis dan Gateway mengarah ke Router.

YAML

network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      dhcp4: no
      addresses:
        - 192.168.10.2/24
      routes:
        - to: default
          via: 192.168.10.1
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
2. NAT & Port Forwarding (IPTables)
Konfigurasi firewall untuk mentranslasikan alamat jaringan.

Masquerade (SNAT): Mengizinkan Client (IP Private) mengakses internet dengan meminjam IP Router.

Bash

sudo iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE
Port Forwarding (DNAT): Mengarahkan trafik dari Router Port 8080 ke Client Port 80.

Bash

sudo iptables -t nat -A PREROUTING -i enp0s3 -p tcp --dport 8080 -j DNAT --to-destination 192.168.10.2:80
Forwarding Rules: Membuka jalur paket agar diizinkan melintas antar interface.

Bash

sudo iptables -A FORWARD -p tcp -d 192.168.10.2 --dport 80 -j ACCEPT
sudo iptables -A FORWARD -m state --state RELATED,ESTABLISHED -j ACCEPT
🔧 Troubleshooting & Challenges
Masalah Utama: Persistence Config vs Cloud-Init
Deskripsi: Setiap kali VM di-reboot, konfigurasi Netplan kembali ke default (DHCP) dan file manual tertimpa. Analisis Root Cause: Paket cloud-init secara otomatis men-generate file /etc/netplan/50-cloud-init.yaml saat boot, menimpa konfigurasi statis. Solusi:

Membuat override config untuk mematikan fitur network pada cloud-init:

Bash

echo "network: {config: disabled}" | sudo tee /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
Menghapus file yang digenerate otomatis:

Bash

sudo rm -f /etc/netplan/50-cloud-init.yaml
Membuat file konfigurasi manual yang permanen (01-netcfg.yaml).

Masalah: IP Forwarding Reset
Deskripsi: Setelah reboot, Client kehilangan koneksi internet. Solusi: Mengaktifkan packet forwarding di level kernel secara persisten via sysctl.

Edit /etc/sysctl.conf -> uncomment net.ipv4.ip_forward=1.

🚀 Verification Results
Connectivity: Client berhasil melakukan ping 8.8.8.8 dan ping google.com.

Routing: traceroute dari Client menunjukkan hop pertama adalah 192.168.10.1 (Router).

Service Exposure: Browser pada Host (Laptop) berhasil mengakses http://127.0.0.1:8080 dan menampilkan halaman default Nginx yang berjalan di dalam Client VM yang terisolasi.

Created as part of System Engineer Intensive Roadmap.


---

### Rangkuman: Apa Saja yang Kita Lakukan Hari Ini?
*(Ini catatan untuk kamu pribadi, tidak perlu masuk ke GitHub)*

1.  **Membangun Lab Virtual:** Kita setup 2 VM di VirtualBox dengan topologi "Internal Network" yang aman, hemat resource laptop.
2.  **Konfigurasi IP Manual:** Kita belajar pakai YAML di Netplan, bukan GUI. Kita menentukan mana IP Gateway dan mana IP Client.
3.  **Mengaktifkan Router:** Kita mengubah Ubuntu Server biasa menjadi Router dengan mengaktifkan `ip_forward`.
4.  **Manipulasi Trafik (IPTables):**
    * Kita bikin Client bisa internetan (NAT/Masquerade).
    * Kita bikin Web Server Client bisa dibuka dari laptop (DNAT/Port Forwarding).
5.  **Troubleshooting Masalah Restart:** Ini yang paling mahal. Kita mengalahkan `cloud-init` yang selalu mereset settingan kita, dan membuat konfigurasi kita **TAHAN BANTING (Persistent)** terhadap reboot.

**Langkah Selanjutnya:**
Simpan file ini, lakukan:
```bash
git add README.md
git commit -m "Update dokumentasi lengkap Week 1 Networking"
git push origin main