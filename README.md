# Hướng dẫn triển khai hệ thống Monitoring & Observability Stack

Đây là tài liệu hướng dẫn tổng quan về kiến trúc và cách triển khai hệ thống Monitoring sử dụng hệ sinh thái **VictoriaMetrics**, **Grafana** và **Tempo**. Hệ thống này cung cấp giải pháp toàn diện cho cả 3 trụ cột của Observability: **Metrics**, **Logs** và **Traces**.

## 1. Kiến trúc hệ thống (Components)

Hệ thống được đóng gói thông qua Docker Compose (`compose.yaml`) bao gồm các thành phần sau:

### 1.1. Core Storage (Lưu trữ)
* **VictoriaMetrics (Live Metrics):** Cơ sở dữ liệu Time-series cực nhanh dùng để lưu trữ metrics tức thời (Lưu trữ 24 ngày).
* **Report (Long-term Metrics):** Một instance VictoriaMetrics khác dùng để lưu trữ metrics dài hạn phục vụ báo cáo (Lưu trữ 365 ngày).
* **VictoriaLogs (Logs Storage):** Hệ thống lưu trữ Log tập trung hiệu năng cao (Lưu trữ 90 ngày).
* **Tempo (Traces Storage):** Backend lưu trữ Distributed Tracing từ Grafana.

### 1.2. Thu thập & Cảnh báo (Scraping & Alerting)
* **Node Exporter:** Agent chạy trên máy chủ để phơi bày (expose) các metrics về phần cứng và hệ điều hành (CPU, RAM, Disk, Network).
* **vmagent:** Agent nhẹ của VictoriaMetrics, đóng vai trò cào (scrape) metrics từ Node Exporter và các target khác (cấu hình trong `vmagent/prometheus.yml`) rồi đẩy về hệ thống lưu trữ.
* **vmalert:** Thực thi các Rules (Alerting và Recording) dựa trên dữ liệu từ VictoriaMetrics.
* **Alertmanager:** Tiếp nhận cảnh báo từ `vmalert` và điều phối/gửi thông báo (Slack, Email, Telegram...).

### 1.3. Routing, Security & Visualization
* **vmauth:** API Gateway/Router quản lý xác thực và điều hướng dữ liệu. Mọi dữ liệu đẩy vào hệ thống (Metrics/Logs) đều đi qua đây (Port `8427`).
* **Grafana:** Nền tảng hiển thị Dashboards trực quan (Port `3000`). Đã được cài sẵn plugin VictoriaLogs và tự động cấu hình các data sources.

---

## 2. Yêu cầu hệ thống (Prerequisites)

* Hệ điều hành: Linux (Ubuntu/Debian) được khuyến nghị.
* RAM trống tối thiểu: `~2GB`. Khuyến nghị bật Swap nếu VPS ít RAM.

### 2.1. Cài đặt Docker & Docker Compose
Nếu server của bạn chưa có Docker, hãy chạy chuỗi lệnh sau để cài đặt (dành cho Ubuntu/Debian):

```bash
sudo apt update
sudo apt install -y docker.io docker-compose-v2
sudo systemctl enable --now docker
# Thêm user hiện tại vào nhóm docker để không cần dùng sudo khi gõ lệnh docker
sudo usermod -aG docker $USER
```
*(Lưu ý: Bạn có thể cần đăng xuất và đăng nhập lại SSH để quyền usermod có hiệu lực).*

### 2.2. Bổ sung RAM ảo (Swap)
Do stack Monitoring gồm nhiều thành phần (Grafana, VictoriaMetrics, v.v.), nếu server của bạn có ít RAM (nhỏ hơn 4GB), hãy tạo thêm 2GB Swap để tránh tình trạng tràn RAM (OOM - Out of Memory):

```bash
# Tạo file swap 2GB
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# Cấu hình tự động bật swap khi khởi động lại máy
echo '/swapfile swap swap defaults 0 0' | sudo tee -a /etc/fstab

# Kiểm tra lại xem swap đã nhận chưa
free -h
```

---

## 3. Các bước chuẩn bị thư mục (Dành cho Server mới)
Nếu bạn setup từ đầu trên một máy chủ trắng, hãy tạo các thư mục cần thiết phân quyền tương tự lệnh sau:
```bash
sudo mkdir -p /opt/monitoring
sudo chown -R $USER:$USER /opt/monitoring
cd /opt/monitoring

# Tạo các thư mục mount cho file config
mkdir -p vmauth vmalert alertmanager grafana/provisioning/datasources tempo/data

# Tạo các thư mục lưu trữ dữ liệu với quyền root
sudo mkdir -p /data/vmetrics-data /data/report-data /data/vlogs-data

# Tạo folder mount dữ liệu cho tempo tránh lỗi khi start tempo
sudo mkdir -p tempo/data/wal
sudo mkdir -p tempo/data/blocks
sudo chown -R 10001:10001 tempo/data

# Create folder and add permission for pyroscope
mkdir -p pyroscope/data
sudo chown -R 10001:10001 pyroscope/data

# Create folder and add permission for uptime
mkdir -p uptime-kuma/data
sudo chown -R 10001:10001 uptime-kuma/data
```

---

## 4. Khởi động hệ thống

### Bước 1: Chuẩn bị cấu hình
1. Kiểm tra file `vmagent/prometheus.yml` để đảm bảo đã cấu hình đúng các mục tiêu (targets) cần thu thập metrics.
2. Kiểm tra file `vmauth/auth.yml` để xem thông tin tài khoản đẩy dữ liệu (Basic Auth). Đảm bảo username/password khớp với cấu hình trong ứng dụng của bạn.
3. Kiểm tra file `alertmanager/alertmanager.yml` nếu bạn muốn nhận cảnh báo qua Slack/Telegram.

### Bước 2: Khởi động hệ thống
Mở terminal/command prompt tại thư mục chứa file `compose.yaml` (thư mục `stack`), chạy lệnh sau để khởi động toàn bộ hệ thống dưới dạng background:

```bash
docker compose up -d
```

### Bước 3: Kiểm tra trạng thái
Kiểm tra xem tất cả các container đã chạy ổn định chưa:
```bash
docker compose ps
```

Nếu có container nào bị thoát (Exit), bạn có thể xem log của nó bằng lệnh:
```bash
docker compose logs -f <tên_container>
# Ví dụ: docker compose logs -f grafana
```

---

## 4. Thông tin truy cập

Sau khi hệ thống khởi động thành công, bạn có thể truy cập các dịch vụ qua các địa chỉ sau:

| Dịch vụ | Địa chỉ truy cập / Endpoint | Ghi chú |
| :--- | :--- | :--- |
| **Grafana** | `http://localhost:3000` | Username: `admin` / Password: `123123` |
| **vmauth (Ingest)** | `http://localhost:8427` | Port dùng để nhận dữ liệu Metrics/Logs từ các ứng dụng đẩy về. |
| **Tempo (OTLP/gRPC)** | `localhost:4317` | Port để nhận Traces (giao thức gRPC) |
| **Tempo (OTLP/HTTP)** | `http://localhost:4318` | Port để nhận Traces (giao thức HTTP) |
| **Node Exporter** | `http://localhost:9100/metrics` | Endpoint xem metrics hệ điều hành |

---

## 5. Hướng dẫn tích hợp ứng dụng (Instrumentation)

Để ứng dụng của bạn (Node.js, Java, Python...) gửi dữ liệu về stack này:

* **Logs (VictoriaLogs OTLP):** 
  - Endpoint: `http://<IP>:8427/insert/opentelemetry/v1/logs`
  - Header yêu cầu: `Authorization: Basic <Base64(user:pass)>`
* **Traces (Tempo OTLP):**
  - Endpoint: `http://<IP>:4318/v1/traces`
* **Metrics (Push OTLP hoặc vmagent scrape):**
  - Bạn có thể cấu hình ứng dụng expose endpoint `/metrics` (định dạng Prometheus) và thêm target vào file `vmagent/prometheus.yml`.

---

## 6. Dừng hoặc Gỡ cài đặt

* Để dừng hệ thống nhưng vẫn giữ lại dữ liệu:
  ```bash
  docker compose stop
  ```
* Để dừng và gỡ bỏ hoàn toàn hệ thống (Bao gồm xóa cả dữ liệu trong volume):
  ```bash
  docker compose down -v
  ```
