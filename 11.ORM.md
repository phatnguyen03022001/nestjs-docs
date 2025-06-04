## 11. Tích hợp CSDL (TypeORM / Mongoose)

NestJS không ép buộc bạn phải sử dụng một ORM (Object-Relational Mapper) hoặc ODM (Object-Document Mapper) cụ thể. Thay vào đó, nó cung cấp hệ thống module linh hoạt để tích hợp bất kỳ thư viện nào bạn muốn. TypeORM và Mongoose là hai lựa chọn phổ biến nhất, đều có module NestJS chính thức hỗ trợ.

### 11.1. Cài đặt và cấu hình kết nối CSDL

#### Với TypeORM (CSDL quan hệ như PostgreSQL, MySQL, SQLite)

**Cài đặt:**
```bash
npm install @nestjs/typeorm typeorm pg # hoặc mysql2, sqlite3 tùy CSDL bạn dùng
npm install --save-dev @types/node
```
- `@nestjs/typeorm`: Module tích hợp TypeORM cho NestJS.
- `typeorm`: Thư viện TypeORM cốt lõi.
- `pg`: Driver cho PostgreSQL (thay bằng `mysql2` cho MySQL, `sqlite3` cho SQLite, ...).

**Cấu hình trong AppModule:**
```typescript
// src/app.module.ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { User } from './users/user.entity';

@Module({
  imports: [
    TypeOrmModule.forRoot({
      type: 'postgres', // hoặc 'mysql', 'sqlite', ...
      host: 'localhost',
      port: 5432,
      username: 'your_username',
      password: 'your_password',
      database: 'your_database',
      entities: [User],
      synchronize: true, // Chỉ dùng cho dev
      logging: true,
    }),
  ],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```
> `synchronize: true` chỉ nên dùng trong môi trường phát triển.

#### Với Mongoose (MongoDB)

**Cài đặt:**
```bash
npm install @nestjs/mongoose mongoose
```
- `@nestjs/mongoose`: Module tích hợp Mongoose cho NestJS.
- `mongoose`: Thư viện Mongoose cốt lõi.

**Cấu hình trong AppModule:**
```typescript
// src/app.module.ts
import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';
import { AppController } from './app.controller';
import { AppService } from './app.service';

@Module({
  imports: [
    MongooseModule.forRoot('mongodb://localhost:27017/your_database_name'),
  ],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

### 11.2. Tạo Entity/Schema

#### TypeORM (Entities)

```typescript
// src/users/user.entity.ts
import { Entity, PrimaryGeneratedColumn, Column } from 'typeorm';

@Entity()
export class User {
  @PrimaryGeneratedColumn()
  id: number;

  @Column({ unique: true })
  username: string;

  @Column({ default: true })
  isActive: boolean;

  @Column({ nullable: true })
  email: string;
}
```
- `@Entity()`: Định nghĩa Entity.
- `@PrimaryGeneratedColumn()`: Khóa chính tự động tăng.
- `@Column()`: Định nghĩa cột.

#### Mongoose (Schemas)

```typescript
// src/users/schemas/user.schema.ts
import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';
import { HydratedDocument } from 'mongoose';

export type UserDocument = HydratedDocument<User>;

@Schema()
export class User {
  @Prop({ required: true, unique: true })
  username: string;

  @Prop({ default: true })
  isActive: boolean;

  @Prop({ sparse: true })
  email: string;
}

export const UserSchema = SchemaFactory.createForClass(User);
```
- `@Schema()`: Định nghĩa Schema.
- `@Prop()`: Định nghĩa thuộc tính document.

### 11.3. Repository pattern trong NestJS

#### TypeORM

Inject Repository vào service với `@InjectRepository()`:

```typescript
// src/users/users.module.ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { UsersService } from './users.service';
import { UsersController } from './users.controller';
import { User } from './user.entity';

@Module({
  imports: [TypeOrmModule.forFeature([User])],
  providers: [UsersService],
  controllers: [UsersController],
  exports: [UsersService],
})
export class UsersModule {}
```

```typescript
// src/users/users.service.ts
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { User } from './user.entity';
import { CreateUserDto } from './dto/create-user.dto';

@Injectable()
export class UsersService {
  constructor(
    @InjectRepository(User)
    private usersRepository: Repository<User>,
  ) {}

  async create(createUserDto: CreateUserDto): Promise<User> {
    const newUser = this.usersRepository.create(createUserDto);
    return this.usersRepository.save(newUser);
  }

  async findAll(): Promise<User[]> {
    return this.usersRepository.find();
  }

  async findOne(id: number): Promise<User> {
    return this.usersRepository.findOne({ where: { id } });
  }
}
```

#### Mongoose

Inject Model với `@InjectModel()`:

```typescript
// src/users/users.module.ts
import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';
import { UsersService } from './users.service';
import { UsersController } from './users.controller';
import { User, UserSchema } from './schemas/user.schema';

@Module({
  imports: [MongooseModule.forFeature([{ name: User.name, schema: UserSchema }])],
  providers: [UsersService],
  controllers: [UsersController],
  exports: [UsersService],
})
export class UsersModule {}
```

```typescript
// src/users/users.service.ts
import { Injectable } from '@nestjs/common';
import { InjectModel } from '@nestjs/mongoose';
import { Model } from 'mongoose';
import { User, UserDocument } from './schemas/user.schema';
import { CreateUserDto } from './dto/create-user.dto';

@Injectable()
export class UsersService {
  constructor(
    @InjectModel(User.name)
    private userModel: Model<UserDocument>,
  ) {}

  async create(createUserDto: CreateUserDto): Promise<User> {
    const createdUser = new this.userModel(createUserDto);
    return createdUser.save();
  }

  async findAll(): Promise<User[]> {
    return this.userModel.find().exec();
  }

  async findOne(id: string): Promise<User> {
    return this.userModel.findById(id).exec();
  }
}
```

### 11.4. CRUD cơ bản

#### TypeORM

```typescript
// Create
async create(createUserDto: CreateUserDto): Promise<User> {
  const newUser = this.usersRepository.create(createUserDto);
  return this.usersRepository.save(newUser);
}

// Read
async findAll(): Promise<User[]> {
  return this.usersRepository.find();
}
async findOne(id: number): Promise<User> {
  return this.usersRepository.findOne({ where: { id } });
}
async findByUsername(username: string): Promise<User> {
  return this.usersRepository.findOne({ where: { username } });
}

// Update
async update(id: number, updateUserDto: UpdateUserDto): Promise<User> {
  await this.usersRepository.update(id, updateUserDto);
  return this.findOne(id);
}

// Delete
async remove(id: number): Promise<void> {
  await this.usersRepository.delete(id);
}
```

#### Mongoose

```typescript
// Create
async create(createUserDto: CreateUserDto): Promise<User> {
  const createdUser = new this.userModel(createUserDto);
  return createdUser.save();
}

// Read
async findAll(): Promise<User[]> {
  return this.userModel.find().exec();
}
async findOne(id: string): Promise<User> {
  return this.userModel.findById(id).exec();
}
async findByUsername(username: string): Promise<User> {
  return this.userModel.findOne({ username }).exec();
}

// Update
async update(id: string, updateUserDto: UpdateUserDto): Promise<User> {
  return this.userModel.findByIdAndUpdate(id, updateUserDto, { new: true }).exec();
}

// Delete
async remove(id: string): Promise<any> {
  return this.userModel.findByIdAndDelete(id).exec();
}
```

### 11.5. Quan hệ bảng (OneToMany, ManyToOne…)

#### TypeORM

Các decorator quan hệ:
- `@OneToOne`
- `@OneToMany`
- `@ManyToOne`
- `@ManyToMany`

**Ví dụ: User - Post**

```typescript
// src/users/user.entity.ts
import { Entity, PrimaryGeneratedColumn, Column, OneToMany } from 'typeorm';
import { Post } from '../posts/post.entity';

@Entity()
export class User {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  username: string;

  @OneToMany(() => Post, post => post.user)
  posts: Post[];
}

// src/posts/post.entity.ts
import { Entity, PrimaryGeneratedColumn, Column, ManyToOne } from 'typeorm';
import { User } from '../users/user.entity';

@Entity()
export class Post {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  title: string;

  @Column()
  content: string;

  @ManyToOne(() => User, user => user.posts)
  user: User;
}
```

**Truy vấn kèm quan hệ:**
```typescript
// trong UsersService
async findUserWithPosts(userId: number): Promise<User> {
  return this.usersRepository.findOne({
    where: { id: userId },
    relations: ['posts'],
  });
}
```

#### Mongoose

Không có quan hệ bảng truyền thống, nhưng có thể:
- **Nhúng tài liệu (Embedded Documents):**

```typescript
@Schema()
class Address {
  @Prop()
  street: string;
  @Prop()
  city: string;
}
const AddressSchema = SchemaFactory.createForClass(Address);

@Schema()
export class User {
  @Prop()
  username: string;

  @Prop({ type: [AddressSchema] })
  addresses: Address[];
}
```

- **Tham chiếu (References / Population):**

```typescript
// src/users/schemas/user.schema.ts
import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';
import { HydratedDocument, Types } from 'mongoose';

export type UserDocument = HydratedDocument<User>;

@Schema()
export class User {
  @Prop({ required: true, unique: true })
  username: string;

  @Prop({ type: [{ type: Types.ObjectId, ref: 'Post' }] })
  posts: Types.ObjectId[];
}

export const UserSchema = SchemaFactory.createForClass(User);

// src/posts/schemas/post.schema.ts
import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';
import { HydratedDocument, Types } from 'mongoose';

export type PostDocument = HydratedDocument<Post>;

@Schema()
export class Post {
  @Prop({ required: true })
  title: string;

  @Prop()
  content: string;

  @Prop({ type: Types.ObjectId, ref: 'User', required: true })
  user: Types.ObjectId;
}

export const PostSchema = SchemaFactory.createForClass(Post);
```

**Truy vấn kèm populate:**
```typescript
// trong UsersService (Mongoose)
async findUserWithPosts(userId: string): Promise<User> {
  return this.userModel
    .findById(userId)
    .populate('posts')
    .exec();
}
```

---

Việc lựa chọn TypeORM hay Mongoose phụ thuộc vào loại CSDL bạn sử dụng (quan hệ hay NoSQL) và yêu cầu dự án. NestJS hỗ trợ tốt cả hai.
