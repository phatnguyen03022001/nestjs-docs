## 15. Tính năng nâng cao trong phát triển ứng dụng
Khi xây dựng các ứng dụng hiện đại, việc tích hợp các tính năng nâng cao không chỉ giúp cải thiện hiệu suất mà còn tăng cường khả năng bảo trì và trải nghiệm người dùng. Dưới đây là một số tính năng quan trọng:

---

### 13.1. Swagger API Docs

**Swagger (OpenAPI)** là bộ công cụ mã nguồn mở giúp thiết kế, xây dựng, ghi chú và sử dụng dịch vụ web RESTful. Nó cho phép mô tả cấu trúc API một cách chuẩn hóa và tự động tạo tài liệu API tương tác.

**Lợi ích:**
- **Tạo tài liệu tự động:** Tiết kiệm thời gian, đảm bảo tài liệu luôn đồng bộ với mã nguồn.
- **Thử nghiệm API trực tiếp:** Giao diện người dùng để gửi yêu cầu và kiểm tra endpoint ngay trên trình duyệt.
- **Phát triển đồng bộ:** Giúp frontend và backend hiểu rõ API, giảm hiểu lầm, tăng tốc phát triển.
- **Tạo mã client/server:** Tự động sinh mã client (SDK) hoặc server từ định nghĩa OpenAPI.

---

### 13.2. File Upload với Multer

**Multer** là middleware Node.js chuyên xử lý dữ liệu `multipart/form-data`, thường dùng cho upload tệp tin. Multer xây dựng trên busboy để tăng hiệu quả.

**Cách hoạt động:**  
Multer phân tích dữ liệu tệp đến, lưu vào thư mục tạm hoặc bộ nhớ, cung cấp thông tin tệp (tên, kích thước, mimetype, đường dẫn) cho controller xử lý.

**Tính năng nổi bật:**
- Hỗ trợ nhiều phương thức lưu trữ (disk, memory).
- Giới hạn kích thước tệp, loại tệp.
- Dễ tích hợp với các framework như Express, NestJS.

---

### 13.3. Redis cho Caching

**Redis** là một kho lưu trữ cấu trúc dữ liệu trong bộ nhớ (key-value store) mã nguồn mở. Redis rất phổ biến cho mục đích caching vì tốc độ cực nhanh và hỗ trợ nhiều kiểu dữ liệu (chuỗi, hash, list, set, sorted set).

**Lợi ích khi dùng Redis cho caching:**
- Tăng tốc độ truy xuất dữ liệu.
- Giảm tải cho cơ sở dữ liệu.
- Cải thiện khả năng mở rộng của ứng dụng.

---

### 13.4. Rate Limiting

**Rate Limiting** là kỹ thuật kiểm soát số lượng yêu cầu mà một người dùng, địa chỉ IP hoặc ứng dụng có thể thực hiện đến API hoặc dịch vụ trong một khoảng thời gian nhất định.

**Mục đích:**
- Bảo vệ khỏi tấn công DDoS/Brute Force.
- Ngăn chặn lạm dụng API.
- Đảm bảo công bằng tài nguyên cho người dùng.

**Cách triển khai:**  
Thường sử dụng các thuật toán như Token Bucket hoặc Leaky Bucket, kết hợp với bộ nhớ cache (như Redis) để lưu trữ số lượng yêu cầu của từng người dùng.

---

### 13.5. WebSocket (Gateway)

**WebSocket** là giao thức truyền thông hai chiều qua một kết nối TCP duy nhất. Khác với HTTP, WebSocket duy trì kết nối mở giữa client và server, cho phép trao đổi dữ liệu thời gian thực mà không cần gửi lại yêu cầu HTTP cho mỗi lần giao tiếp.

**Ứng dụng:**
- Ứng dụng chat.
- Trò chơi trực tuyến.
- Dashboard thời gian thực.
- Thông báo đẩy.

**Gateway trong NestJS:**  
Gateways là module cho phép tích hợp và quản lý kết nối WebSocket, xử lý các sự kiện đến và đi một cách dễ dàng.

---

### 13.6. EventEmitter

**EventEmitter** là module cốt lõi của Node.js, cung cấp cơ chế lập trình hướng sự kiện (event-driven programming). Cho phép tạo đối tượng phát ra sự kiện (emit) và các đối tượng khác lắng nghe (listen) để phản ứng lại sự kiện đó.

**Cách hoạt động:**
- **Phát sự kiện (emit):** Một phần mã phát ra sự kiện với tên cụ thể và dữ liệu kèm theo.
- **Lắng nghe sự kiện (on/addListener):** Đăng ký callback để thực thi khi sự kiện được phát ra.

**Lợi ích:**
- Tách biệt logic, dễ bảo trì và mở rộng.
- Hỗ trợ tốt cho tác vụ bất đồng bộ.
- Tạo thành phần tái sử dụng, giao tiếp qua sự kiện mà không phụ thuộc vào triển khai cụ thể.