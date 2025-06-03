## 6. EXCEPTION FILTERS – XỬ LÝ LỖI

### 6.1. Cơ chế xử lý exception mặc định của NestJS

- Khi một exception không được xử lý trong route handler, NestJS sẽ tự động bắt và xử lý nó.
- Với các exception thuộc loại `HttpException` (hoặc các lớp kế thừa), NestJS tạo response HTTP với status code và JSON body tương ứng.
- JSON body mặc định gồm:
    - `statusCode`: Mã trạng thái HTTP.
    - `message`: Thông báo lỗi (string hoặc array).
    - `error`: Tên lỗi (ví dụ: "Unauthorized", "Not Found").
- Nếu exception không phải `HttpException`, NestJS trả về status code 500 và message mặc định "Internal server error".

---

### 6.2. Tạo custom Exception Filter

- Để tùy chỉnh xử lý exception, tạo custom Exception Filter bằng cách implement interface `ExceptionFilter`.
- Interface yêu cầu phương thức `catch(exception: any, host: ArgumentsHost)`.
    - `exception`: Instance của exception bị ném ra.
    - `host`: Đối tượng `ArgumentsHost` cung cấp các utility function lấy context phù hợp (HTTP, WebSocket, gRPC). Với REST API, dùng `host.switchToHttp()`.

**Các bước tạo custom Exception Filter:**
1. Tạo file filter (ví dụ: `http-exception.filter.ts`).
2. Định nghĩa class implement `ExceptionFilter` và decorate với `@Catch()`.
3. Trong `@Catch()`, truyền vào loại exception filter sẽ xử lý (hoặc bỏ trống để xử lý tất cả).
4. Implement phương thức `catch()` để xử lý exception và tạo response tùy chỉnh.

**Ví dụ: `http-exception.filter.ts`**
```typescript
import { ExceptionFilter, Catch, ArgumentsHost, HttpException } from '@nestjs/common';
import { Request, Response } from 'express';

@Catch(HttpException)
export class HttpExceptionFilter implements ExceptionFilter {
    catch(exception: HttpException, host: ArgumentsHost) {
        const ctx = host.switchToHttp();
        const response = ctx.getResponse<Response>();
        const request = ctx.getRequest<Request>();
        const status = exception.getStatus();
        const exceptionResponse = exception.getResponse();
        const errorMessage =
            typeof exceptionResponse === 'string'
                ? exceptionResponse
                : (exceptionResponse as { message: string[] | string }).message;

        console.error(`HTTP Exception ${status}: ${request.method} ${request.url} - ${JSON.stringify(errorMessage)}`);

        response.status(status).json({
            statusCode: status,
            message: errorMessage,
            timestamp: new Date().toISOString(),
            path: request.url,
        });
    }
}
```

**Ví dụ filter bắt tất cả exception: `all-exceptions.filter.ts`**
```typescript
import { ExceptionFilter, Catch, ArgumentsHost, HttpException, HttpStatus } from '@nestjs/common';
import { HttpAdapterHost } from '@nestjs/core';

@Catch()
export class AllExceptionsFilter implements ExceptionFilter {
    constructor(private readonly httpAdapterHost: HttpAdapterHost) {}

    catch(exception: any, host: ArgumentsHost): void {
        const { httpAdapter } = this.httpAdapterHost;
        const ctx = host.switchToHttp();

        const httpStatus =
            exception instanceof HttpException
                ? exception.getStatus()
                : HttpStatus.INTERNAL_SERVER_ERROR;

        const responseBody = {
            statusCode: httpStatus,
            message: exception.message || 'Internal server error',
            timestamp: new Date().toISOString(),
            path: httpAdapter.getRequestUrl(ctx.getRequest()),
        };

        httpAdapter.reply(ctx.getResponse(), responseBody, httpStatus);
    }
}
```

---

### 6.3. Áp dụng filter ở controller, route, global

- **Controller:** Dùng decorator `@UseFilters()` ở cấp class để áp dụng filter cho tất cả route handlers trong controller.

    ```typescript
    import { Controller, Get, UseFilters, HttpException } from '@nestjs/common';
    import { HttpExceptionFilter } from './http-exception.filter';

    @Controller('users')
    @UseFilters(HttpExceptionFilter)
    export class UsersController {
        @Get(':id')
        async findOne(id: string) {
            if (!id) {
                throw new HttpException('User ID is required', 400);
            }
            return { id };
        }
    }
    ```

- **Route:** Dùng `@UseFilters()` ở cấp method để áp dụng filter cho route handler cụ thể.

    ```typescript
    import { Controller, Get, Param, UseFilters, HttpException } from '@nestjs/common';
    import { HttpExceptionFilter } from './http-exception.filter';

    @Controller('products')
    export class ProductsController {
        @Get(':id')
        @UseFilters(new HttpExceptionFilter())
        async findOne(@Param('id') id: string) {
            if (!id) {
                throw new HttpException('Product ID is required', 400);
            }
            return { id };
        }
    }
    ```

- **Global:** Dùng `useGlobalFilters()` trong `main.ts` để áp dụng filter cho toàn bộ ứng dụng.

    ```typescript
    import { NestFactory } from '@nestjs/core';
    import { AppModule } from './app.module';
    import { HttpExceptionFilter } from './http-exception.filter';

    async function bootstrap() {
        const app = await NestFactory.create(AppModule);
        app.useGlobalFilters(new HttpExceptionFilter());
        await app.listen(3000);
    }
    bootstrap();
    ```

---

### 6.4. HttpException và các loại lỗi chuẩn

- `HttpException` là class cơ sở cho các exception HTTP trong NestJS.
- Nhận hai tham số khi khởi tạo:
    - `response`: String hoặc object mô tả lỗi.
    - `status`: Mã trạng thái HTTP (400, 401, 404, 500, ...).
- NestJS cung cấp nhiều lớp exception chuẩn kế thừa từ `HttpException`:
    - `BadRequestException` (400)
    - `UnauthorizedException` (401)
    - `ForbiddenException` (403)
    - `NotFoundException` (404)
    - `NotAcceptableException` (406)
    - `RequestTimeoutException` (408)
    - `ConflictException` (409)
    - `GoneException` (410)
    - `PayloadTooLargeException` (413)
    - `UnsupportedMediaTypeException` (415)
    - `ImATeapotException` (418)
    - `TooManyRequestsException` (429)
    - `InternalServerErrorException` (500)
    - `NotImplementedException` (501)
    - `BadGatewayException` (502)
    - `ServiceUnavailableException` (503)
    - `GatewayTimeoutException` (504)
    - `HttpVersionNotSupportedException` (505)
- Sử dụng các exception chuẩn giúp trả về lỗi nhất quán, tuân thủ tiêu chuẩn HTTP.

---

### 6.5. Response chuẩn hóa lỗi (ErrorResponse)

- Để đảm bảo nhất quán và dễ xử lý lỗi phía client, nên định nghĩa cấu trúc response lỗi chuẩn.
- Thường gồm:
    - `statusCode`: Mã trạng thái HTTP.
    - `message`: Thông báo lỗi (string hoặc array).
    - `error` (tùy chọn): Tên lỗi hoặc thông tin chi tiết.
    - `timestamp` (tùy chọn): Thời điểm lỗi.
    - `path` (tùy chọn): Đường dẫn request gây lỗi.
    - `http method`
    - Các trường tùy chỉnh khác.

**Ví dụ interface ErrorResponse:**
```typescript
export interface ErrorResponse {
    statusCode: number;
    message: string | string[];
    error?: string;
    timestamp?: string;
    path?: string;
    // Các trường tùy chỉnh khác
}
```

- Trong custom Exception Filter, tạo response JSON dựa trên cấu trúc này.
- Chuẩn hóa response lỗi giúp client dễ phân tích, hiển thị thông tin lỗi nhất quán, cải thiện trải nghiệm người dùng và debug.

