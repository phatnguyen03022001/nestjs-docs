# MODULES – TỔ CHỨC MÃ NGUỒN

## 2.1. Khái niệm Module
Trong **NestJS**, module là đơn vị tổ chức mã nguồn chính. Mỗi ứng dụng **Nest** đều có ít nhất một module gốc (**AppModule**). Modules giúp nhóm các phần liên quan (**controllers, providers, services**) lại với nhau, tăng tính tái sử dụng và dễ bảo trì.

## 2.2. @Module() Decorator
`@Module()` là một decorator dùng để đánh dấu một class là một **Nest module**. Nó nhận một metadata object bao gồm các thuộc tính như: **imports, controllers, providers, exports**.

Ví dụ:
```typescript
@Module({
  imports: [],
  controllers: [UserController],
  providers: [UserService],
  exports: [UserService],
})
export class UserModule {}
```

## 2.3. Imports, Providers, Controllers, Exports

- **imports**: Nhập các module khác mà module hiện tại cần dùng.
- **controllers**: Chứa các controller xử lý HTTP request.
- **providers**: Các service hoặc class chứa logic nghiệp vụ.
- **exports**: Khai báo những provider muốn chia sẻ với module khác khi được import.

---

## 2.4. Root Module (AppModule)

Đây là module đầu tiên trong ứng dụng NestJS. Nó đóng vai trò như là **entry point** của ứng dụng, nơi bạn định nghĩa các module khác được import vào. `AppModule` là nơi tập hợp tất cả các module khác mà ứng dụng cần sử dụng.

> Mỗi ứng dụng NestJS đều có ít nhất một **Root Module**.

## 2.5. Dynamic Modules
**Dynamic Module** là module được cấu hình động tại thời điểm import. Thường dùng khi cần truyền options/config từ bên ngoài vào module.

Ví dụ:
```typescript
@Module({})
export class ConfigModule {
  static forRoot(options: ConfigOptions): DynamicModule {
    return {
      module: ConfigModule,
      providers: [{ provide: 'CONFIG_OPTIONS', useValue: options }],
      exports: ['CONFIG_OPTIONS'],
    };
  }
}
```

## 2.6. Shared Modules
Shared Module là module dùng chung giữa nhiều module khác. Để tránh instance bị tạo lại nhiều lần, cần export provider và không đặt SharedModule trong exports nhiều lần ở các nơi khác nhau (nên import một lần ở AppModule hoặc CoreModule).

> Lưu ý: Không import SharedModule vào SharedModule khác (tránh vòng lặp).