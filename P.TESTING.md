# 14. Testing trong NestJS

NestJS được thiết kế với khả năng kiểm thử cao, giúp bạn dễ dàng viết các loại test để đảm bảo chất lượng code. Mặc định, NestJS sử dụng Jest cho kiểm thử và có thể kết hợp Supertest để kiểm thử tích hợp (E2E).

## 14.1. Unit Test với Jest

Unit Test tập trung kiểm thử các thành phần nhỏ nhất, độc lập của ứng dụng như hàm, phương thức hoặc class riêng lẻ mà không phụ thuộc vào các thành phần bên ngoài (database, HTTP request).

**Các thành phần chính của Jest:**

- `describe()`: Nhóm các test case.
- `it()` hoặc `test()`: Định nghĩa một test case.
- `expect()`: Thực hiện các khẳng định (assertions).
- **Matchers**: Các phương thức đi kèm với `expect()` (ví dụ: `toBe`, `toEqual`, `toHaveBeenCalledWith`).

**Ví dụ cấu trúc file test với Jest:**

```typescript
// my-feature.service.spec.ts
describe('MyFeatureService', () => {
  let service: MyFeatureService;

  beforeEach(() => {
    service = new MyFeatureService();
  });

  it('should be defined', () => {
    expect(service).toBeDefined();
  });

  it('should return "Hello World!"', () => {
    expect(service.getHello()).toBe('Hello World!');
  });
});
```

## 14.2. Test Controller, Service

### Kiểm thử Service (Unit Test)

Service chứa logic nghiệp vụ thuần túy nên kiểm thử unit khá đơn giản.

```typescript
// src/users/users.service.ts
import { Injectable } from '@nestjs/common';
import { User } from './entities/user.entity';

@Injectable()
export class UsersService {
  private users: User[] = [];
  private nextId = 1;

  create(name: string): User {
    const newUser = { id: this.nextId++, name };
    this.users.push(newUser);
    return newUser;
  }

  findAll(): User[] {
    return this.users;
  }

  findOne(id: number): User | undefined {
    return this.users.find(user => user.id === id);
  }
}
```

```typescript
// src/users/users.service.spec.ts
import { UsersService } from './users.service';

describe('UsersService', () => {
  let service: UsersService;

  beforeEach(() => {
    service = new UsersService();
  });

  it('should be defined', () => {
    expect(service).toBeDefined();
  });

  it('should create a new user', () => {
    const user = service.create('Alice');
    expect(user).toEqual({ id: 1, name: 'Alice' });
    expect(service.findAll()).toHaveLength(1);
  });

  it('should find all users', () => {
    service.create('Bob');
    service.create('Charlie');
    expect(service.findAll()).toHaveLength(2);
  });

  it('should find one user by id', () => {
    service.create('David');
    const foundUser = service.findOne(1);
    expect(foundUser).toEqual({ id: 1, name: 'David' });
  });

  it('should return undefined if user not found', () => {
    expect(service.findOne(999)).toBeUndefined();
  });
});
```

### Kiểm thử Controller (Integration Test / Unit Test với Mocking)

Kiểm thử controller kiểm tra cách controller xử lý HTTP request và tương tác với service phụ thuộc.

```typescript
// src/users/users.controller.ts
import { Controller, Get, Post, Body, Param, ParseIntPipe } from '@nestjs/common';
import { UsersService } from './users.service';

@Controller('users')
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Post()
  create(@Body('name') name: string) {
    return this.usersService.create(name);
  }

  @Get()
  findAll() {
    return this.usersService.findAll();
  }

  @Get(':id')
  findOne(@Param('id', ParseIntPipe) id: number) {
    return this.usersService.findOne(id);
  }
}
```

```typescript
// src/users/users.controller.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { UsersController } from './users.controller';
import { UsersService } from './users.service';

describe('UsersController', () => {
  let controller: UsersController;
  let service: UsersService;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      controllers: [UsersController],
      providers: [UsersService],
    }).compile();

    controller = module.get<UsersController>(UsersController);
    service = module.get<UsersService>(UsersService);
  });

  it('should be defined', () => {
    expect(controller).toBeDefined();
  });

  it('should return an array of users', () => {
    const result = [{ id: 1, name: 'Test User' }];
    jest.spyOn(service, 'findAll').mockImplementation(() => result);

    expect(controller.findAll()).toBe(result);
  });

  it('should create a user', () => {
    const newUser = { id: 1, name: 'New User' };
    jest.spyOn(service, 'create').mockReturnValue(newUser);

    expect(controller.create('New User')).toEqual(newUser);
  });

  it('should find one user by id', () => {
    const user = { id: 1, name: 'Found User' };
    jest.spyOn(service, 'findOne').mockReturnValue(user);

    expect(controller.findOne(1)).toEqual(user);
  });
});
```

## 14.3. Mocking Providers

Khi kiểm thử một thành phần, bạn thường muốn "mock" các phụ thuộc để test nhanh, độc lập và dễ tái tạo.

**Các cách mocking trong Jest/NestJS:**

- `jest.fn()`: Tạo hàm mock.
- `jest.spyOn()`: Giả lập một phương thức cụ thể.
- `mockImplementation()` / `mockReturnValue()`: Định nghĩa hành vi mock.
- `Test.createTestingModule().overrideProvider().useValue()`: Thay thế provider bằng mock.

**Ví dụ sử dụng overrideProvider để mock Service:**

```typescript
// src/users/users.controller.spec.ts (mocking nâng cao)
import { Test, TestingModule } from '@nestjs/testing';
import { UsersController } from './users.controller';
import { UsersService } from './users.service';

describe('UsersController', () => {
  let controller: UsersController;
  const mockUsersService = {
    create: jest.fn(name => ({ id: 1, name })),
    findAll: jest.fn(() => [{ id: 1, name: 'Mocked User' }]),
    findOne: jest.fn(id => (id === 1 ? { id: 1, name: 'Mocked User' } : undefined)),
  };

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      controllers: [UsersController],
      providers: [
        {
          provide: UsersService,
          useValue: mockUsersService,
        },
      ],
    }).compile();

    controller = module.get<UsersController>(UsersController);
  });

  it('should be defined', () => {
    expect(controller).toBeDefined();
  });

  it('should call findAll on the service', () => {
    controller.findAll();
    expect(mockUsersService.findAll).toHaveBeenCalled();
  });

  it('should return mocked users', () => {
    expect(controller.findAll()).toEqual([{ id: 1, name: 'Mocked User' }]);
  });

  it('should call create on the service and return new user', () => {
    const newUser = controller.create('Test Name');
    expect(mockUsersService.create).toHaveBeenCalledWith('Test Name');
    expect(newUser).toEqual({ id: 1, name: 'Test Name' });
  });
});
```

## 14.4. E2E Test với SuperTest

E2E Test mô phỏng hành vi người dùng trên toàn hệ thống, kiểm tra luồng request từ client qua controller, service, database và trả về response.

NestJS khuyến nghị dùng Supertest với Jest cho E2E testing.

**Cài đặt:**

```bash
npm install --save-dev supertest @types/supertest
```

**Ví dụ E2E test cho ứng dụng NestJS:**

```typescript
// src/app.e2e-spec.ts
import * as request from 'supertest';
import { Test, TestingModule } from '@nestjs/testing';
import { INestApplication } from '@nestjs/common';
import { AppModule } from './app.module';

describe('AppController (e2e)', () => {
  let app: INestApplication;

  beforeEach(async () => {
    const moduleFixture: TestingModule = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    app = moduleFixture.createNestApplication();
    await app.init();
  });

  it('/ (GET)', () => {
    return request(app.getHttpServer())
      .get('/')
      .expect(200)
      .expect('Hello World!');
  });

  afterAll(async () => {
    await app.close();
  });
});
```

**Ví dụ E2E cho /users endpoint:**

```typescript
// src/users/users.e2e-spec.ts
import * as request from 'supertest';
import { Test, TestingModule } from '@nestjs/testing';
import { INestApplication } from '@nestjs/common';
import { AppModule } from '../app.module';
import { TypeOrmModule } from '@nestjs/typeorm';
import { User } from './entities/user.entity';

describe('UsersController (e2e)', () => {
  let app: INestApplication;

  beforeEach(async () => {
    const moduleFixture: TestingModule = await Test.createTestingModule({
      imports: [
        AppModule,
        TypeOrmModule.forRoot({
          type: 'sqlite',
          database: ':memory:',
          entities: [User],
          synchronize: true,
        }),
      ],
    }).compile();

    app = moduleFixture.createNestApplication();
    await app.init();
  });

  afterAll(async () => {
    await app.close();
  });

  it('/users (POST) should create a user', () => {
    return request(app.getHttpServer())
      .post('/users')
      .send({ name: 'Alice' })
      .expect(201)
      .expect(res => {
        expect(res.body).toHaveProperty('id');
        expect(res.body.name).toBe('Alice');
      });
  });

  it('/users (GET) should return all users', async () => {
    await request(app.getHttpServer())
      .post('/users')
      .send({ name: 'Bob' })
      .expect(201);

    return request(app.getHttpServer())
      .get('/users')
      .expect(200)
      .expect(res => {
        expect(res.body).toHaveLength(1);
        expect(res.body[0].name).toBe('Bob');
      });
  });
});
```

**Lưu ý khi chạy E2E tests:**

- **Database Testing:** Nên dùng database riêng biệt cho test (ví dụ: SQLite in-memory hoặc Docker container tạm thời).
- **Cleanup:** Dọn dẹp dữ liệu sau mỗi test hoặc test suite để tránh ảnh hưởng lẫn nhau.

---

Kiểm thử là phần không thể thiếu trong phát triển phần mềm hiện đại. Với NestJS, bạn có đầy đủ công cụ để viết unit test và E2E test toàn diện, giúp đảm bảo chất lượng ứng dụng.
