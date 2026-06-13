### Database

Nest không phụ thuộc vào cơ sở dữ liệu, cho phép bạn dễ dàng tích hợp với bất kỳ cơ sở dữ liệu SQL hoặc NoSQL nào. Bạn có một số lựa chọn có sẵn, tùy thuộc vào sở thích của bạn. Ở mức độ chung nhất, kết nối Nest với một cơ sở dữ liệu chỉ đơn giản là vấn đề tải một driver Node.js thích hợp cho cơ sở dữ liệu, giống như bạn sẽ làm với [Express](https://expressjs.com/en/guide/database-integration.html) hoặc Fastify.

Bạn cũng có thể trực tiếp sử dụng bất kỳ thư viện tích hợp cơ sở dữ liệu Node.js mục đích chung **nào** hoặc ORM, chẳng hạn như [MikroORM](https://mikro-orm.io/) (xem [công thức MikroORM](/recipes/mikroorm)), [Sequelize](https://sequelize.org/) (xem [tích hợp Sequelize](/techniques/database#sequelize-integration)), [Knex.js](https://knexjs.org/) (xem [hướng dẫn Knex.js](https://dev.to/nestjs/build-a-nestjs-module-for-knex-js-or-other-resource-based-libraries-in-5-minutes-12an)), [TypeORM](https://github.com/typeorm/typeorm), và [Prisma](https://www.github.com/prisma/prisma) (xem [công thức Prisma](/recipes/prisma)), để hoạt động ở mức độ trừu tượng cao hơn.

Để thuận tiện, Nest cung cấp tích hợp chặt chẽ với TypeORM và Sequelize out-of-the-box với các package `@nestjs/typeorm` và `@nestjs/sequelize` tương ứng, mà chúng tôi sẽ bao gồm trong chương hiện tại, và Mongoose với `@nestjs/mongoose`, được bao gồm trong [chương này](/techniques/mongodb). Các tích hợp này cung cấp các tính năng đặc thù cho NestJS, chẳng hạn như injection model/repository, khả năng kiểm thử, và cấu hình bất đồng bộ để làm cho việc truy cập cơ sở dữ liệu đã chọn của bạn dễ dàng hơn.

### Tích hợp TypeORM

Để tích hợp với cơ sở dữ liệu SQL và NoSQL, Nest cung cấp package `@nestjs/typeorm`. [TypeORM](https://github.com/typeorm/typeorm) là Object Relational Mapper (ORM) trưởng thành nhất có sẵn cho TypeScript. Vì nó được viết bằng TypeScript, nó tích hợp tốt với framework Nest.

Để bắt đầu sử dụng nó, trước tiên chúng ta cài đặt các dependency cần thiết. Trong chương này, chúng tôi sẽ chứng minh sử dụng [MySQL](https://www.mysql.com/) Relational DBMS phổ biến, nhưng TypeORM cung cấp hỗ trợ cho nhiều cơ sở dữ liệu quan hệ, chẳng hạn như PostgreSQL, Oracle, Microsoft SQL Server, SQLite, và thậm chí cả cơ sở dữ liệu NoSQL như MongoDB. Quy trình chúng tôi đi qua trong chương này sẽ giống nhau cho bất kỳ cơ sở dữ liệu nào được hỗ trợ bởi TypeORM. Bạn chỉ cần cài đặt các thư viện API client liên quan cho cơ sở dữ liệu đã chọn của bạn.

```bash
$ npm install --save @nestjs/typeorm typeorm mysql2
```

Sau khi quá trình cài đặt hoàn tất, chúng ta có thể nhập `TypeOrmModule` vào `AppModule` gốc.

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';

@Module({
  imports: [
    TypeOrmModule.forRoot({
      type: 'mysql',
      host: 'localhost',
      port: 3306,
      username: 'root',
      password: 'root',
      database: 'test',
      entities: [],
      synchronize: true,
    }),
  ],
})
export class AppModule {}
```

> warning **Cảnh báo** Đặt `synchronize: true` không nên được sử dụng trong production - nếu không bạn có thể mất dữ liệu production.

Phương thức `forRoot()` hỗ trợ tất cả các thuộc tính cấu hình được expose bởi constructor `DataSource` từ package [TypeORM](https://typeorm.io/data-source-options#common-data-source-options). Ngoài ra, có một số thuộc tính cấu hình bổ sung được mô tả dưới đây.

<table>
  <tr>
    <td><code>retryAttempts</code></td>
    <td>Số lần thử kết nối đến cơ sở dữ liệu (mặc định: <code>10</code>)</td>
  </tr>
  <tr>
    <td><code>retryDelay</code></td>
    <Độ trễ giữa các lần thử kết nối lại (ms) (mặc định: <code>3000</code>)</td>
  </tr>
  <tr>
    <td><code>autoLoadEntities</code></td>
    <td>Nếu <code>true</code>, các entities sẽ được tải tự động (mặc định: <code>false</code>)</td>
  </tr>
</table>

> info **Gợi ý** Tìm hiểu thêm về các tùy chọn data source [ở đây](https://typeorm.io/data-source-options).

Sau khi điều này được thực hiện, các đối tượng TypeORM `DataSource` và `EntityManager` sẽ có sẵn để inject trên toàn bộ dự án (không cần nhập bất kỳ module nào), ví dụ:

```typescript
@@filename(app.module)
import { DataSource } from 'typeorm';

@Module({
  imports: [TypeOrmModule.forRoot(), UsersModule],
})
export class AppModule {
  constructor(private dataSource: DataSource) {}
}
@@switch
import { DataSource } from 'typeorm';

@Dependencies(DataSource)
@Module({
  imports: [TypeOrmModule.forRoot(), UsersModule],
})
export class AppModule {
  constructor(dataSource) {
    this.dataSource = dataSource;
  }
}
```

#### Mẫu Repository

[TypeORM](https://github.com/typeorm/typeorm) hỗ trợ **mẫu thiết kế repository**, vì vậy mỗi entity có repository riêng của nó. Các repositories này có thể được thu được từ data source cơ sở dữ liệu.

Để tiếp tục ví dụ, chúng ta cần ít nhất một entity. Hãy định nghĩa entity `User`.

```typescript
@@filename(user.entity)
import { Entity, Column, PrimaryGeneratedColumn } from 'typeorm';

@Entity()
export class User {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  firstName: string;

  @Column()
  lastName: string;

  @Column({ default: true })
  isActive: boolean;
}
```

> info **Gợi ý** Tìm hiểu thêm về các entities trong [tài liệu TypeORM](https://typeorm.io/docs/entity/entities/).

File entity `User` nằm trong thư mục `users`. Thư mục này chứa tất cả các file liên quan đến `UsersModule`. Bạn có thể quyết định nơi giữ các file model của mình, tuy nhiên, chúng tôi khuyến nghị tạo chúng gần **domain** của chúng, trong thư mục module tương ứng.

Để bắt đầu sử dụng entity `User`, chúng ta cần để TypeORM biết về nó bằng cách chèn nó vào mảng `entities` trong các tùy chọn phương thức `forRoot()` của module (trừ khi bạn sử dụng đường dẫn glob tĩnh):

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { User } from './users/user.entity';

@Module({
  imports: [
    TypeOrmModule.forRoot({
      type: 'mysql',
      host: 'localhost',
      port: 3306,
      username: 'root',
      password: 'root',
      database: 'test',
      entities: [User],
      synchronize: true,
    }),
  ],
})
export class AppModule {}
```

Tiếp theo, hãy nhìn vào `UsersModule`:

```typescript
@@filename(users.module)
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { UsersService } from './users.service';
import { UsersController } from './users.controller';
import { User } from './user.entity';

@Module({
  imports: [TypeOrmModule.forFeature([User])],
  providers: [UsersService],
  controllers: [UsersController],
})
export class UsersModule {}
```

Module này sử dụng phương thức `forFeature()` để định nghĩa các repositories nào được đăng ký trong phạm vi hiện tại. Với điều này đã sẵn sàng, chúng ta có thể inject `UsersRepository` vào `UsersService` bằng cách sử dụng decorator `@InjectRepository()`:

```typescript
@@filename(users.service)
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { User } from './user.entity';

@Injectable()
export class UsersService {
  constructor(
    @InjectRepository(User)
    private usersRepository: Repository<User>,
  ) {}

  findAll(): Promise<User[]> {
    return this.usersRepository.find();
  }

  findOne(id: number): Promise<User | null> {
    return this.usersRepository.findOneBy({ id });
  }

  async remove(id: number): Promise<void> {
    await this.usersRepository.delete(id);
  }
}
@@switch
import { Injectable, Dependencies } from '@nestjs/common';
import { getRepositoryToken } from '@nestjs/typeorm';
import { User } from './user.entity';

@Injectable()
@Dependencies(getRepositoryToken(User))
export class UsersService {
  constructor(usersRepository) {
    this.usersRepository = usersRepository;
  }

  findAll() {
    return this.usersRepository.find();
  }

  findOne(id) {
    return this.usersRepository.findOneBy({ id });
  }

  async remove(id) {
    await this.usersRepository.delete(id);
  }
}
```

> warning **Lưu ý** Đừng quên nhập `UsersModule` vào `AppModule` gốc.

Nếu bạn muốn sử dụng repository bên ngoài module nhập `TypeOrmModule.forFeature`, bạn sẽ cần xuất lại các providers được tạo bởi nó.
Bạn có thể làm điều này bằng cách xuất toàn bộ module, như sau:

```typescript
@@filename(users.module)
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { User } from './user.entity';

@Module({
  imports: [TypeOrmModule.forFeature([User])],
  exports: [TypeOrmModule]
})
export class UsersModule {}
```

Bây giờ nếu chúng ta nhập `UsersModule` trong `UserHttpModule`, chúng ta có thể sử dụng `@InjectRepository(User)` trong các providers của module sau.

```typescript
@@filename(users-http.module)
import { Module } from '@nestjs/common';
import { UsersModule } from './users.module';
import { UsersService } from './users.service';
import { UsersController } from './users.controller';

@Module({
  imports: [UsersModule],
  providers: [UsersService],
  controllers: [UsersController]
})
export class UserHttpModule {}
```

#### Relations

Relations là các liên kết được thiết lập giữa hai hoặc nhiều bảng. Relations dựa trên các trường chung từ mỗi bảng, thường liên quan đến các khóa chính và khóa ngoại.

Có ba loại relations:

<table>
  <tr>
    <td><code>One-to-one</code></td>
    <td>Mỗi hàng trong bảng chính có một và chỉ một hàng liên quan trong bảng ngoại. Sử dụng decorator <code>@OneToOne()</code> để định nghĩa loại relation này.</td>
  </tr>
  <tr>
    <td><code>One-to-many / Many-to-one</code></td>
    <td>Mỗi hàng trong bảng chính có một hoặc nhiều hàng liên quan trong bảng ngoại. Sử dụng các decorator <code>@OneToMany()</code> và <code>@ManyToOne()</code> để định nghĩa loại relation này.</td>
  </tr>
  <tr>
    <td><code>Many-to-many</code></td>
    <td>Mỗi hàng trong bảng chính có nhiều hàng liên quan trong bảng ngoại, và mỗi bản ghi trong bảng ngoại có nhiều hàng liên quan trong bảng chính. Sử dụng decorator <code>@ManyToMany()</code> để định nghĩa loại relation này.</td>
  </tr>
</table>

Để định nghĩa relations trong các entities, sử dụng các **decorators** tương ứng. Ví dụ, để định nghĩa rằng mỗi `User` có thể có nhiều photos, sử dụng decorator `@OneToMany()`.

```typescript
@@filename(user.entity)
import { Entity, Column, PrimaryGeneratedColumn, OneToMany } from 'typeorm';
import { Photo } from '../photos/photo.entity';

@Entity()
export class User {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  firstName: string;

  @Column()
  lastName: string;

  @Column({ default: true })
  isActive: boolean;

  @OneToMany(type => Photo, photo => photo.user)
  photos: Photo[];
}
```

> info **Gợi ý** Để tìm hiểu thêm về relations trong TypeORM, hãy truy cập [tài liệu TypeORM](https://typeorm.io/docs/relations/relations).

#### Tự động tải entities

Thêm thủ công các entities vào mảng `entities` của các tùy chọn data source có thể là tẻ nhạt. Ngoài ra, tham chiếu các entities từ module gốc làm vỡ các ranh giới domain của ứng dụng và gây rò rỉ chi tiết triển khai sang các phần khác của ứng dụng. Để giải quyết vấn đề này, một giải pháp thay thế được cung cấp. Để tự động tải các entities, đặt thuộc tính `autoLoadEntities` của đối tượng cấu hình (được truyền vào phương thức `forRoot()`) thành `true`, như được hiển thị dưới đây:

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';

@Module({
  imports: [
    TypeOrmModule.forRoot({
      ...
      autoLoadEntities: true,
    }),
  ],
})
export class AppModule {}
```

Với tùy chọn này được chỉ định, mọi entity được đăng ký thông qua phương thức `forFeature()` sẽ tự động được thêm vào mảng `entities` của đối tượng cấu hình.

> warning **Cảnh báo** Lưu ý rằng các entities không được đăng ký thông qua phương thức `forFeature()`, nhưng chỉ được tham chiếu từ entity (thông qua một relation), sẽ không được bao gồm bằng cách cài đặt `autoLoadEntities`.

#### Tách biệt định nghĩa entity

Bạn có thể định nghĩa một entity và các cột của nó ngay trong model, sử dụng các decorator. Nhưng một số người thích định nghĩa các entities và các cột của chúng trong các file riêng biệt bằng cách sử dụng ["entity schemas"](https://typeorm.io/docs/entity/separating-entity-definition).

```typescript
import { EntitySchema } from 'typeorm';
import { User } from './user.entity';

export const UserSchema = new EntitySchema<User>({
  name: 'User',
  target: User,
  columns: {
    id: {
      type: Number,
      primary: true,
      generated: true,
    },
    firstName: {
      type: String,
    },
    lastName: {
      type: String,
    },
    isActive: {
      type: Boolean,
      default: true,
    },
  },
  relations: {
    photos: {
      type: 'one-to-many',
      target: 'Photo', // tên của PhotoSchema
    },
  },
});
```

> warning error **Cảnh báo** Nếu bạn cung cấp tùy chọn `target`, giá trị tùy chọn `name` phải giống với tên của class target.
> Nếu bạn không cung cấp `target`, bạn có thể sử dụng bất kỳ tên nào.

Nest cho phép bạn sử dụng một instance `EntitySchema` ở bất cứ nơi nào một `Entity` được mong đợi, ví dụ:

```typescript
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { UserSchema } from './user.schema';
import { UsersController } from './users.controller';
import { UsersService } from './users.service';

@Module({
  imports: [TypeOrmModule.forFeature([UserSchema])],
  providers: [UsersService],
  controllers: [UsersController],
})
export class UsersModule {}
```

#### Transactions TypeORM

Một giao dịch cơ sở dữ liệu biểu thị một đơn vị công việc được thực hiện trong một hệ thống quản lý cơ sở dữ liệu chống lại một cơ sở dữ liệu, và được xử lý theo cách nhất quán và đáng tin cậy độc lập với các giao dịch khác. Một giao dịch thường đại diện cho bất kỳ thay đổi nào trong một cơ sở dữ liệu ([tìm hiểu thêm](https://en.wikipedia.org/wiki/Database_transaction)).

Có nhiều chiến lược khác nhau để xử lý [transactions TypeORM](https://typeorm.io/docs/advanced-topics/transactions/). Chúng tôi khuyến nghị sử dụng class `QueryRunner` vì nó cung cấp kiểm soát đầy đủ over giao dịch.

Đầu tiên, chúng ta cần inject đối tượng `DataSource` vào một class theo cách thông thường:

```typescript
@Injectable()
export class UsersService {
  constructor(private dataSource: DataSource) {}
}
```

> info **Gợi ý** Class `DataSource` được nhập từ package `typeorm`.

Bây giờ, chúng ta có thể sử dụng đối tượng này để tạo một giao dịch.

```typescript
async createMany(users: User[]) {
  const queryRunner = this.dataSource.createQueryRunner();

  await queryRunner.connect();
  await queryRunner.startTransaction();
  try {
    await queryRunner.manager.save(users[0]);
    await queryRunner.manager.save(users[1]);

    await queryRunner.commitTransaction();
  } catch (err) {
    // vì chúng ta có lỗi hãy rollback các thay đổi chúng ta đã thực hiện
    await queryRunner.rollbackTransaction();
  } finally {
    // bạn cần giải phóng một queryRunner được khởi tạo thủ công
    await queryRunner.release();
  }
}
```

> info **Gợi ý** Lưu ý rằng `dataSource` chỉ được sử dụng để tạo `QueryRunner`. Tuy nhiên, để kiểm thử class này sẽ yêu cầu mock toàn bộ đối tượng `DataSource` (và expose một số phương thức). Vì vậy, chúng tôi khuyến nghị sử dụng một class factory helper (ví dụ, `QueryRunnerFactory`) và định nghĩa một interface với một tập hợp các phương thức giới hạn cần thiết để duy trì các giao dịch. Kỹ thuật này làm cho việc mock các phương thức này khá đơn giản.

<app-banner-devtools></app-banner-devtools>

Ngoài ra, bạn có thể sử dụng cách tiếp cận kiểu callback với phương thức `transaction` của đối tượng `DataSource` ([đọc thêm](https://typeorm.io/docs/advanced-topics/transactions/#creating-and-using-transactions)).

```typescript
async createMany(users: User[]) {
  await this.dataSource.transaction(async manager => {
    await manager.save(users[0]);
    await manager.save(users[1]);
  });
}
```

#### Subscribers

Với TypeORM [subscribers](https://typeorm.io/docs/advanced-topics/listeners-and-subscribers#what-is-a-subscriber), bạn có thể lắng nghe các sự kiện entity cụ thể.

```typescript
import {
  DataSource,
  EntitySubscriberInterface,
  EventSubscriber,
  InsertEvent,
} from 'typeorm';
import { User } from './user.entity';

@EventSubscriber()
export class UserSubscriber implements EntitySubscriberInterface<User> {
  constructor(dataSource: DataSource) {
    dataSource.subscribers.push(this);
  }

  listenTo() {
    return User;
  }

  beforeInsert(event: InsertEvent<User>) {
    console.log(`BEFORE USER INSERTED: `, event.entity);
  }
}
```

> error **Cảnh báo** Event subscribers không thể là [phạm vi request](/fundamentals/injection-scopes).

Bây giờ, thêm class `UserSubscriber` vào mảng `providers`:

```typescript
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { User } from './user.entity';
import { UsersController } from './users.controller';
import { UsersService } from './users.service';
import { UserSubscriber } from './user.subscriber';

@Module({
  imports: [TypeOrmModule.forFeature([User])],
  providers: [UsersService, UserSubscriber],
  controllers: [UsersController],
})
export class UsersModule {}
```

#### Migrations

[Migrations](https://typeorm.io/docs/advanced-topics/migrations/) cung cấp một cách để cập nhật tăng dần schema cơ sở dữ liệu để giữ nó đồng bộ với mô hình dữ liệu của ứng dụng trong khi bảo toàn dữ liệu hiện có trong cơ sở dữ liệu. Để tạo, chạy, và hoàn tác migrations, TypeORM cung cấp một [CLI](https://typeorm.io/docs/advanced-topics/migrations/#creating-a-new-migration) chuyên dụng.

Các lớp migration tách biệt khỏi mã nguồn ứng dụng Nest. Vòng đời của chúng được duy trì bởi CLI TypeORM. Do đó, bạn không thể tận dụng dependency injection và các tính năng đặc thù Nest khác với các migrations. Để tìm hiểu thêm về migrations, hãy làm theo hướng dẫn trong [tài liệu TypeORM](https://typeorm.io/docs/advanced-topics/migrations/).

#### Nhiều cơ sở dữ liệu

Một số dự án yêu cầu nhiều kết nối cơ sở dữ liệu. Điều này cũng có thể đạt được với module này. Để làm việc với nhiều kết nối, trước tiên hãy tạo các kết nối. Trong trường hợp này, đặt tên data source trở thành **bắt buộc**.

Giả sử bạn có một entity `Album` được lưu trữ trong cơ sở dữ liệu riêng của nó.

```typescript
const defaultOptions = {
  type: 'postgres',
  port: 5432,
  username: 'user',
  password: 'password',
  database: 'db',
  synchronize: true,
};

@Module({
  imports: [
    TypeOrmModule.forRoot({
      ...defaultOptions,
      host: 'user_db_host',
      entities: [User],
    }),
    TypeOrmModule.forRoot({
      ...defaultOptions,
      name: 'albumsConnection',
      host: 'album_db_host',
      entities: [Album],
    }),
  ],
})
export class AppModule {}
```

> warning **Lưu ý** Nếu bạn không đặt `name` cho một data source, tên của nó được đặt thành `default`. Vui lòng lưu ý rằng bạn không nên có nhiều kết nối không có tên, hoặc có cùng tên, nếu không chúng sẽ bị ghi đè.

> warning **Lưu ý** Nếu bạn đang sử dụng `TypeOrmModule.forRootAsync`, bạn phải **cũng** đặt tên data source bên ngoài `useFactory`. Ví dụ:
>
> ```typescript
> TypeOrmModule.forRootAsync({
>   name: 'albumsConnection',
>   useFactory: ...,
>   inject: ...,
> }),
> ```
>
> Xem [vấn đề này](https://github.com/nestjs/typeorm/issues/86) để biết thêm chi tiết.

Tại thời điểm này, bạn có các entities `User` và `Album` được đăng ký với data source riêng của chúng. Với cấu hình này, bạn phải nói cho phương thức `TypeOrmModule.forFeature()` và decorator `@InjectRepository()` biết data source nào nên được sử dụng. Nếu bạn không truyền tên data source nào, data source `default` được sử dụng.

```typescript
@Module({
  imports: [
    TypeOrmModule.forFeature([User]),
    TypeOrmModule.forFeature([Album], 'albumsConnection'),
  ],
})
export class AppModule {}
```