# 5. MIDDLEWARE – XỬ LÝ TRƯỚC KHI VÀO CONTROLLER

## 5.1. Middleware là gì?

- Middleware là các hàm được gọi trước khi route handler (controller) xử lý request.
- Có quyền truy cập vào đối tượng `request` (req), `response` (res) và hàm `next()` trong vòng đời request-response.
- Tác vụ thường gặp:
  - Thực hiện logic tùy chỉnh trước khi request đến handler.
  - Sửa đổi request hoặc response.
  - Kết thúc vòng đời request-response bằng cách gửi response trực tiếp.
  - Gọi middleware tiếp theo bằng `next()`.
- Use case phổ biến:
  - Logging request
  - Xác thực (authentication)
  - Ủy quyền (authorization)
  - Parsing dữ liệu (JSON, URL-encoded)
  - Nén response
  - Thêm headers tùy chỉnh

---

## 5.2. Tạo custom middleware

- Trong NestJS, tạo custom middleware bằng cách implement interface `NestMiddleware`.
- Interface yêu cầu implement phương thức `use(req: Request, res: Response, next: NextFunction)`.

**Các bước tạo custom middleware:**
1. Tạo file middleware (vd: `logger.middleware.ts`)
2. Định nghĩa class implement `NestMiddleware`
3. Implement phương thức `use()`
4. (Tùy chọn) Có thể dùng function đơn giản (functional middleware)

**Ví dụ: logger.middleware.ts (class-based)**
```typescript
import { Injectable, NestMiddleware } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';

@Injectable()
export class LoggerMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    console.log(`[${new Date().toISOString()}] Request: ${req.method} ${req.url}`);
    next();
  }
}
```

**Ví dụ: logger.middleware.ts (functional)**
```typescript
import { Request, Response, NextFunction } from 'express';

export function LoggerMiddleware(req: Request, res: Response, next: NextFunction) {
  console.log(`[${new Date().toISOString()}] Functional Request: ${req.method} ${req.url}`);
  next();
}
```

---

## 5.3. Áp dụng middleware cho route, module, global

### Áp dụng cho route cụ thể

- Middleware được cấu hình ở cấp module, không dùng decorator như guard/interceptor/filter.
- Trong module, implement interface `NestModule` và dùng phương thức `configure(consumer: MiddlewareConsumer)`.
- Dùng `consumer.apply(LoggerMiddleware).forRoutes('some/route')` để áp dụng cho route cụ thể.
- Có thể dùng wildcard `*` hoặc object chỉ định method/path.

**Ví dụ: your.module.ts**
```typescript
import { Module, NestModule, MiddlewareConsumer } from '@nestjs/common';
import { YourController } from './your.controller';
import { YourService } from './your.service';
import { LoggerMiddleware } from '../logger.middleware';

@Module({
  controllers: [YourController],
  providers: [YourService],
})
export class YourModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes('your-resource');
    // Hoặc:
    // .forRoutes({ path: 'your-resource/:id', method: RequestMethod.GET });
  }
}
```

### Áp dụng cho module

- Dùng `forRoutes('*')` để áp dụng cho tất cả route trong module.

**Ví dụ: another.module.ts**
```typescript
import { Module, NestModule, MiddlewareConsumer } from '@nestjs/common';
import { AnotherController } from './another.controller';
import { AnotherService } from './another.service';
import { LoggerMiddleware } from '../logger.middleware';

@Module({
  controllers: [AnotherController],
  providers: [AnotherService],
})
export class AnotherModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes('*');
  }
}
```

### Áp dụng toàn cục

- Dùng `app.use()` trong `main.ts` để áp dụng middleware cho toàn bộ ứng dụng.
- Có thể truyền function hoặc class vào `app.use()`.

**Ví dụ: main.ts**
```typescript
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { LoggerMiddleware } from './logger.middleware';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.use(LoggerMiddleware); // Áp dụng toàn cục
  await app.listen(3000);
}
bootstrap();
```

---

## 5.4. Phân biệt middleware với interceptor & guard

| Đặc điểm       | Middleware                                | Interceptor                              | Guard                                   |
| -------------- | ----------------------------------------- | ---------------------------------------- | --------------------------------------- |
| Thời điểm chạy | Trước handler                             | Trước & sau handler                      | Trước handler                           |
| Quyền truy cập | req, res, next                            | ExecutionContext, CallHandler            | ExecutionContext                        |
| Mục đích chính | Tiền xử lý, logging, parsing, auth cơ bản | Biến đổi response, log, xử lý lỗi, cache | Ủy quyền/authorization                  |
| Cách sử dụng   | configure() trong module, app.use()       | @UseInterceptors() decorator             | @UseGuards() decorator                  |
| Kết quả trả về | next() hoặc kết thúc response             | Observable (RxJS)                        | boolean/Promise/Observable (true/false) |

**Tóm tắt:**
- **Middleware:** Tiền xử lý, sửa đổi req/res, logging, parsing, chạy trước handler.
- **Interceptor:** Biến đổi dữ liệu, log chi tiết, xử lý lỗi, chạy trước & sau handler.
- **Guard:** Quyết định quyền truy cập, chạy trước handler, trả về true/false.
