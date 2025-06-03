# 8. Guards – Bảo vệ route

Guards (hay còn gọi là "người gác cổng") là thành phần quan trọng trong kiến trúc ứng dụng web hiện đại, đặc biệt là trong NestJS, dùng để kiểm soát quyền truy cập đến các route handler. Chúng hoạt động như một lớp bảo vệ, được thực thi trước khi request được xử lý bởi handler chính.

## 8.1. Guard là gì?

Guards là các class được chú thích với `@Injectable()` và triển khai interface `CanActivate`. Chúng có phương thức `canActivate()` trả về boolean, Promise<boolean> hoặc Observable<boolean>.

- Nếu `canActivate()` trả về `true`, request sẽ tiếp tục đến handler.
- Nếu trả về `false`, request bị chặn và NestJS trả về lỗi `ForbiddenException` (HTTP 403) nếu không có ngoại lệ khác.

**Mục đích chính của Guards:**

- **Xác thực quyền truy cập (Authorization):** Xác định xem người dùng có quyền truy cập tài nguyên/hành động cụ thể không. Khác với xác thực danh tính (Authentication).
- **Kiểm tra điều kiện cụ thể:** Ví dụ, kiểm tra vai trò quản trị viên, sở hữu tài nguyên, hoặc đủ số dư giao dịch.

## 8.2. Tạo custom guard (implements CanActivate)

Tạo custom guard bằng cách implement interface `CanActivate`:

```typescript
// src/common/guards/auth.guard.ts
import { Injectable, CanActivate, ExecutionContext, UnauthorizedException } from '@nestjs/common';
import { Observable } from 'rxjs';

@Injectable()
export class AuthGuard implements CanActivate {
    canActivate(context: ExecutionContext): boolean | Promise<boolean> | Observable<boolean> {
        const request = context.switchToHttp().getRequest();
        const authHeader = request.headers.authorization;

        if (!authHeader || !authHeader.startsWith('Bearer ')) {
            throw new UnauthorizedException('Authentication token required');
        }

        const token = authHeader.split(' ')[1];
        // Thực tế sẽ giải mã và xác thực JWT ở đây
        if (token === 'mysecrettoken') {
            request.user = { id: 1, roles: ['admin'] };
            return true;
        }

        throw new UnauthorizedException('Invalid token');
    }
}
```

**Giải thích:**

- `@Injectable()`: Đánh dấu class là provider.
- `canActivate(context: ExecutionContext)`: Phương thức kiểm tra quyền truy cập.
- `context`: Cung cấp thông tin về request, response, handler.

## 8.3. ExecutionContext và Reflector (metadata access)

### ExecutionContext

- Mở rộng từ `ArgumentsHost`.
- Truy cập thông tin ngữ cảnh thực thi (HTTP, WebSockets, gRPC).

**Các phương thức quan trọng:**

- `getType()`: Loại ứng dụng ('http', 'ws', 'rpc').
- `getClass()`: Class controller.
- `getHandler()`: Method handler.
- `switchToHttp()`: Truy cập request, response HTTP.

**Ví dụ:**

```typescript
const request = context.switchToHttp().getRequest();
const response = context.switchToHttp().getResponse();
const controllerClass = context.getClass();
const handlerMethod = context.getHandler();
```

### Reflector (metadata access)

- Helper class để đọc metadata từ decorator tùy chỉnh.

**Tạo custom decorator:**

```typescript
// src/common/decorators/roles.decorator.ts
import { SetMetadata } from '@nestjs/common';

export const ROLES_KEY = 'roles';
export const Roles = (...roles: string[]) => SetMetadata(ROLES_KEY, roles);
```

**Sử dụng decorator trên controller/route:**

```typescript
// src/users/users.controller.ts
import { Controller, Post, Get, UseGuards } from '@nestjs/common';
import { Roles } from 'src/common/decorators/roles.decorator';
import { AuthGuard } from 'src/common/guards/auth.guard';
import { RolesGuard } from 'src/common/guards/roles.guard';

@Controller('users')
@UseGuards(AuthGuard, RolesGuard)
export class UsersController {
    @Post()
    @Roles('admin')
    create() {
        return 'User created (Admin only)';
    }

    @Get()
    @Roles('admin', 'user')
    findAll() {
        return 'List of users';
    }
}
```

**Đọc metadata trong Guard với Reflector:**

```typescript
// src/common/guards/roles.guard.ts
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { ROLES_KEY } from 'src/common/decorators/roles.decorator';

@Injectable()
export class RolesGuard implements CanActivate {
    constructor(private reflector: Reflector) {}

    canActivate(context: ExecutionContext): boolean {
        const requiredRoles = this.reflector.getAllAndOverride<string[]>(ROLES_KEY, [
            context.getHandler(),
            context.getClass(),
        ]);

        if (!requiredRoles) {
            return true;
        }

        const { user } = context.switchToHttp().getRequest();
        if (!user || !user.roles) {
            return false;
        }

        return requiredRoles.some((role) => user.roles.includes(role));
    }
}
```

## 8.4. Áp dụng guard cho route, controller, global

Guards có thể áp dụng ở ba cấp độ (thứ tự thực thi: global → controller → method):

### Cấp độ phương thức/route

```typescript
@Get('profile')
@UseGuards(AuthGuard)
getProfile() {
    return 'User profile data';
}
```

### Cấp độ controller

```typescript
@Controller('admin')
@UseGuards(AuthGuard, RolesGuard)
export class AdminController {
    // ...
}
```

### Cấp độ toàn cục (Global-scoped)

```typescript
// main.ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { AuthGuard } from './common/guards/auth.guard';

async function bootstrap() {
    const app = await NestFactory.create(AppModule);
    app.useGlobalGuards(new AuthGuard());
    await app.listen(3000);
}
bootstrap();
```

**Hoặc qua module:**

```typescript
// app.module.ts
import { Module } from '@nestjs/common';
import { APP_GUARD } from '@nestjs/core';
import { AuthGuard } from './common/guards/auth.guard';

@Module({
    providers: [
        {
            provide: APP_GUARD,
            useClass: AuthGuard,
        },
    ],
})
export class AppModule {}
```

## 8.5. Role-based access control (RBAC)

RBAC quản lý quyền truy cập dựa trên vai trò. Trong NestJS, triển khai RBAC bằng cách:

- Custom Decorator (`@Roles()`): Đính kèm vai trò yêu cầu cho route.
- Custom Guard (`RolesGuard`): Đọc vai trò từ decorator và so sánh với vai trò user.

Ví dụ đã trình bày ở phần trên.

## 8.6. Kết hợp với JWT/Passport

Guards thường kết hợp với JWT hoặc Passport.js để xây dựng hệ thống bảo mật:

**Xác thực danh tính (Authentication):**

- Khi đăng nhập, server trả về JWT.
- Client gửi JWT trong header Authorization cho các request tiếp theo.
- NestJS dùng passport-jwt hoặc custom strategy để xác thực JWT.
- Nếu hợp lệ, user object được gán vào request.

**Ủy quyền (Authorization):**

- Sau xác thực, các Guards (như `RolesGuard`) kiểm tra vai trò user.
- Sử dụng Reflector để đọc vai trò yêu cầu từ decorator.
- So sánh vai trò user với vai trò yêu cầu để quyết định quyền truy cập.

**Luồng hoạt động:**

1. Client gửi request với JWT.
2. `AuthGuard` kiểm tra và giải mã JWT, gán user vào request.
3. `RolesGuard` đọc vai trò từ decorator, kiểm tra với user.
4. Nếu hợp lệ, request tiếp tục đến handler.

Sự kết hợp giữa JWT/Passport cho xác thực và Guards (đặc biệt với Reflector cho RBAC) tạo nên hệ thống bảo mật mạnh mẽ, rõ ràng trong NestJS.

