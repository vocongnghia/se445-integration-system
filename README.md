# Nền tảng Tích hợp ABC Retail — khung dự án

Khung này đã chạy được. Việc của nhóm là **điền phần logic còn thiếu**, không phải dựng lại từ đầu.

---

## 1. Chạy lên

Giải nén vào đường dẫn **không dấu tiếng Việt, không khoảng trắng**, ví dụ `C:\projects\si-platform`.

```bash
docker compose up -d --build
docker compose ps
```

Lần đầu mất 3–5 phút để tải ảnh và cài thư viện. Những lần sau chỉ vài giây.

| Địa chỉ | Là gì |
|---|---|
| http://localhost:8000 | **API Gateway** — cổng vào duy nhất, mọi request đi qua đây |
| http://localhost:3000 | Dashboard cho CEO |
| http://localhost:15672 | RabbitMQ Management — `guest` / `guest` |
| http://localhost:8001 | Kong Admin — kiểm tra route đã nạp chưa |

Kiểm tra nhanh — lệnh này **phải chạy được ngay** sau khi `up`:

```bash
curl http://localhost:8000/api/v1/products/SKU123/stock
```

Kết quả mong đợi:

```json
{"sku":"SKU123","name":"Ao thun co tron","availableStock":42,"warehouseId":"HCM-01"}
```

Thấy kết quả này nghĩa là cả chuỗi Gateway → Integration Service → System API → WMS đã thông.

### Không có Docker?

Chạy trực tiếp bằng Node 18+, mở 5 cửa sổ terminal:

```bash
node mocks/services/wms.js       # cổng 3001
node mocks/services/tms.js       # cổng 3002
node mocks/services/crm.js       # cổng 3003
node mocks/services/payment.js   # cổng 3004

cd integration-service && npm install && npm start   # cổng 4000
```

Mất Gateway và RabbitMQ, nhưng API đồng bộ vẫn hoạt động để nhóm phát triển.

---

## 2. Cấu trúc

```
si-platform/
├── docker-compose.yml          ← dựng cả hệ thống
├── kong/kong.yml               ← route + rate limit. TODO: bật JWT
│
├── integration-service/        ← PHẦN CHÍNH CỦA NHÓM
│   └── src/
│       ├── index.js            ← các endpoint
│       ├── system-api/         ← TẦNG 1: bọc hệ thống cũ, dịch tên trường
│       ├── process-api/        ← TẦNG 2: logic nghiệp vụ + consumer
│       ├── experience-api/     ← TẦNG 3: đóng gói cho từng client
│       └── messaging/          ← RabbitMQ: publish, consume, retry, DLQ
│
├── dashboard/                  ← giao diện giám sát, đã chạy được
│
└── mocks/                      ← HỆ THỐNG CỦA KHÁCH HÀNG — KHÔNG SỬA
    ├── services/               ← WMS, TMS, CRM, Payment
    ├── data/                   ← dữ liệu POS cũ cho phần AI
    └── API-mock-systems.md     ← tài liệu API
```

> **Quy định:** không sửa thư mục `mocks/`. Đó là hệ thống của khách hàng — ngoài đời bạn cũng không có quyền sửa. Hôm bảo vệ giảng viên sẽ thay bằng bản gốc.

---

## 3. Việc cần làm — theo thứ tự

Mỗi chỗ cần viết đều có chú thích `TODO` trong mã nguồn. Tìm bằng `Ctrl+Shift+F` với từ khoá `TODO`.

### Đã chạy sẵn, không phải làm
- Gateway định tuyến và giới hạn tần suất
- `GET /api/v1/products/{sku}/stock` — luồng đồng bộ đầy đủ
- Khung publish/consume có sẵn retry 3 lần và Dead Letter Queue
- Dashboard hiển thị trạng thái 4 hệ thống
- Cấu trúc thông báo lỗi thống nhất

### Kịch bản 1 — Đặt hàng *(làm trước nhất)*
| File | Việc |
|---|---|
| `system-api/index.js` | Viết `reserveStock`, `createShipment`, `addLoyaltyPoints` |
| `process-api/orders.js` | Hoàn thiện `createOrder`: kiểm tồn kho → lưu đơn → phát sự kiện |
| `process-api/consumers.js` | Viết consumer TMS và CRM |

Xong bước này là có luồng đồng bộ + bất đồng bộ chạy thật.

### Kịch bản 2 — Bảo mật
| File | Việc |
|---|---|
| `kong/kong.yml` | Bỏ comment khối `consumers` và plugin `jwt` ở cuối file |

Rate limiting đã bật sẵn 20 request/phút. Thử: bấm nút *Bắn 30 request* trên Dashboard.

### Kịch bản 3 — Chống trùng và hàng đợi lỗi
| File | Việc |
|---|---|
| `process-api/orders.js` | Xử lý `Idempotency-Key`: cùng khoá → trả đơn cũ |
| `messaging/index.js` | Bỏ comment phần kiểm tra `eventId` đã xử lý |
| `process-api/consumers.js` | Lưu `eventId` sau khi xử lý thành công |

Thử: `docker compose stop mock-tms`, đặt một đơn, xem hàng đợi `orders.dlq` trên RabbitMQ.

### Kịch bản 5 — AI ánh xạ dữ liệu
| File | Việc |
|---|---|
| `src/index.js` | Hoàn thiện `POST /api/v1/ai/mapping` |

Dữ liệu đầu vào: `mocks/data/pos-legacy-samples.json`. Gọi Ollama ở `http://host.docker.internal:11434` hoặc OpenAI API. **Phải kiểm tra kết quả AI trả về trước khi dùng** — AI hay trả thiếu trường hoặc sai kiểu.

### Kịch bản 6 — Đóng gói
Đã xong. Chỉ cần giữ cho `docker compose down -v && docker compose up --build` luôn chạy được trên máy sạch.

---

## 4. Lệnh dùng hằng ngày

```bash
docker compose logs -f integration-service   # xem log trực tiếp
docker compose restart integration-service   # áp dụng thay đổi code
docker compose restart kong                  # áp dụng thay đổi kong.yml
docker compose stop mock-tms                 # giả lập TMS chết (kịch bản 3)
docker compose start mock-tms                # bật lại
docker compose down                          # tắt hết
docker compose down -v && docker compose up --build   # lệnh giảng viên dùng khi chấm
```

Xem dữ liệu bên trong hệ thống cũ — rất tiện khi demo:

```
http://localhost:3001/api/v1/wms/_debug/reservations
http://localhost:3002/api/v1/tms/_debug/shipments
http://localhost:3003/api/v1/crm/_debug/points-log
```

---

## 5. Lưu ý

**Trong mạng Docker gọi nhau bằng tên container**, không dùng `localhost`. Đã cấu hình sẵn trong biến môi trường: `WMS_URL=http://mock-wms:3001`.

**Dữ liệu nằm trong bộ nhớ.** `docker compose down` là tồn kho về 42, mọi đơn hàng biến mất. Tiện để thử lại nhiều lần. Chuyển sang PostgreSQL khi nhóm thấy cần.

**TMS không chống trùng.** Gọi hai lần cùng `orderId` sẽ tạo hai vận đơn. Đây là ràng buộc của nhà cung cấp, nhóm phải tự xử lý ở phía mình.

**Chỉ một người sửa `docker-compose.yml`.** Đây là file gây xung đột khi merge nhiều nhất.

---

## 6. Lỗi hay gặp

| Triệu chứng | Xử lý |
|---|---|
| `port is already allocated` | Đổi số cổng bên trái trong `docker-compose.yml` |
| `Cannot connect to the Docker daemon` | Chưa mở Docker Desktop |
| Kong trả 404 cho mọi request | `docker compose restart kong`, kiểm tra `http://localhost:8001/routes` |
| Integration Service báo không kết nối được RabbitMQ | Bình thường lúc mới khởi động, nó tự thử lại 10 lần |
| Sửa code mà không thấy đổi | `docker compose up -d --build integration-service` |
| Lỗi lạ trên Windows | Đường dẫn có dấu tiếng Việt — chuyển sang `C:\projects\` |
