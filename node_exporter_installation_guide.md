# Hướng dẫn cài đặt Node Exporter trên Server đích

## 1. Chuẩn bị trên server đích
Giả sử server đích chạy hệ điều hành **Ubuntu Linux amd64**.

**SSH vào server đích:**
```bash
ssh ubuntu@<TARGET_SERVER_IP>
```

**Kiểm tra kiến trúc CPU (architecture):**
```bash
uname -m
```
* Nếu trả về `x86_64`: Sử dụng bản **linux-amd64**.
* Nếu trả về `aarch64`: Sử dụng bản **linux-arm64**.

**Tạo user hệ thống riêng cho node_exporter:**
```bash
sudo useradd \
  --system \
  --no-create-home \
  --shell /usr/sbin/nologin \
  node_exporter
```

**Kiểm tra user đã tạo:**
```bash
id node_exporter
```

---

## 2. Download và giải nén node_exporter
*(Áp dụng cho kiến trúc `amd64`)*

**Tải gói cài đặt:**
```bash
cd /tmp
curl -LO https://github.com/prometheus/node_exporter/releases/download/v1.12.1/node_exporter-1.12.1.linux-amd64.tar.gz
```

**Giải nén:**
```bash
tar xzf node_exporter-1.12.1.linux-amd64.tar.gz
```

**Copy file thực thi (binary) vào hệ thống:**
```bash
sudo cp node_exporter-1.12.1.linux-amd64/node_exporter /usr/local/bin/node_exporter
```

**Phân quyền cho file:**
```bash
sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter
sudo chmod 755 /usr/local/bin/node_exporter
```

**Kiểm tra phiên bản:**
```bash
/usr/local/bin/node_exporter --version
```

> **Lưu ý:** Node Exporter là một static binary nên không yêu cầu cài đặt môi trường Go hay Docker trên server đích.

---

## 3. Tạo systemd service
Tạo service giúp `node_exporter` chạy như dịch vụ hệ thống và tự động khởi động khi server reboot.

**Tạo và mở file cấu hình service:**
```bash
sudo nano /etc/systemd/system/node_exporter.service
```

**Nội dung cấu hình:**
```ini
[Unit]
Description=Prometheus Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple

ExecStart=/usr/local/bin/node_exporter

Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```
*(Lưu lại file và thoát)*

---

## 4. Enable + Start Service

**Reload lại daemon systemd:**
```bash
sudo systemctl daemon-reload
```

**Kích hoạt và khởi chạy service:**
```bash
sudo systemctl enable --now node_exporter
```

**Kiểm tra trạng thái service:**
```bash
sudo systemctl status node_exporter
```
Cần đảm bảo thấy dòng: `Active: active (running)`.

---

## 5. Kiểm tra node_exporter trên server đích

**Kiểm tra danh sách metrics:**
```bash
curl http://localhost:9100/metrics
```

**Hoặc xem 20 dòng đầu tiên:**
```bash
curl http://localhost:9100/metrics | head -20
```

Các metric trả về sẽ có tiền tố `node_` như:
* `node_cpu_seconds_total`
* `node_memory_MemTotal_bytes`
* `node_memory_MemAvailable_bytes`
* `node_filesystem_avail_bytes`
* `node_network_receive_bytes_total`

**Kiểm tra riêng metric RAM:**
```bash
curl -s http://localhost:9100/metrics | grep node_memory_MemTotal_bytes
```