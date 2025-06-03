# 3. CONTROLLER

## 3.1. Controller là gì?
- Trong **NestJS**, Controller chịu trách nhiệm xử lý các request gửi đến từ client.
- Mục đích chính là nhận request cụ thể cho ứng dụng và sau đó ủy quyền việc xử lý cho các **providers (services)**.
- Controller về cơ bản là một **class** được đánh dấu bằng decorator `@Controller()`.
- Decorator này có thể nhận một **tiền tố (prefix)** đường dẫn, giúp nhóm các route liên quan lại với nhau.

---

## 3.2. Các route decorators: `@Get`, `@Post`, `@Put`, `@Delete`, `@Patch`

- Các decorator này được sử dụng để ánh xạ các **HTTP request methods** đến các method cụ thể trong controller.

#### Chi tiết:
- `@Get(path?: string)`:  
  Xử lý các request HTTP GET. Nếu không có `path`, nó sẽ ánh xạ đến **root path** của controller.

- `@Post(path?: string)`:  
  Xử lý các request HTTP POST, thường dùng để **tạo tài nguyên mới**.

- `@Put(path?: string)`:  
  Xử lý các request HTTP PUT, thường dùng để **cập nhật toàn bộ tài nguyên**.

- `@Delete(path?: string)`:  
  Xử lý các request HTTP DELETE, thường dùng để **xóa tài nguyên**.

- `@Patch(path?: string)`:  
  Xử lý các request HTTP PATCH, thường dùng để **cập nhật một phần tài nguyên**.

> **Lưu ý:** `path` là một chuỗi **tùy chọn** định nghĩa đường dẫn cụ thể cho route này.  
> Ví dụ: `@Get('users')`

---

## 3.3. Truy cập dữ liệu request: `@Param`, `@Query`, `@Body`, `@Headers`

NestJS cung cấp các decorator để dễ dàng truy cập dữ liệu từ request:

- `@Param(key?: string)`:  
  Lấy tham số từ **đường dẫn (route parameters)**.  
  - Nếu không có `key`, trả về một **object chứa tất cả các tham số**.  
  - Ví dụ:  
    ```ts
    @Get('users/:id')
    getUser(@Param('id') id: string)
    ```

- `@Query(key?: string)`:  
  Lấy tham số từ **query string của URL**.  
  - Nếu không có `key`, trả về object chứa tất cả query parameters.  
  - Ví dụ:  
    ```ts
    @Get('products')
    getProducts(@Query('category') category: string)
    ```

- `@Body(key?: string)`:  
  Lấy dữ liệu từ phần **body của request** (thường trong các request POST, PUT, PATCH).  
  - Nếu không có `key`, trả về toàn bộ body.  
  - Ví dụ:  
    ```ts
    @Post('users')
    createUser(@Body() userData: CreateUserDto)
    ```

- `@Headers(name?: string)`:  
  Lấy giá trị của một **header cụ thể** trong request.  
  - Nếu không có `name`, trả về object chứa tất cả các headers.  
  - Ví dụ:  
    ```ts
    @Get('profile')
    getProfile(@Headers('Authorization') token: string)
    ```

---


## 3.4. Trả response & custom status code

- Các method trong controller thường trả về một giá trị JavaScript (object, array, primitive).  
  NestJS sẽ **tự động chuyển đổi nó thành JSON** và gửi lại cho client với **status code mặc định là `200 OK`** (hoặc `201 Created` cho POST request thành công).

- Để **tùy chỉnh status code**, bạn có thể sử dụng decorator `@HttpCode(statusCode)` trên method của controller.  
  Ví dụ:
  ```ts
  @Post('users')
  @HttpCode(HttpStatus.CREATED)
  createUser(...) { ... }
  ```


  @Get()
  getData(@Res() res: Response) {
    res.status(200).json({ message: 'OK' });
  }

- Bạn cũng có thể sử dụng đối tượng `Response` của Express (được inject thông qua `@Res()`) để kiểm soát response một cách chi tiết hơn, bao gồm cả việc thiết lập headers, status code, và gửi dữ liệu. Tuy nhiên, cách này thường không được khuyến khích trong NestJS vì nó bỏ qua một số tính năng của framework.
- NestJS cung cấp `HttpStatus` enum từ `@nestjs/common` để dễ dàng sử dụng các mã trạng thái HTTP tiêu chuẩn.

## 3.5. Route grouping và versioning
   - Route grouping (tiền tố đường dẫn): Sử dụng decorator `@Controller('prefix')` trên class controller để nhóm tất cả các route bên trong controller với một tiền tố chung. Ví dụ, `@Controller('api/users')` sẽ khiến tất cả các route trong controller này bắt đầu bằng `/api/users`.
   - Route versioning: NestJS hỗ trợ versioning API để quản lý các phiên bản khác nhau của API. Có nhiều cách để thực hiện versioning:
     + URI Versioning: Bao gồm số phiên bản trong URL (ví dụ: `/v1/users`, `/v2/users`). NestJS cung cấp cấu hình `version` trong `main.ts` hoặc decorator `@Version('v1')` trên controller hoặc route handler.
     + Header Versioning: Xác định phiên bản thông qua một header (ví dụ: `X-API-Version: 1`). Cấu hình `header` trong tùy chọn versioning.
     + Media Type (Accept Header) Versioning: Dựa vào `Accept` header trong request. Cấu hình `mediaType` trong tùy chọn versioning.
     + NestJS cho phép cấu hình versioning toàn cầu hoặc trên từng controller/route. Bạn có thể chỉ định nhiều phiên bản cho một route handler bằng cách truyền một mảng vào decorator `@Version(['1', '2'])`.

