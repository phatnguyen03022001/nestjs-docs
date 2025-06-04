## 1. 🔐 Xác thực và Phân quyền

Trong phát triển ứng dụng web, **xác thực** (Authentication) là quá trình kiểm tra danh tính người dùng (bạn là ai?), còn **phân quyền** (Authorization) là xác định quyền truy cập vào tài nguyên hoặc hành động cụ thể (bạn được phép làm gì?). NestJS cung cấp các công cụ mạnh mẽ để quản lý cả hai.

### 1.1. Passport.js trong NestJS

[Passport.js](http://www.passportjs.org/) là middleware xác thực linh hoạt cho Node.js, sử dụng hệ thống "chiến lược" (strategies). NestJS tích hợp Passport qua module `@nestjs/passport`.

**Cách hoạt động:**

1. Cấu hình một hoặc nhiều chiến lược xác thực (ví dụ: Local, JWT).
2. Nếu xác thực thành công, Passport gắn thông tin người dùng vào `req.user`.
3. Nếu thất bại, trả về lỗi xác thực.

**Cài đặt:**

```bash
npm install @nestjs/passport passport passport-local @types/passport-local
# Nếu dùng JWT:
npm install passport-jwt @nestjs/jwt @types/passport-jwt
```

---

### 1.2. JWT Authentication

JWT (JSON Web Token) là chuẩn mở (RFC 7519) để tạo token an toàn, đại diện cho việc truyền thông tin giữa các bên dưới dạng JSON. Token này được ký điện tử, đảm bảo tính toàn vẹn và không thể bị giả mạo. JWT rất phổ biến cho xác thực không trạng thái (stateless authentication) trong API RESTful.

**Cấu trúc JWT:**

- **Header:** Loại token (JWT) và thuật toán mã hóa (HS256, RS256).
- **Payload:** Chứa các "claims" (ví dụ: userId, username, roles, exp, iat). Không nên đặt thông tin nhạy cảm vì chỉ mã hóa Base64.
- **Signature:** Kết hợp Header, Payload và "secret key" để xác minh token.

**Luồng JWT Authentication:**

1. **Đăng nhập:** Người dùng gửi username/password.
2. **Server:** Xác thực, tạo JWT và trả về cho client.
3. **Client:** Lưu JWT (localStorage, sessionStorage hoặc cookie).
4. **Các request tiếp theo:** Gửi JWT trong header Authorization (Bearer \<token\>).
5. **Server:** Dùng JWT Strategy để xác minh token và gắn thông tin vào `req.user`.

---

### 1.3. Local Strategy vs JWT Strategy

Cả hai là các chiến lược của Passport, phục vụ các mục đích khác nhau.

#### Local Strategy (Username/Password)

- **Mục đích:** Xác thực người dùng dựa trên username/password, thường ở bước đăng nhập.
- **Cách hoạt động:** Định nghĩa class `LocalStrategy`, kiểm tra thông tin đăng nhập với DB, trả về user nếu hợp lệ.

```typescript
// src/auth/local.strategy.ts
import { Strategy } from 'passport-local';
import { PassportStrategy } from '@nestjs/passport';
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { AuthService } from './auth.service';

@Injectable()
export class LocalStrategy extends PassportStrategy(Strategy) {
    constructor(private authService: AuthService) {
        super();
    }
    async validate(username: string, password: string): Promise<any> {
        const user = await this.authService.validateUser(username, password);
        if (!user) throw new UnauthorizedException();
        return user;
    }
}
```

```typescript
// src/auth/auth.controller.ts
import { Controller, Post, Request, UseGuards } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';
import { AuthService } from './auth.service';

@Controller('auth')
export class AuthController {
    constructor(private authService: AuthService) {}

    @UseGuards(AuthGuard('local'))
    @Post('login')
    async login(@Request() req) {
        return this.authService.login(req.user); // Trả về JWT
    }
}
```

#### JWT Strategy (Token)

- **Mục đích:** Xác thực người dùng cho các request tiếp theo dựa trên JWT.
- **Cách hoạt động:** Định nghĩa class `JwtStrategy`, trích xuất JWT từ header, giải mã và xác minh token.

```typescript
// src/auth/jwt.strategy.ts
import { ExtractJwt, Strategy } from 'passport-jwt';
import { PassportStrategy } from '@nestjs/passport';
import { Injectable } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
    constructor(private configService: ConfigService) {
        super({
            jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
            ignoreExpiration: false,
            secretOrKey: configService.get<string>('JWT_SECRET'),
        });
    }
    async validate(payload: any) {
        return { userId: payload.sub, username: payload.username, roles: payload.roles };
    }
}
```

```typescript
// src/users/users.controller.ts
import { Controller, Get, UseGuards, Request } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Controller('users')
export class UsersController {
    @UseGuards(AuthGuard('jwt'))
    @Get('profile')
    getProfile(@Request() req) {
        return req.user;
    }
}
```

---

### 1.4. Refresh Token & Bảo mật nâng cao

Chỉ dùng một JWT duy nhất có thể gây rủi ro bảo mật và trải nghiệm người dùng kém. **Refresh Token** là giải pháp.

**Luồng Refresh Token:**

1. **Đăng nhập:** Server trả về Access Token (JWT, sống ngắn) và Refresh Token (sống dài hơn, lưu trong cookie HttpOnly).
2. **Sử dụng Access Token:** Client gửi Access Token cho các API.
3. **Access Token hết hạn:** Client nhận lỗi xác thực.
4. **Yêu cầu Refresh Token:** Client gửi Refresh Token đến endpoint đặc biệt (ví dụ: `/auth/refresh`).
5. **Server:** Xác minh Refresh Token, tạo Access Token mới (và có thể Refresh Token mới), trả về cho client. Nếu Refresh Token không hợp lệ, yêu cầu đăng nhập lại.

**Lợi ích:**

- **Bảo mật:** Access Token sống ngắn, có thể thu hồi Refresh Token để vô hiệu hóa truy cập.
- **Trải nghiệm:** Người dùng không phải đăng nhập lại thường xuyên.

**Các biện pháp bảo mật nâng cao:**

- Lưu Refresh Token an toàn (cookie HttpOnly).
- Thu hồi Refresh Token (lưu trong DB, xóa khi cần).
- Danh sách đen JWT (blacklist) khi đăng xuất.
- Luôn dùng HTTPS.
- Hash mật khẩu với bcrypt.
- Rate limiting các endpoint xác thực.

---

Việc triển khai xác thực và phân quyền là rất quan trọng. Kết hợp Passport.js, JWT, Refresh Token và các biện pháp bảo mật nâng cao giúp xây dựng hệ thống an toàn, mạnh mẽ với NestJS.
