# 1. NestJS là gì? Tại sao nên dùng?

NestJS là một framework Node.js hiện đại để xây dựng ứng dụng backend hiệu quả, mở rộng tốt. Viết bằng TypeScript và áp dụng OOP, FP, FRP.

---

## 1.1. Tại sao nên dùng NestJS?

- Hỗ trợ mạnh TypeScript (có thể dùng JavaScript)
- Cấu trúc rõ ràng, dễ bảo trì – phù hợp team lớn
- Tích hợp sẵn: DI, Middleware, Pipes, Guards, Interceptors...
- Dựa trên Express (hoặc Fastify)
- Tốt cho microservices và GraphQL

---

## 1.2. So sánh NestJS với Express

| Tiêu chí          | NestJS                                   | Express                                 |
|-------------------|-------------------------------------------|------------------------------------------|
| Ngôn ngữ chính     | TypeScript                                | JavaScript (có hỗ trợ TypeScript)        |
| Cấu trúc dự án     | Rõ ràng, module hóa, dễ mở rộng           | Tùy biến, dễ rối nếu không kiểm soát     |
| Hỗ trợ DI          | Có sẵn, mạnh                              | Không có, tự cài và xử lý                 |
| Mức độ học         | Cao hơn do cấu trúc phức tạp hơn          | Dễ học, đơn giản                         |
| Hiệu năng          | Tương đương (dùng Express/Fastify)        | Tốt                                      |
| Tích hợp           | Phong phú (CLI, Middleware, Pipes...)     | Cần thêm thư viện ngoài                  |

👉 **Express** phù hợp dự án nhỏ/lẻ  
👉 **NestJS** phù hợp dự án lớn, tổ chức tốt, dễ mở rộng

---

## 1.3. Kiến trúc hướng module – Ưu điểm

NestJS dùng kiến trúc **module**, mỗi module chứa logic riêng biệt.

**Ưu điểm:**

- **Rõ ràng**: Phân chia module (UserModule, AuthModule…)
- **Tái sử dụng**: Dễ chia sẻ và tái dùng
- **Dễ test**: Test độc lập từng module
- **Tách biệt logic**: Controller, Service rõ ràng

---

## 1.4. Các khái niệm chính trong NestJS

| Khái niệm         | Mô tả                                                                 |
|-------------------|----------------------------------------------------------------------|
| **Module**        | Gom nhóm controller, service, provider liên quan                     |
| **Controller**    | Xử lý request, định tuyến                                            |
| **Service**       | Chứa logic nghiệp vụ, gọi từ controller                              |
| **Provider**      | Thành phần có thể inject (thường là service)                         |
| **Decorator**     | Đánh dấu metadata (@Controller, @Injectable, @Get...)                |
| **Middleware**    | Chạy trước khi request tới controller                                |
| **Guard**         | Xác thực và phân quyền                                               |
| **Pipe**          | Validate, transform dữ liệu                                          |
| **Interceptor**   | Can thiệp vào luồng xử lý (logging, timeout, modify response...)     |
| **Dependency Injection (DI)** | Cung cấp dependency tự động khi cần thiết            |
