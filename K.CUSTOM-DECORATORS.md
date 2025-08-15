## 10. Custom Decorators – Tái sử dụng logic

Decorators là một tính năng của TypeScript (đang ở Stage 3 ECMAScript proposal) cho phép thêm siêu dữ liệu (metadata) hoặc hành vi vào class, method, property hoặc parameter. Decorator được biểu diễn bằng ký hiệu `@` trước tên decorator.

Trong NestJS, custom decorators giúp trích xuất và tái sử dụng logic liên quan đến request/response, xác thực, ủy quyền, hoặc dữ liệu ngữ cảnh khác.

### 10.1. Decorator trong TypeScript

Decorator là một hàm được gọi tại thời điểm định nghĩa class, method, property hoặc parameter. Có 5 loại decorator:

- **Class Decorators**: Áp dụng cho class constructor.

    ```typescript
    function sealed(constructor: Function) {
      Object.seal(constructor);
      Object.seal(constructor.prototype);
    }

    @sealed
    class Greeter {
      greeting: string;
      constructor(message: string) {
        this.greeting = message;
      }
      greet() {
        return "Hello, " + this.greeting;
      }
    }
    ```

- **Method Decorators**: Áp dụng cho method.

    ```typescript
    function enumerable(value: boolean) {
      return function (target: any, propertyKey: string, descriptor: PropertyDescriptor) {
        descriptor.enumerable = value;
      };
    }

    class Greeter {
      greeting: string;
      constructor(message: string) { this.greeting = message; }

      @enumerable(false)
      greet() {
        return "Hello, " + this.greeting;
      }
    }
    ```

- **Property Decorators**: Áp dụng cho property.

    ```typescript
    function format(formatString: string) {
      return function (target: any, propertyKey: string) {
        let value = target[propertyKey];
        const getter = function () {
          return formatString.replace('%s', value);
        };
        const setter = function (newVal: string) {
          value = newVal;
        };
        Object.defineProperty(target, propertyKey, {
          get: getter,
          set: setter,
          enumerable: true,
          configurable: true,
        });
      };
    }

    class Greeter {
      @format('Hello, %s')
      name: string;
      constructor(name: string) {
        this.name = name;
      }
    }
    ```

- **Parameter Decorators**: Áp dụng cho parameter của method.

    ```typescript
    function required(target: any, propertyKey: string, parameterIndex: number) {
      console.log(`Parameter at index ${parameterIndex} of method ${propertyKey} is required.`);
    }

    class Greeter {
      greet(@required message: string) {
        console.log(message);
      }
    }
    ```

- **Accessor Decorators**: Áp dụng cho getter/setter (ít phổ biến trong NestJS).

---

### 10.2. Tạo custom parameter decorator (`@User()`, `@Lang()`)

Parameter decorator thường dùng trong NestJS để trích xuất dữ liệu từ request object và truyền vào handler.

#### `@User()` decorator

- **Mục đích**: Lấy thông tin user từ request (thường được gắn bởi AuthGuard).
- **Cách tạo**:

    ```typescript
    // src/common/decorators/user.decorator.ts
    import { createParamDecorator, ExecutionContext } from '@nestjs/common';

    export const User = createParamDecorator(
      (data: string, ctx: ExecutionContext) => {
        const request = ctx.switchToHttp().getRequest();
        const user = request.user;
        return data ? user?.[data] : user;
      },
    );
    ```

- **Cách sử dụng**:

    ```typescript
    // src/users/users.controller.ts
    import { Controller, Get, UseGuards } from '@nestjs/common';
    import { AuthGuard } from 'src/common/guards/auth.guard';
    import { User } from 'src/common/decorators/user.decorator';

    interface UserPayload {
      id: number;
      username: string;
      roles: string[];
    }

    @Controller('profile')
    @UseGuards(AuthGuard)
    export class ProfileController {
      @Get()
      getProfile(@User() user: UserPayload) {
        return `Hello, ${user.username}! Your ID is ${user.id}`;
      }

      @Get('roles')
      getRoles(@User('roles') roles: string[]) {
        return `Your roles: ${roles.join(', ')}`;
      }
    }
    ```

#### `@Lang()` decorator

- **Mục đích**: Lấy ngôn ngữ từ query `lang` hoặc header `Accept-Language`.
- **Cách tạo**:

    ```typescript
    // src/common/decorators/lang.decorator.ts
    import { createParamDecorator, ExecutionContext } from '@nestjs/common';

    export const Lang = createParamDecorator(
      (data: string, ctx: ExecutionContext) => {
        const request = ctx.switchToHttp().getRequest();
        const lang = request.query.lang || request.headers['accept-language']?.split(',')[0] || 'en';
        return lang;
      },
    );
    ```

- **Cách sử dụng**:

    ```typescript
    // src/app.controller.ts
    import { Controller, Get } from '@nestjs/common';
    import { Lang } from './common/decorators/lang.decorator';

    @Controller()
    export class AppController {
      @Get('hello')
      getHello(@Lang() lang: string) {
        if (lang === 'vi') {
          return 'Xin chào!';
        }
        return 'Hello!';
      }
    }
    ```

---

### 10.3. Tạo custom method/class decorators (`@Roles()`)

Method và class decorators giúp thêm metadata vào class/method để các thành phần khác như Guards hoặc Interceptors có thể đọc và xử lý.

#### `@Roles()` decorator

- **Mục đích**: Đánh dấu route/controller yêu cầu vai trò cụ thể.
- **Cách tạo**:

    ```typescript
    // src/common/decorators/roles.decorator.ts
    import { SetMetadata } from '@nestjs/common';

    export const ROLES_KEY = 'roles';
    export const Roles = (...roles: string[]) => SetMetadata(ROLES_KEY, roles);
    ```

- **Cách sử dụng**:

    ```typescript
    // src/users/users.controller.ts
    import { Controller, Post, UseGuards } from '@nestjs/common';
    import { Roles } from 'src/common/decorators/roles.decorator';
    import { AuthGuard } from 'src/common/guards/auth.guard';
    import { RolesGuard } from 'src/common/guards/roles.guard';

    @Controller('users')
    @UseGuards(AuthGuard, RolesGuard)
    export class UsersController {
      @Post()
      @Roles('admin')
      create() {
        return 'Create user (Admin only)';
      }
    }
    ```

Guard (`RolesGuard`) sẽ dùng Reflector để đọc metadata này.

---

### 10.4. Kết hợp với Guards & Interceptors

Custom decorators mạnh mẽ khi kết hợp với Guards và Interceptors:

- **Với Guards**: Decorator như `@Roles()` cung cấp metadata, Guard đọc metadata này để xác thực quyền truy cập.
- **Với Interceptors**: Decorator có thể đánh dấu method cần cache hoặc định dạng response đặc biệt.

#### Ví dụ `@Cacheable()`

- **Decorator**:

    ```typescript
    // src/common/decorators/cacheable.decorator.ts
    import { SetMetadata } from '@nestjs/common';

    export const CACHE_KEY = 'cacheable';
    export const Cacheable = (ttl?: number) => SetMetadata(CACHE_KEY, ttl || true);
    ```

- **Interceptor**:

    ```typescript
    // src/common/interceptors/cache.interceptor.ts
    import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
    import { Observable, of } from 'rxjs';
    import { tap } from 'rxjs/operators';
    import { Reflector } from '@nestjs/core';
    import { CACHE_KEY } from '../decorators/cacheable.decorator';

    @Injectable()
    export class CacheInterceptor implements NestInterceptor {
      private readonly cache = new Map<string, any>();
      constructor(private reflector: Reflector) {}

      intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
        const cacheOptions = this.reflector.get<number | boolean>(CACHE_KEY, context.getHandler());
        if (!cacheOptions) {
          return next.handle();
        }
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
            // Có thể thêm logic TTL dựa vào cacheOptions ở đây
          }),
        );
      }
    }
    ```

- **Sử dụng**:

    ```typescript
    // src/products/products.controller.ts
    import { Controller, Get, UseInterceptors } from '@nestjs/common';
    import { Cacheable } from 'src/common/decorators/cacheable.decorator';
    import { CacheInterceptor } from 'src/common/interceptors/cache.interceptor';

    @Controller('products')
    @UseInterceptors(CacheInterceptor)
    export class ProductsController {
      @Get()
      @Cacheable(60000)
      findAll() {
        return 'List of all products (cached)';
      }
    }
    ```

---

### Lợi ích của Custom Decorators

- **Tái sử dụng code**: Gom logic chung vào decorator, dùng lại nhiều nơi.
- **Khai báo rõ ràng**: Code dễ đọc, thể hiện ý định rõ ràng tại điểm sử dụng.
- **Tách biệt mối quan tâm**: Controller/service tập trung vào nghiệp vụ, còn xác thực, logging, caching... để decorator + guard/interceptor xử lý.
- **Giảm boilerplate**: Giảm code lặp lại.
