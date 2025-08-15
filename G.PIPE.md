## 7. Pipes – Xử lý dữ liệu đầu vào

Pipes là một khái niệm mạnh mẽ trong các framework web hiện đại như NestJS, dùng để xử lý và biến đổi dữ liệu đầu vào. Chúng hoạt động như các bộ lọc hoặc bộ chuyển đổi, được áp dụng tuần tự lên dữ liệu trước khi đến các handler (ví dụ: controller methods).

### 7.1. Pipes là gì? Khi nào dùng?

**Pipes** là các class được chú thích với `@Injectable()` và triển khai interface `PipeTransform`. Chúng có phương thức `transform(value, metadata)` để:

- **Biến đổi**: Chuyển đổi dữ liệu đầu vào sang định dạng mong muốn (ví dụ: chuỗi số thành số nguyên).
- **Xác thực**: Kiểm tra dữ liệu đầu vào hợp lệ, nếu không sẽ ném ngoại lệ (thường là `BadRequestException`).

**Khi nào dùng Pipes?**

- Chuyển đổi kiểu dữ liệu (ví dụ: từ chuỗi sang số nguyên).
- Xác thực dữ liệu đầu vào.
- Thiết lập giá trị mặc định cho tham số.
- Tái sử dụng logic xử lý dữ liệu.
- Tách biệt xử lý dữ liệu khỏi logic nghiệp vụ trong controller.

### 7.2. Các built-in pipes (ValidationPipe, ParseIntPipe, DefaultValuePipe)

NestJS cung cấp nhiều built-in pipes hữu ích:

#### ValidationPipe

- **Mục đích**: Xác thực toàn bộ payload dựa trên class-validator và class-transformer.
- **Cách dùng**: Áp dụng toàn cục, controller hoặc method.

```typescript
// main.ts (global validation pipe)
import { ValidationPipe } from '@nestjs/common';
async function bootstrap() {
    const app = await NestFactory.create(AppModule);
    app.useGlobalPipes(new ValidationPipe());
    await app.listen(3000);
}
```

```typescript
// users.controller.ts
import { Body, Controller, Post } from '@nestjs/common';
import { CreateUserDto } from './dto/create-user.dto';

@Controller('users')
export class UsersController {
    @Post()
    create(@Body() createUserDto: CreateUserDto) {
        return createUserDto;
    }
}
```

#### ParseIntPipe

- **Mục đích**: Chuyển chuỗi thành số nguyên, ném lỗi nếu không hợp lệ.
- **Cách dùng**: Áp dụng ở tham số.

```typescript
import { Controller, Get, Param, ParseIntPipe } from '@nestjs/common';

@Controller('products')
export class ProductsController {
    @Get(':id')
    findOne(@Param('id', ParseIntPipe) id: number) {
        return `Product with ID: ${id}`;
    }
}
```

#### DefaultValuePipe

- **Mục đích**: Cung cấp giá trị mặc định nếu tham số là `undefined` hoặc `null`.
- **Cách dùng**: Áp dụng ở tham số.

```typescript
import { Controller, Get, Query, DefaultValuePipe } from '@nestjs/common';

@Controller('items')
export class ItemsController {
    @Get()
    findAll(
        @Query('limit', new DefaultValuePipe(10)) limit: number,
        @Query('offset', new DefaultValuePipe(0)) offset: number,
    ) {
        return `Fetching items with limit: ${limit}, offset: ${offset}`;
    }
}
```

### 7.3. Tạo custom pipe (implements PipeTransform)

Tạo custom pipe bằng cách implement interface `PipeTransform<T, R>`:

```typescript
// src/common/pipes/uppercase.pipe.ts
import { PipeTransform, Injectable, ArgumentMetadata, BadRequestException } from '@nestjs/common';

@Injectable()
export class UppercasePipe implements PipeTransform {
    transform(value: string, metadata: ArgumentMetadata) {
        if (typeof value !== 'string') {
            throw new BadRequestException('Input must be a string');
        }
        return value.toUpperCase();
    }
}
```

```typescript
// src/app.controller.ts
import { Controller, Get, Query } from '@nestjs/common';
import { UppercasePipe } from './common/pipes/uppercase.pipe';

@Controller()
export class AppController {
    @Get('greet')
    greet(@Query('name', UppercasePipe) name: string) {
        return `HELLO, ${name}!`;
    }
}
```

- `UppercasePipe` biến đổi chuỗi thành chữ in hoa, ném lỗi nếu không phải chuỗi.
- `ArgumentMetadata` cung cấp thông tin về đối số pipe xử lý.

### 7.4. Áp dụng pipe ở đâu: parameter, controller, global

Pipes có thể áp dụng ở ba cấp độ:

- **Parameter-scoped**: Áp dụng cho tham số cụ thể.
    ```typescript
    @Get(':id')
    findOne(@Param('id', ParseIntPipe) id: number) { ... }
    ```
- **Controller-scoped**: Áp dụng cho toàn bộ controller.
    ```typescript
    import { Controller, UsePipes, ValidationPipe } from '@nestjs/common';

    @Controller('users')
    @UsePipes(ValidationPipe)
    export class UsersController { ... }
    ```
- **Global-scoped**: Áp dụng cho toàn bộ ứng dụng (thường trong `main.ts`).
    ```typescript
    import { NestFactory } from '@nestjs/core';
    import { AppModule } from './app.module';
    import { ValidationPipe } from '@nestjs/common';

    async function bootstrap() {
        const app = await NestFactory.create(AppModule);
        app.useGlobalPipes(new ValidationPipe());
        await app.listen(3000);
    }
    bootstrap();
    ```

**Thứ tự thực thi**: parameter → controller → global.

### 7.5. Validation bằng class-validator & class-transformer

Kết hợp `ValidationPipe` với hai thư viện:

- **class-validator**: Định nghĩa quy tắc xác thực bằng decorator.
- **class-transformer**: Chuyển đổi plain object thành instance của class.

**Cài đặt:**

```bash
npm install class-validator class-transformer
```

**Định nghĩa DTO:**

```typescript
// src/users/dto/create-user.dto.ts
import { IsString, IsEmail, IsInt, MinLength, MaxLength, IsOptional, IsBoolean } from 'class-validator';
import { Type } from 'class-transformer';

export class CreateUserDto {
    @IsString()
    @MinLength(3)
    @MaxLength(20)
    name: string;

    @IsEmail()
    email: string;

    @IsInt()
    @Type(() => Number)
    age: number;

    @IsOptional()
    @IsBoolean()
    @Type(() => Boolean)
    isActive?: boolean;
}
```

**Áp dụng ValidationPipe:**

```typescript
// main.ts
import { ValidationPipe } from '@nestjs/common';
async function bootstrap() {
    const app = await NestFactory.create(AppModule);
    app.useGlobalPipes(new ValidationPipe({
        transform: true,
        whitelist: true,
        forbidNonWhitelisted: true,
    }));
    await app.listen(3000);
}
```

```typescript
// src/users/users.controller.ts
import { Body, Controller, Post } from '@nestjs/common';
import { CreateUserDto } from './dto/create-user.dto';

@Controller('users')
export class UsersController {
    @Post()
    create(@Body() createUserDto: CreateUserDto) {
        return createUserDto;
    }
}
```

**Giải thích options:**

- `transform: true`: Tự động chuyển đổi plain object thành instance của DTO.
- `whitelist: true`: Loại bỏ thuộc tính không định nghĩa trong DTO.
- `forbidNonWhitelisted: true`: Ném lỗi nếu có thuộc tính không định nghĩa.

**Lợi ích:**

- Mã nguồn rõ ràng, dễ đọc.
- Tái sử dụng DTO.
- Tự động hóa xác thực và chuyển đổi.
- Xử lý lỗi tự động.
- Đảm bảo type safety cho controller methods.

