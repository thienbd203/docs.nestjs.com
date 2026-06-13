### Async Local Storage

`AsyncLocalStorage` là một [API Node.js](https://nodejs.org/api/async_context.html#class-asynclocalstorage) (dựa trên API `async_hooks`) cung cấp một cách thay thế để lan truyền trạng thái cục bộ qua ứng dụng mà không cần truyền nó rõ ràng như một tham số hàm. Nó tương tự như một bộ nhớ cục bộ luồng trong các ngôn ngữ khác.

Ý tưởng chính của Async Local Storage là chúng ta có thể _bọc_ một số lời gọi hàm với lời gọi `AsyncLocalStorage#run`. Tất cả mã được gọi trong lời gọi được bọc đều có quyền truy cập vào cùng `store`, sẽ là duy nhất cho mỗi chuỗi gọi.

Trong ngữ cảnh của NestJS, điều đó có nghĩa là nếu chúng ta có thể tìm một nơi trong vòng đời yêu cầu nơi chúng ta có thể bọc phần còn lại của mã yêu cầu, chúng ta sẽ có thể truy cập và sửa đổi trạng thái chỉ nhìn thấy bởi yêu cầu đó, có thể phục vụ như một thay thế cho các provider phạm vi REQUEST và một số giới hạn của chúng.

Ngoài ra, chúng ta có thể sử dụng ALS để lan truyền ngữ cảnh chỉ cho một phần của hệ thống (ví dụ đối tượng _transaction_) mà không cần truyền nó rõ ràng qua các service, có thể tăng sự cô lập và đóng gói.

#### Triển khai tùy chỉnh

NestJS tự nó không cung cấp bất kỳ sự trừu tượng tích hợp sẵn nào cho `AsyncLocalStorage`, vì vậy hãy đi qua cách chúng ta có thể triển khai nó cho trường hợp HTTP đơn giản nhất để hiểu rõ hơn về toàn bộ khái niệm:

> info **info** Để biết [gói chuyên dụng](recipes/async-local-storage#nestjs-cls) đã sẵn sàng, hãy tiếp tục đọc dưới đây.

1. Trước tiên, tạo một instance mới của `AsyncLocalStorage` trong một file nguồn được chia sẻ. Vì chúng ta đang sử dụng NestJS, hãy cũng biến nó thành một module với một provider tùy chỉnh.

```ts
@@filename(als.module)
@Module({
  providers: [
    {
      provide: AsyncLocalStorage,
      useValue: new AsyncLocalStorage(),
    },
  ],
  exports: [AsyncLocalStorage],
})
export class AlsModule {}
```
>  info **Gợi ý** `AsyncLocalStorage` được nhập từ `async_hooks`.

2. Chúng ta chỉ quan tâm đến HTTP, vì vậy hãy sử dụng một middleware để bọc hàm `next` với `AsyncLocalStorage#run`. Vì middleware là thứ đầu tiên mà yêu cầu gặp, điều này sẽ làm cho `store` có sẵn trong tất cả các bộ tăng cường và phần còn lại của hệ thống.

```ts
@@filename(app.module)
@Module({
  imports: [AlsModule],
  providers: [CatsService],
  controllers: [CatsController],
})
export class AppModule implements NestModule {
  constructor(
    // inject the AsyncLocalStorage in the module constructor,
    private readonly als: AsyncLocalStorage
  ) {}

  configure(consumer: MiddlewareConsumer) {
    // bind the middleware,
    consumer
      .apply((req, res, next) => {
        // populate the store with some default values
        // based on the request,
        const store = {
          userId: req.headers['x-user-id'],
        };
        // and pass the "next" function as callback
        // to the "als.run" method together with the store.
        this.als.run(store, () => next());
      })
      .forRoutes('*path');
  }
}
@@switch
@Module({
  imports: [AlsModule],
  providers: [CatsService],
  controllers: [CatsController],
})
@Dependencies(AsyncLocalStorage)
export class AppModule {
  constructor(als) {
    // inject the AsyncLocalStorage in the module constructor,
    this.als = als
  }

  configure(consumer) {
    // bind the middleware,
    consumer
      .apply((req, res, next) => {
        // populate the store with some default values
        // based on the request,
        const store = {
          userId: req.headers['x-user-id'],
        };
        // and pass the "next" function as callback
        // to the "als.run" method together with the store.
        this.als.run(store, () => next());
      })
      .forRoutes('*path');
  }
}
```

3. Bây giờ, bất cứ đâu trong vòng đời của một yêu cầu, chúng ta có thể truy cập instance store cục bộ.

```ts
@@filename(cats.service)
@Injectable()
export class CatsService {
  constructor(
    // We can inject the provided ALS instance.
    private readonly als: AsyncLocalStorage,
    private readonly catsRepository: CatsRepository,
  ) {}

  getCatForUser() {
    // The "getStore" method will always return the
    // store instance associated with the given request.
    const userId = this.als.getStore()["userId"] as number;
    return this.catsRepository.getForUser(userId);
  }
}
@@switch
@Injectable()
@Dependencies(AsyncLocalStorage, CatsRepository)
export class CatsService {
  constructor(als, catsRepository) {
    // We can inject the provided ALS instance.
    this.als = als
    this.catsRepository = catsRepository
  }

  getCatForUser() {
    // The "getStore" method will always return the
    // store instance associated with the given request.
    const userId = this.als.getStore()["userId"] as number;
    return this.catsRepository.getForUser(userId);
  }
}
```

4. Đó là tất cả. Bây giờ chúng ta có một cách để chia sẻ trạng thái liên quan đến yêu cầu mà không cần inject toàn bộ đối tượng `REQUEST`.

> warning **cảnh báo** Xin lưu ý rằng trong khi kỹ thuật này hữu ích cho nhiều trường hợp sử dụng, nó vốn làm mờ dòng chảy mã (tạo ngữ cảnh ngầm định), vì vậy hãy sử dụng nó một cách có trách nhiệm và đặc biệt tránh tạo ra các "[God objects](https://en.wikipedia.org/wiki/God_object)" theo ngữ cảnh.

### NestJS CLS

Gói [nestjs-cls](https://github.com/Papooch/nestjs-cls) cung cấp một số cải thiện DX so với việc sử dụng `AsyncLocalStorage` thuần túy (`CLS` là viết tắt của thuật ngữ _continuation-local storage_). Nó trừu tượng hóa triển khai vào một `ClsModule` cung cấp nhiều cách khác nhau để khởi tạo `store` cho các transport khác nhau (không chỉ HTTP), cũng như hỗ trợ typing mạnh.

Store sau đó có thể được truy cập với một `ClsService` có thể inject, hoặc hoàn toàn trừu tượng hóa khỏi logic kinh doanh bằng cách sử dụng [Proxy Providers](https://www.npmjs.com/package/nestjs-cls#proxy-providers).

> info **info** `nestjs-cls` là một gói bên thứ ba và không được quản lý bởi nhóm cốt lõi NestJS. Vui lòng báo cáo bất kỳ vấn đề nào được tìm thấy với thư viện trong [repository thích hợp](https://github.com/Papooch/nestjs-cls/issues).

#### Cài đặt

Ngoài một phụ thuộc ngang hàng trên các thư viện `@nestjs`, nó chỉ sử dụng API Node.js tích hợp sẵn. Cài đặt nó như bất kỳ gói nào khác.

```bash
npm i nestjs-cls
```

#### Sử dụng

Một chức năng tương tự như được mô tả [ở trên](recipes/async-local-storage#custom-implementation) có thể được triển khai sử dụng `nestjs-cls` như sau:

1. Nhập `ClsModule` trong module gốc.

```ts
@@filename(app.module)
@Module({
  imports: [
    // Register the ClsModule,
    ClsModule.forRoot({
      middleware: {
        // automatically mount the
        // ClsMiddleware for all routes
        mount: true,
        // and use the setup method to
        // provide default store values.
        setup: (cls, req) => {
          cls.set('userId', req.headers['x-user-id']);
        },
      },
    }),
  ],
  providers: [CatsService],
  controllers: [CatsController],
})
export class AppModule {}
```

2. Và sau đó có thể sử dụng `ClsService` để truy cập các giá trị store.

```ts
@@filename(cats.service)
@Injectable()
export class CatsService {
  constructor(
    // We can inject the provided ClsService instance,
    private readonly cls: ClsService,
    private readonly catsRepository: CatsRepository,
  ) {}

  getCatForUser() {
    // and use the "get" method to retrieve any stored value.
    const userId = this.cls.get('userId');
    return this.catsRepository.getForUser(userId);
  }
}
@@switch
@Injectable()
@Dependencies(AsyncLocalStorage, CatsRepository)
export class CatsService {
  constructor(cls, catsRepository) {
    // We can inject the provided ClsService instance,
    this.cls = cls
    this.catsRepository = catsRepository
  }

  getCatForUser() {
    // and use the "get" method to retrieve any stored value.
    const userId = this.cls.get('userId');
    return this.catsRepository.getForUser(userId);
  }
}
```

3. Để có typing mạnh của các giá trị store được quản lý bởi `ClsService` (và cũng nhận được gợi ý tự động của các khóa chuỗi), chúng ta có thể sử dụng một tham số kiểu tùy chọn `ClsService<MyClsStore>` khi inject nó.

```ts
export interface MyClsStore extends ClsStore {
  userId: number;
}
```

> info **gợi ý** Nó cũng có thể để cho phép gói tự động tạo một Request ID và truy cập nó sau với `cls.getId()`, hoặc để lấy toàn bộ đối tượng Request sử dụng `cls.get(CLS_REQ)`.
#### Kiểm tra

Vì `ClsService` chỉ là một provider có thể inject khác, nó có thể được hoàn toàn mock trong các bài kiểm tra đơn vị.

Tuy nhiên, trong một số bài kiểm tra tích hợp, chúng ta có thể vẫn muốn sử dụng triển khai `ClsService` thực tế. Trong trường hợp đó, chúng ta sẽ cần bọc phần mã có ý thức ngữ cảnh với một lời gọi đến `ClsService#run` hoặc `ClsService#runWith`.

```ts
describe('CatsService', () => {
  let service: CatsService
  let cls: ClsService
  const mockCatsRepository = createMock<CatsRepository>()

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      // Set up most of the testing module as we normally would.
      providers: [
        CatsService,
        {
          provide: CatsRepository
          useValue: mockCatsRepository
        }
      ],
      imports: [
        // Import the static version of ClsModule which only provides
        // the ClsService, but does not set up the store in any way.
        ClsModule
      ],
    }).compile()

    service = module.get(CatsService)

    // Also retrieve the ClsService for later use.
    cls = module.get(ClsService)
  })

  describe('getCatForUser', () => {
    it('retrieves cat based on user id', async () => {
      const expectedUserId = 42
      mocksCatsRepository.getForUser.mockImplementationOnce(
        (id) => ({ userId: id })
      )

      // Wrap the test call in the `runWith` method
      // in which we can pass hand-crafted store values.
      const cat = await cls.runWith(
        { userId: expectedUserId },
        () => service.getCatForUser()
      )

      expect(cat.userId).toEqual(expectedUserId)
    })
  })
})
```

#### Thêm thông tin

Truy cập [Trang GitHub NestJS CLS](https://github.com/Papooch/nestjs-cls) để biết tài liệu API đầy đủ và thêm ví dụ mã.