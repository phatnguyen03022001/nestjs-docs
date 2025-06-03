## 9. Interceptors – Can thiệp dòng xử lý

Interceptors (bộ chặn) là một tính năng mạnh mẽ trong NestJS cho phép bạn can thiệp vào luồng xử lý request/response tại các điểm cụ thể, tương tự AOP (Aspect-Oriented Programming). Chúng giúp bạn thêm logic trước hoặc sau khi một handler được thực thi.

### 9.1. Interceptor là gì?

- Interceptor là class được đánh dấu `@Injectable()` và implement interface `NestInterceptor`.
- Có phương thức duy nhất `intercept(context: ExecutionContext, next: CallHandler): Observable<any>`.
- Mục đích:
    - Chuyển đổi response/request
    - Ghi log
    - Xử lý ngoại lệ/lỗi
    - Bộ nhớ đệm (caching)
    - Timeout

Interceptors tận dụng RxJS Observable và các operator để xử lý bất đồng bộ và luồng dữ liệu.

---

### 9.2. Tạo custom interceptor

Ví dụ custom interceptor chuẩn hóa response:

```typescript
// src/common/interceptors/transform.interceptor.ts
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

export interface Response<T> {
    data: T;
}

@Injectable()
export class TransformInterceptor<T> implements NestInterceptor<T, Response<T>> {
    intercept(context: ExecutionContext, next: CallHandler): Observable<Response<T>> {
        return next.handle().pipe(map(data => ({ data })));
    }
}
```

- `@Injectable()`: Đánh dấu class là provider.
- `intercept()`: Phương thức chính.
- `next.handle()`: Observable thực thi handler.
- `pipe(map(...))`: Biến đổi dữ liệu trả về thành `{ data: ... }`.

---

### 9.3. Use-case phổ biến

#### Logging

Ghi lại thời gian xử lý request:

```typescript
// src/common/interceptors/logging.interceptor.ts
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';

@Injectable()
export class LoggingInterceptor implements NestInterceptor {
    intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
        console.log('Before...');
        const now = Date.now();
        return next.handle().pipe(
            tap(() => console.log(`After... ${Date.now() - now}ms`)),
        );
    }
}
```

#### Response mapping

Chuẩn hóa cấu trúc response, ví dụ như `TransformInterceptor` ở trên.

#### Timeout

Hủy request nếu xử lý quá lâu:

```typescript
// src/common/interceptors/timeout.interceptor.ts
import { CallHandler, ExecutionContext, Injectable, NestInterceptor, RequestTimeoutException } from '@nestjs/common';
import { Observable, throwError, TimeoutError } from 'rxjs';
import { catchError, timeout } from 'rxjs/operators';

@Injectable()
export class TimeoutInterceptor implements NestInterceptor {
    intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
        return next.handle().pipe(
            timeout(5000),
            catchError(err => {
                if (err instanceof TimeoutError) {
                    return throwError(() => new RequestTimeoutException('Request processing timed out'));
                }
                return throwError(() => err);
            }),
        );
    }
}
```

#### Caching

Lưu trữ kết quả request để tăng hiệu suất:

```typescript
// src/common/interceptors/cache.interceptor.ts
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable, of } from 'rxjs';
import { tap } from 'rxjs/operators';

@Injectable()
export class CacheInterceptor implements NestInterceptor {
    private readonly cache = new Map<string, any>();

    intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
        const request = context.switchToHttp().getRequest();
        const url = request.url;

        if (this.cache.has(url)) {
            console.log(`Cache hit for ${url}`);
            return of(this.cache.get(url));
        }

        return next.handle().pipe(
            tap(response => {
                console.log(`Caching response for ${url}`);
                this.cache.set(url, response);
            }),
        );
    }
}
```

---

### 9.4. Kết hợp RxJS operators

- **map**: Biến đổi giá trị trả về từ Observable, ví dụ chuẩn hóa response.
- **catchError**: Bắt và xử lý lỗi Observable.

Ví dụ:

```typescript
import { catchError } from 'rxjs/operators';
import { throwError } from 'rxjs';
import { HttpException, HttpStatus } from '@nestjs/common';

return next.handle().pipe(
    catchError(err => {
        console.error('Error in handler:', err);
        if (err instanceof CustomApplicationError) {
            return throwError(() => new HttpException('Application specific error', HttpStatus.INTERNAL_SERVER_ERROR));
        }
        return throwError(() => err);
    }),
);
```

Các operator khác như `tap`, `delay`, `filter`, `mergeMap`... cũng có thể dùng tùy logic.

---

### 9.5. Global interceptors

Có thể áp dụng interceptor ở 3 cấp độ:

- **Method-scoped**: Dùng `@UseInterceptors()` trên method.

    ```typescript
    @Post()
    @UseInterceptors(TransformInterceptor)
    create(@Body() createUserDto: CreateUserDto) {
        return { id: 1, ...createUserDto };
    }
    ```

- **Controller-scoped**: Dùng `@UseInterceptors()` trên controller.

    ```typescript
    @Controller('users')
    @UseInterceptors(LoggingInterceptor, TransformInterceptor)
    export class UsersController {}
    ```

- **Global-scoped**: Đăng ký trong `main.ts` hoặc qua module.

    ```typescript
    // main.ts
    import { NestFactory } from '@nestjs/core';
    import { AppModule } from './app.module';
    import { TransformInterceptor } from './common/interceptors/transform.interceptor';

    async function bootstrap() {
        const app = await NestFactory.create(AppModule);
        app.useGlobalInterceptors(new TransformInterceptor());
        await app.listen(3000);
    }
    bootstrap();
    ```

    ```typescript
    // app.module.ts
    import { Module } from '@nestjs/common';
    import { APP_INTERCEPTOR } from '@nestjs/core';
    import { LoggingInterceptor } from './common/interceptors/logging.interceptor';

    @Module({
        providers: [
            {
                provide: APP_INTERCEPTOR,
                useClass: LoggingInterceptor,
            },
        ],
    })
    export class AppModule {}
    ```

**Thứ tự thực thi:**  
Global → Controller → Method (request vào); Method → Controller → Global (response ra).

---

Interceptors là công cụ mạnh để xử lý cross-cutting concerns mà không làm thay đổi logic controller.
