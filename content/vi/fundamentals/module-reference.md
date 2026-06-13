### Module reference

Nest cung cấp class `ModuleRef` để điều hướng danh sách providers nội bộ và obtain một tham chiếu đến bất kỳ provider nào sử dụng injection token của nó như một lookup key. Class `ModuleRef` cũng cung cấp một cách để dynamically instantiate cả static và scoped providers. `ModuleRef` có thể được inject vào một class theo cách bình thường:

```typescript
@@filename(cats.service)
@Injectable()
export class CatsService {
  constructor(private moduleRef: ModuleRef) {}
}
@@switch
@Injectable()
@Dependencies(ModuleRef)
export class CatsService {
  constructor(moduleRef) {
    this.moduleRef = moduleRef;
  }
}
```

> info **Gợi ý** Class `ModuleRef` được import từ package `@nestjs/core`.

#### Retrieving instances

Instance `ModuleRef` (sau đây chúng ta sẽ gọi nó là **module reference**) có một method `get()`. Theo mặc định, method này trả về một provider, controller, hoặc injectable (ví dụ, guard, interceptor, v.v.) đã được đăng ký và đã được instantiated trong *module hiện tại* sử dụng injection token/class name của nó. Nếu instance không được tìm thấy, một exception sẽ được raised.

```typescript
@@filename(cats.service)
@Injectable()
export class CatsService implements OnModuleInit {
  private service: Service;
  constructor(private moduleRef: ModuleRef) {}

  onModuleInit() {
    this.service = this.moduleRef.get(Service);
  }
}
@@switch
@Injectable()
@Dependencies(ModuleRef)
export class CatsService {
  constructor(moduleRef) {
    this.moduleRef = moduleRef;
  }

  onModuleInit() {
    this.service = this.moduleRef.get(Service);
  }
}
```

> warning **Cảnh báo** Bạn không thể retrieve scoped providers (transient hoặc request-scoped) với method `get()`. Thay vào đó, sử dụng kỹ thuật được mô tả ở dưới. Tìm hiểu cách kiểm soát scopes ở đây.

Để retrieve một provider từ global context (ví dụ, nếu provider đã được inject trong một module khác), truyền option `strict: false` như một đối số thứ hai đến `get()`.

```typescript
this.moduleRef.get(Service, { strict: false });
```

#### Resolving scoped providers

Để dynamically resolve một scoped provider (transient hoặc request-scoped), sử dụng method `resolve()`, truyền injection token của provider như một đối số.

```typescript
@@filename(cats.service)
@Injectable()
export class CatsService implements OnModuleInit {
  private transientService: TransientService;
  constructor(private moduleRef: ModuleRef) {}

  async onModuleInit() {
    this.transientService = await this.moduleRef.resolve(TransientService);
  }
}
@@switch
@Injectable()
@Dependencies(ModuleRef)
export class CatsService {
  constructor(moduleRef) {
    this.moduleRef = moduleRef;
  }

  async onModuleInit() {
    this.transientService = await this.moduleRef.resolve(TransientService);
  }
}
```

Method `resolve()` trả về một instance duy nhất của provider, từ **DI container sub-tree** của chính nó. Mỗi sub-tree có một **context identifier** duy nhất. Do đó, nếu bạn gọi method này nhiều hơn một lần và so sánh các tham chiếu instance, bạn sẽ thấy rằng chúng không bằng nhau.

```typescript
@@filename(cats.service)
@Injectable()
export class CatsService implements OnModuleInit {
  constructor(private moduleRef: ModuleRef) {}

  async onModuleInit() {
    const transientServices = await Promise.all([
      this.moduleRef.resolve(TransientService),
      this.moduleRef.resolve(TransientService),
    ]);
    console.log(transientServices[0] === transientServices[1]); // false
  }
}
@@switch
@Injectable()
@Dependencies(ModuleRef)
export class CatsService {
  constructor(moduleRef) {
    this.moduleRef = moduleRef;
  }

  async onModuleInit() {
    const transientServices = await Promise.all([
      this.moduleRef.resolve(TransientService),
      this.moduleRef.resolve(TransientService),
    ]);
    console.log(transientServices[0] === transientServices[1]); // false
  }
}
```

Để tạo một instance duy nhất trên nhiều cuộc gọi `resolve()`, và đảm bảo chúng chia sẻ cùng một DI container sub-tree được tạo ra, bạn có thể truyền một context identifier đến method `resolve()`. Sử dụng class `ContextIdFactory` để tạo một context identifier. Class này cung cấp một method `create()` trả về một identifier duy nhất phù hợp.

```typescript
@@filename(cats.service)
@Injectable()
export class CatsService implements OnModuleInit {
  constructor(private moduleRef: ModuleRef) {}

  async onModuleInit() {
    const contextId = ContextIdFactory.create();
    const transientServices = await Promise.all([
      this.moduleRef.resolve(TransientService, contextId),
      this.moduleRef.resolve(TransientService, contextId),
    ]);
    console.log(transientServices[0] === transientServices[1]); // true
  }
}
@@switch
@Injectable()
@Dependencies(ModuleRef)
export class CatsService {
  constructor(moduleRef) {
    this.moduleRef = moduleRef;
  }

  async onModuleInit() {
    const contextId = ContextIdFactory.create();
    const transientServices = await Promise.all([
      this.moduleRef.resolve(TransientService, contextId),
      this.moduleRef.resolve(TransientService, contextId),
    ]);
    console.log(transientServices[0] === transientServices[1]); // true
  }
}
```

> info **Gợi ý** Class `ContextIdFactory` được import từ package `@nestjs/core`.

#### Registering `REQUEST` provider

Các context identifiers được tạo thủ công (với `ContextIdFactory.create()`) đại diện cho DI sub-trees trong đó provider `REQUEST` là `undefined` vì chúng không được instantiated và quản lý bởi hệ thống dependency injection của Nest.

Để đăng ký một đối tượng `REQUEST` tùy chỉnh cho một DI sub-tree được tạo thủ công, sử dụng method `ModuleRef#registerRequestByContextId()`, như sau:

```typescript
const contextId = ContextIdFactory.create();
this.moduleRef.registerRequestByContextId(/* YOUR_REQUEST_OBJECT */, contextId);
```

#### Getting current sub-tree

Đôi khi, bạn có thể muốn resolve một instance của một provider request-scoped trong một **request context**. Hãy nói rằng `CatsService` là request-scoped và bạn muốn resolve instance `CatsRepository` cũng được đánh dấu như một provider request-scoped. Để chia sẻ cùng một DI container sub-tree, bạn phải obtain context identifier hiện tại thay vì tạo một cái mới (ví dụ, với function `ContextIdFactory.create()`, như được hiển thị ở trên). Để obtain context identifier hiện tại, bắt đầu bằng cách inject đối tượng request sử dụng decorator `@Inject()`.

```typescript
@@filename(cats.service)
@Injectable()
export class CatsService {
  constructor(
    @Inject(REQUEST) private request: Record<string, unknown>,
  ) {}
}
@@switch
@Injectable()
@Dependencies(REQUEST)
export class CatsService {
  constructor(request) {
    this.request = request;
  }
}
```

> info **Gợi ý** Tìm hiểu thêm về request provider ở đây.

Bây giờ, sử dụng method `getByRequest()` của class `ContextIdFactory` để tạo một context id dựa trên đối tượng request, và truyền điều này đến cuộc gọi `resolve()`:

```typescript
const contextId = ContextIdFactory.getByRequest(this.request);
const catsRepository = await this.moduleRef.resolve(CatsRepository, contextId);
```

#### Instantiating custom classes dynamically

Để dynamically instantiate một class mà **không được đăng ký trước đó** như một **provider**, sử dụng method `create()` của module reference.

```typescript
@@filename(cats.service)
@Injectable()
export class CatsService implements OnModuleInit {
  private catsFactory: CatsFactory;
  constructor(private moduleRef: ModuleRef) {}

  async onModuleInit() {
    this.catsFactory = await this.moduleRef.create(CatsFactory);
  }
}
@@switch
@Injectable()
@Dependencies(ModuleRef)
export class CatsService {
  constructor(moduleRef) {
    this.moduleRef = moduleRef;
  }

  async onModuleInit() {
    this.catsFactory = await this.moduleRef.create(CatsFactory);
  }
}
```

Kỹ thuật này cho phép bạn conditionally instantiate các classes khác nhau bên ngoài container của framework.

<app-banner-devtools></app-banner-devtools>