# Monitoring Stack

Stack monitoring berbasis Docker Compose untuk memantau resource host dan container Docker, lengkap dengan penyimpanan metrik (Prometheus) dan visualisasi (Grafana).

## Komponen

| Service | Image | Deskripsi | Port |
| --- | --- | --- | --- |
| `node-exporter` | `prom/node-exporter:latest` | Mengumpulkan metrik host (CPU, memory, disk, network) | `9100` |
| `cadvisor` | `gcr.io/cadvisor/cadvisor:latest` | Mengumpulkan metrik container Docker | `8080` |
| `prometheus` | `prom/prometheus:latest` | Scrape, menyimpan, dan query metrik (retensi 30 hari) | `9090` |
| `grafana` | `grafana/grafana:latest` | Dashboard dan visualisasi metrik | `127.0.0.1:3001` → `3000` |

Semua service terhubung ke network bridge `monitoring`. Data Prometheus dan Grafana disimpan pada volume `prometheus_data` dan `grafana_data` agar tetap ada saat container di-restart.

## Struktur Project

```
.
├── docker-compose.yml
├── prometheus/
│   └── prometheus.yml
└── README.md
```

## Prasyarat

- Docker Engine
- Docker Compose v2
- Host berbasis Linux (cadvisor dan node-exporter membutuhkan akses ke `/proc`, `/sys`, dan `/var/lib/docker`)

## Cara Menjalankan

Jalankan seluruh stack:

```bash
docker compose up -d
```

Cek status container:

```bash
docker compose ps
```

Lihat log:

```bash
docker compose logs -f prometheus
```

Hentikan stack:

```bash
docker compose down
```

Hentikan stack sekaligus hapus volume data:

```bash
docker compose down -v
```

## Akses

- **Prometheus**: http://localhost:9090
- **Grafana**: http://localhost:3001 (hanya dapat diakses dari localhost)
  - Username: `admin`
  - Password: `123456` (diatur melalui `GF_SECURITY_ADMIN_PASSWORD`)

Target scrape dapat diverifikasi di halaman **Status → Targets** pada Prometheus.

## Konfigurasi Prometheus

File konfigurasi berada di [prometheus.yml](prometheus/prometheus.yml):

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
  - job_name: 'node-exporter'
    static_configs:
      - targets: ['localhost:9100']
  - job_name: 'cadvisor'
    static_configs:
      - targets: ['localhost:8080']
```

Untuk menambah target baru, tambahkan entri pada `scrape_configs`, lalu reload Prometheus:

```bash
curl -X POST http://localhost:9090/-/reload
```

## Catatan

Beberapa hal berikut perlu diperhatikan pada [docker-compose.yml](docker-compose.yml):

- Mount path dan argumen `--config.file` Prometheus masih berupa placeholder (`...`). Keduanya perlu diubah menjadi `/etc/prometheus/prometheus.yml` dan `--config.file=/etc/prometheus/prometheus.yml` agar Prometheus dapat membaca konfigurasi.
- `prometheus` belum mem-publish port `9090` ke host. Tambahkan `ports: ["9090:9090"]` agar UI Prometheus dapat diakses.
- `node-exporter` dan `cadvisor` juga belum mem-publish port `9100` dan `8080` ke host.
- Target scrape di `prometheus.yml` menggunakan `localhost`, padahal tiap service berjalan di container terpisah. Gunakan hostname service pada network `monitoring` (misalnya `node-exporter:9100`, `cadvisor:8080`, `prometheus:9090`).
- Password admin Grafana masih default, sebaiknya diubah untuk pemakaian di luar lingkungan lokal.

## Lisensi

Belum ditentukan.
