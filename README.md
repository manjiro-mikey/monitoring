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

* Hệ điều hành: Linux/Windows/macOS có hỗ trợ Docker.
* **Docker** & **Docker Compose** đã được cài đặt.
* RAM trống tối thiểu: `~2GB` (Hệ thống đã được thiết lập `mem_limit` để tránh tiêu thụ quá nhiều RAM).

---

## 3. Các bước triển khai

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
