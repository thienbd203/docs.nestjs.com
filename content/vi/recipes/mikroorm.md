### MikroORM

Công thức này ở đây để giúp người dùng bắt đầu với MikroORM trong Nest. MikroORM là ORM TypeScript cho Node.js dựa trên các pattern Data Mapper, Unit of Work và Identity Map. Nó là một sự thay thế tuyệt vời cho TypeORM và việc di chuyển từ TypeORM nên tương đối dễ dàng. Tài liệu hoàn chỉnh về MikroORM có thể được tìm thấy [ở đây](https://mikro-orm.io/docs).

> info **info** `@mikro-orm/nestjs` là một gói bên thứ ba và không được quản lý bởi nhóm cốt lõi NestJS. Vui lòng báo cáo bất kỳ vấn đề nào được tìm thấy với thư viện trong [repository thích hợp](https://github.com/mikro-orm/nestjs).

#### Cài đặt

Cách dễ nhất để tích hợp MikroORM vào Nest là thông qua [`@mikro-orm/nestjs` module](https://github.com/mikro-orm/nestjs).
Chỉ cần cài đặt nó bên cạnh Nest, MikroORM và driver cơ bản:

```bash
$ npm i @mikro-orm/core @mikro-orm/nestjs @mikro-orm/sqlite
```

MikroORM cũng hỗ trợ `postgres`, `sqlite`, và `mongo`. Xem [tài liệu chính thức](https://mikro-orm.io/docs/usage-with-sql/) để biết tất cả các driver.

Khi quá trình cài đặt hoàn tất, chúng ta có thể nhập `MikroOrmModule` vào `AppModule` gốc.

```typescript
import { SqliteDriver } from '@mikro-orm/sqlite';

@Module({
  imports: [
    MikroOrmModule.forRoot({
      entities: ['./dist/entities'],
      entitiesTs: ['./src/entities'],
      dbName: 'my-db-name.sqlite3',
      driver: SqliteDriver,
    }),
  ],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

Phương thức `forRoot()` chấp nhận cùng đối tượng cấu hình như `init()` từ gói MikroORM. Kiểm tra [trang này](https://mikro-orm.io/docs/configuration) để biết tài liệu cấu hình hoàn chỉnh.

Ngoài ra chúng ta có thể [cấu hình CLI](https://mikro-orm.io/docs/installation#setting-up-the-commandline-tool) bằng cách tạo một file cấu hình `mikro-orm.config.ts` và sau đó gọi `forRoot()` mà không có bất kỳ đối số nào.

```typescript
@Module({
  imports: [
    MikroOrmModule.forRoot(),
  ],
  ...
})
export class AppModule {}
```

Nhưng điều này sẽ không hoạt động khi bạn sử dụng các công cụ build sử dụng tree shaking, vì vậy tốt hơn là cung cấp cấu hình rõ ràng:

```typescript
import config from './mikro-orm.config'; // cấu hình ORM của bạn

@Module({
  imports: [
    MikroOrmModule.forRoot(config),
  ],
  ...
})
export class AppModule {}
```

Sau đó, `EntityManager` sẽ có sẵn để inject trên toàn bộ dự án (mà không cần nhập bất kỳ module nào ở nơi khác).

```ts
// Import mọi thứ từ gói driver của bạn hoặc `@mikro-orm/knex`
import { EntityManager, MikroORM } from '@mikro-orm/sqlite';

@Injectable()
export class MyService {
  constructor(
    private readonly orm: MikroORM,
    private readonly em: EntityManager,
  ) {}
}
```

> info **info** Lưu ý rằng `EntityManager` được nhập từ gói `@mikro-orm/driver`, trong đó driver là `mysql`, `sqlite`, `postgres` hoặc driver nào bạn đang sử dụng. Trong trường hợp bạn có `@mikro-orm/knex` được cài đặt như một phụ thuộc, bạn cũng có thể nhập `EntityManager` từ đó.

#### Repositories

MikroORM hỗ trợ pattern thiết kế repository. Đối với mỗi thực thể, chúng ta có thể tạo một repository. Đọc tài liệu hoàn chỉnh về repositories [ở đây](https://mikro-orm.io/docs/repositories). Để định nghĩa các repositories nào nên được đăng ký trong phạm vi hiện tại bạn có thể sử dụng phương thức `forFeature()`. Ví dụ, theo cách này:

> info **info** Bạn **không nên** đăng ký các thực thể cơ bản của bạn qua `forFeature()`, vì không có
> repositories cho những cái đó. Mặt khác, các thực thể cơ bản cần là một phần của danh sách trong `forRoot()` (hoặc trong cấu hình ORM nói chung).

```typescript
// photo.module.ts
@Module({
  imports: [MikroOrmModule.forFeature([Photo])],
  providers: [PhotoService],
  controllers: [PhotoController],
})
export class PhotoModule {}
```

và nhập nó vào `AppModule` gốc:

```typescript
// app.module.ts
@Module({
  imports: [MikroOrmModule.forRoot(...), PhotoModule],
})
export class AppModule {}
```

Theo cách này chúng ta có thể inject `PhotoRepository` vào `PhotoService` sử dụng decorator `@InjectRepository()`:

```typescript
@Injectable()
export class PhotoService {
  constructor(
    @InjectRepository(Photo)
    private readonly photoRepository: EntityRepository<Photo>,
  ) {}
}
```

#### Sử dụng repositories tùy chỉnh

Khi sử dụng repositories tùy chỉnh, chúng ta không còn cần decorator `@InjectRepository()`
, vì Nest DI giải quyết dựa trên các tham chiếu class.

```ts
// `**./author.entity.ts**`
@Entity({ repository: () => AuthorRepository })
export class Author {
  // để cho phép suy luận trong `em.getRepository()`
  [EntityRepositoryType]?: AuthorRepository;
}

// `**./author.repository.ts**`
export class AuthorRepository extends EntityRepository<Author> {
  // các phương thức tùy chỉnh của bạn...
}
```

Vì tên repository tùy chỉnh giống như những gì `getRepositoryToken()` sẽ
trả về, chúng ta không còn cần decorator `@InjectRepository()` nữa:

```ts
@Injectable()
export class MyService {
  constructor(private readonly repo: AuthorRepository) {}
}
```

#### Tải thực thể tự động

Thêm thủ công các thực thể vào mảng thực thể của tùy chọn kết nối có thể
tẻ nhạt. Ngoài ra, tham chiếu các thực thể từ module gốc phá vỡ các ranh giới
domain ứng dụng và gây rò rỉ chi tiết triển khai sang các phần khác của
ứng dụng. Để giải quyết vấn đề này, các đường dẫn glob tĩnh có thể được sử dụng.

Lưu ý, tuy nhiên, rằng các đường dẫn glob không được hỗ trợ bởi webpack, vì vậy nếu bạn đang xây dựng
ứng dụng của bạn trong một monorepo, bạn sẽ không thể sử dụng chúng. Để giải quyết vấn đề này, một giải pháp thay thế được cung cấp. Để tự động tải các thực thể, đặt
thuộc tính `autoLoadEntities` của đối tượng cấu hình (được truyền vào phương thức `forRoot()`
) thành `true`, như được hiển thị dưới đây:

```ts
@Module({
  imports: [
    MikroOrmModule.forRoot({
      ...
      autoLoadEntities: true,
    }),
  ],
})
export class AppModule {}
```

Với tùy chọn được chỉ định đó, mỗi thực thể được đăng ký thông qua phương thức `forFeature()`
sẽ được tự động thêm vào mảng thực thể của đối tượng cấu hình.

> info **info** Lưu ý rằng các thực thể không được đăng ký thông qua phương thức `forFeature()`, nhưng
> chỉ được tham chiếu từ thực thể (thông qua một quan hệ), sẽ không được bao gồm
> theo cách thiết lập `autoLoadEntities`.

> info **info** Sử dụng `autoLoadEntities` cũng không có hiệu quả trên CLI MikroORM - vì vậy chúng ta
> vẫn cần cấu hình CLI với danh sách đầy đủ các thực thể. Mặt khác, chúng ta có thể
> sử dụng globs ở đó, vì CLI sẽ không đi qua webpack.

#### Serialization

> warning **Lưu ý** MikroORM bọc mọi quan hệ thực thể đơn lẻ trong một đối tượng `Reference<T>` hoặc `Collection<T>`, để cung cấp type-safety tốt hơn. Điều này sẽ làm cho [serializer tích hợp của Nest](/techniques/serialization) mù đối với bất kỳ quan hệ được bọc nào. Nói cách khác, nếu bạn trả về các thực thể MikroORM từ các handler HTTP hoặc WebSocket của bạn, tất cả các quan hệ của chúng sẽ KHÔNG được serialized.

May mắn thay, MikroORM cung cấp một [API serialization](https://mikro-orm.io/docs/serializing) có thể được sử dụng thay cho `ClassSerializerInterceptor`.

```typescript
@Entity()
export class Book {
  @Property({ hidden: true }) // Tương đương với `@Exclude` của class-transformer
  hiddenField = Date.now();

  @Property({ persist: false }) // Tương tự với `@Expose()` của class-transformer. Sẽ chỉ tồn tại trong bộ nhớ, và sẽ được serialized.
  count?: number;

  @ManyToOne({
    serializer: (value) => value.name,
    serializedName: 'authorName',
  }) // Tương đương với `@Transform()` của class-transformer
  author: Author;
}
```

#### Handler phạm vi yêu cầu trong hàng đợi

Như được đề cập trong [tài liệu](https://mikro-orm.io/docs/identity-map), chúng ta cần một trạng thái sạch cho mỗi yêu cầu. Điều đó được xử lý tự động nhờ helper `RequestContext` được đăng ký thông qua middleware.

Nhưng middlewares chỉ được thực thi cho các xử lý yêu cầu HTTP thường xuyên, điều gì xảy ra nếu chúng ta cần
một phương thức phạm vi yêu cầu bên ngoài điều đó? Một ví dụ của điều đó là các handler hàng đợi hoặc
các nhiệm vụ được lên lịch.

Chúng ta có thể sử dụng decorator `@CreateRequestContext()`. Nó yêu cầu bạn trước tiên inject
instance `MikroORM` vào ngữ cảnh hiện tại, nó sẽ sau đó được sử dụng để tạo ngữ cảnh
cho bạn. Bên dưới, decorator sẽ đăng ký ngữ cảnh yêu cầu mới cho
phương thức của bạn và thực thi nó trong ngữ cảnh.

```ts
@Injectable()
export class MyService {
  constructor(private readonly orm: MikroORM) {}

  @CreateRequestContext()
  async doSomething() {
    // điều này sẽ được thực thi trong một ngữ cảnh riêng biệt
  }
}
```

> warning **Lưu ý** Như tên gợi ý, decorator này luôn tạo ngữ cảnh mới, ngược lại với thay thế `@EnsureRequestContext` của nó chỉ tạo nó nếu nó chưa ở trong một ngữ cảnh khác.

#### Kiểm tra

Gói `@mikro-orm/nestjs` expose hàm `getRepositoryToken()` trả về token đã chuẩn bị dựa trên một thực thể được cho để cho phép mock repository.

```typescript
@Module({
  providers: [
    PhotoService,
    {
      // hoặc khi bạn có một repository tùy chỉnh: `provide: PhotoRepository`
      provide: getRepositoryToken(Photo),
      useValue: mockedRepository,
    },
  ],
})
export class PhotoModule {}
```

#### Ví dụ

Một ví dụ thực tế của NestJS với MikroORM có thể được tìm thấy [ở đây](https://github.com/mikro-orm/nestjs-realworld-example-app)