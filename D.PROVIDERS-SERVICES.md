# 4. PROVIDERS & SERVICES – LOGIC NGHIỆP VỤ

## 4.1. Service là gì? Vai trò của Provider

- **Service** là nơi chứa **logic nghiệp vụ** của ứng dụng.
- **Provider** là khái niệm tổng quát trong NestJS, bao gồm:
    - Service
    - Repository
    - Factory
    - Helper
    - ...
- Provider có thể được **inject** vào các class khác (như Controller hoặc Service khác) thông qua **Dependency Injection**.

---

## 4.2. Dependency Injection trong NestJS

- NestJS sử dụng **Dependency Injection (DI)** để quản lý các dependency trong ứng dụng.
- DI giúp:
    - Giảm sự phụ thuộc giữa các class
    - Làm cho code **dễ bảo trì** và **dễ kiểm thử** hơn
- NestJS sử dụng **container nội bộ** để khởi tạo và cung cấp các instance của Provider.

---

## 4.3. `@Injectable()` và `useClass`, `useValue`, `useFactory`

- `@Injectable()` decorator:
    - Đánh dấu một class là **Provider**
    - Cho phép class đó được **inject** vào nơi khác

- **Cấu hình Provider tùy chỉnh** trong module:
    ```ts
    {
        provide: 'TOKEN',
        useClass: SomeClass,     // Dùng class để tạo provider
        useValue: someValue,     // Dùng một giá trị cụ thể
        useFactory: () => {...}, // Dùng một hàm để tạo provider
    }
    ```

    **Giải thích:**
    - `useClass`: Chỉ định class được sử dụng khi inject.
    - `useValue`: Dùng một giá trị cụ thể (object, primitive, function...).
    - `useFactory`: Dùng một hàm để tạo provider, có thể inject dependencies vào hàm.

---

## 4.4. Scope của Providers (Singleton, Request, Transient)

- **Singleton** (mặc định): Một instance duy nhất của Provider được chia sẻ trong toàn ứng dụng.
- **Request**: Một instance mới của Provider được tạo cho mỗi request.
- **Transient**: Một instance mới của Provider được tạo cho mỗi class inject nó.

---

## 4.5. Tổ chức Service trong Modules

- Service thường được khai báo trong `providers` array của Module.
- Các Module khác muốn sử dụng Service này cần import Module chứa nó.
- Có thể export Service từ Module để các Module khác có thể sử dụng.
