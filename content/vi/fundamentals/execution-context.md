### Execution context

Nest cung cấp một số utility class giúp việc viết các ứng dụng hoạt động trên nhiều ngữ cảnh ứng dụng khác nhau (ví dụ, dựa trên Nest HTTP server, microservices và các ngữ cảnh ứng dụng WebSockets) trở nên dễ dàng. Các utility này cung cấp thông tin về ngữ cảnh thực thi hiện tại có thể được sử dụng để xây dựng các [guards](/guards), [filters](/exception-filters), và [interceptors](/interceptors) generic có thể hoạt động trên một bộ rộng các controllers, methods, và ngữ cảnh thực thi.

Chúng tôi bao gồm hai class như vậy trong chương này: `ArgumentsHost` và `ExecutionContext`.

#### ArgumentsHost class

Class `ArgumentsHost` cung cấp các phương thức để truy xuất các đối số đang được truyền đến một handler. Nó cho phép chọn ngữ cảnh phù hợp (ví dụ, HTTP, RPC (microservice), hoặc WebSockets) để truy xuất các đối số từ đó. Framework cung cấp một instance của `ArgumentsHost`, thường được tham chiếu như một tham số `host`, ở những nơi bạn có thể muốn truy cập nó. Ví dụ, phương thức `catch()` của một [exception filter](https://docs.nestjs.com/exception-filters#arguments-host) được gọi với một instance `ArgumentsHost`.

`ArgumentsHost` đơn giản đóng vai trò như một sự trừu tượng hóa trên các đối số của một handler. Ví dụ, đối với các ứng dụng HTTP server (khi `@nestjs/platform-express` đang được sử dụng), object `host` đóng gói mảng `[request, response, next]` của Express, trong đó `request` là object request, `response` là object response, và `next` là một hàm điều khiển chu kỳ request-response của ứng dụng. Mặt khác, đối với các ứng dụng [GraphQL](/graphql/quick-start), object `host` chứa mảng `[root, args, context, info]`.

#### Current application context

Khi xây dựng các [guards](/guards), [filters](/exception-filters), và [interceptors](/interceptors) generic có nghĩa là chạy trên nhiều ngữ cảnh ứng dụng, chúng ta cần một cách để xác định loại ứng dụng mà phương thức của chúng ta hiện đang chạy. Làm điều này với phương thức `getType()` của `ArgumentsHost`:

```typescript
if (host.getType() === 'http') {
  // do something that is only important in the context of regular HTTP requests (REST)
} else if (host.getType() === 'rpc') {
  // do something that is only important in the context of Microservice requests
} else if (host.getType<GqlContextType>() === 'graphql') {
  // do something that is only important in the context of GraphQL requests
}
```

> info **Hint** `GqlContextType` được import từ package `@nestjs/graphql`.

Với loại ứng dụng có sẵn, chúng ta có thể viết các components generic hơn, như được hiển thị bên dưới.

#### Host handler arguments

Để truy xuất mảng các đối số đang được truyền đến handler, một cách tiếp cận là sử dụng phương thức `getArgs()` của object host.

```typescript
const [req, res, next] = host.getArgs();
```

Bạn có thể lấy một đối số cụ thể theo chỉ số sử dụng phương thức `getArgByIndex()`:

```typescript
const request = host.getArgByIndex(0);
const response = host.getArgByIndex(1);
```

Trong các ví dụ này, chúng tôi truy xuất các object request và response theo chỉ số, điều này thường không được khuyến nghị vì nó gắn ứng dụng với một ngữ cảnh thực thi cụ thể. Thay vào đó, bạn có thể làm cho code của mình mạnh mẽ hơn và có thể tái sử dụng bằng cách sử dụng một trong các phương thức utility của object `host` để chuyển sang ngữ cảnh ứng dụng phù hợp cho ứng dụng của bạn. Các phương thức utility chuyển đổi ngữ cảnh được hiển thị bên dưới.

```typescript
/**
 * Switch context to RPC.
 */
switchToRpc(): RpcArgumentsHost;
/**
 * Switch context to HTTP.
 */
switchToHttp(): HttpArgumentsHost;
/**
 * Switch context to WebSockets.
 */
switchToWs(): WsArgumentsHost;
```

Hãy viết lại ví dụ trước sử dụng phương thức `switchToHttp()`. Lời gọi helper `host.switchToHttp()` trả về một object `HttpArgumentsHost` phù hợp cho ngữ cảnh ứng dụng HTTP. Object `HttpArgumentsHost` có hai phương thức hữu ích chúng ta có thể sử dụng để truy xuất các object mong muốn. Chúng tôi cũng sử dụng các type assertions của Express trong trường hợp này để trả về các object được gõ kiểu Express gốc:

```typescript
const ctx = host.switchToHttp();
const request = ctx.getRequest<Request>();
const response = ctx.getResponse<Response>();
```

Tương tự `WsArgumentsHost` và `RpcArgumentsHost` có các phương thức để trả về các object phù hợp trong các ngữ cảnh microservices và WebSockets. Dưới đây là các phương thức cho `WsArgumentsHost`:

```typescript
export interface WsArgumentsHost {
  /**
   * Returns the data object.
   */
  getData<T>(): T;
  /**
   * Returns the client object.
   */
  getClient<T>(): T;
}
```

Sau đây là các phương thức cho `RpcArgumentsHost`:

```typescript
export interface RpcArgumentsHost {
  /**
   * Returns the data object.
   */
  getData<T>(): T;

  /**
   * Returns the context object.
   */
  getContext<T>(): T;
}
```

#### ExecutionContext class

`ExecutionContext` mở rộng `ArgumentsHost`, cung cấp thêm chi tiết về quá trình thực thi hiện tại. Giống như `ArgumentsHost`, Nest cung cấp một instance của `ExecutionContext` ở những places bạn có thể cần nó, như trong phương thức `canActivate()` của một [guard](https://docs.nestjs.com/guards#execution-context) và phương thức `intercept()` của một [interceptor](https://docs.nestjs.com/interceptors#execution-context). Nó cung cấp các phương thức sau:

```typescript
export interface ExecutionContext extends ArgumentsHost {
  /**
   * Returns the type of the controller class which the current handler belongs to.
   */
  getClass<T>(): Type<T>;
  /**
   * Returns a reference to the handler (method) that will be invoked next in the
   * request pipeline.
   */
  getHandler(): Function;
}
```

Phương thức `getHandler()` trả về một tham chiếu đến handler sắp được gọi. Phương thức `getClass()` trả về type của class `Controller` mà handler cụ thể này thuộc về. Ví dụ, trong ngữ cảnh HTTP, nếu request hiện đang được xử lý là một request `POST`, được ràng buộc với phương thức `create()` trên `CatsController`, `getHandler()` trả về một tham chiếu đến phương thức `create()` và `getClass()` trả về class `CatsController` (không phải instance).

```typescript
const methodKey = ctx.getHandler().name; // "create"
const className = ctx.getClass().name; // "CatsController"
```

Khả năng truy cập các tham chiếu đến cả class hiện tại và phương thức handler mang lại sự linh hoạt rất lớn. Quan trọng nhất, nó cho chúng ta cơ hội truy cập metadata được thiết lập thông qua các decorator được tạo bằng `Reflector#createDecorator` hoặc decorator tích hợp sẵn `@SetMetadata()` từ bên trong guards hoặc interceptors. Chúng tôi bao gồm use case này bên dưới.

<app-banner-enterprise></app-banner-enterprise>

#### Reflection and metadata

Nest cung cấp khả năng đính kèm **custom metadata** vào route handlers thông qua các decorator được tạo bằng phương thức `Reflector#createDecorator`, và decorator tích hợp sẵn `@SetMetadata()`. Trong phần này, hãy so sánh hai cách tiếp cận này và xem cách truy cập metadata từ bên trong một guard hoặc interceptor.

Để tạo các decorator có gõ kiểu mạnh sử dụng `Reflector#createDecorator`, chúng ta cần chỉ định đối số type. Ví dụ, hãy tạo một decorator `Roles` nhận một mảng chuỗi làm đối số.

```ts
@@filename(roles.decorator)
import { Reflector } from '@nestjs/core';

export const Roles = Reflector.createDecorator<string[]>();
```

Decorator `Roles` ở đây là một hàm nhận một đối số duy nhất của loại `string[]`.

Bây giờ, để sử dụng decorator này, chúng ta đơn giản annotate handler với nó:

```typescript
@@filename(cats.controller)
@Post()
@Roles(['admin'])
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
@@switch
@Post()
@Roles(['admin'])
@Bind(Body())
async create(createCatDto) {
  this.catsService.create(createCatDto);
}
```

Ở đây chúng tôi đã đính kèm metadata decorator `Roles` vào phương thức `create()`, chỉ ra rằng chỉ những người dùng với role `admin` mới được phép truy cập route này.

Để truy xuất role(s) của route (custom metadata), chúng tôi sẽ sử dụng class helper `Reflector` một lần nữa. `Reflector` có thể được inject vào một class theo cách thông thường:

```typescript
@@filename(roles.guard)
@Injectable()
export class RolesGuard {
  constructor(private reflector: Reflector) {}
}
@@switch
@Injectable()
@Dependencies(Reflector)
export class CatsService {
  constructor(reflector) {
    this.reflector = reflector;
  }
}
```

> info **Hint** Class `Reflector` được import từ package `@nestjs/core`.

Bây giờ, để đọc metadata của handler, sử dụng phương thức `get()`:

```typescript
const roles = this.reflector.get(Roles, context.getHandler());
```

Phương thức `Reflector#get` cho phép chúng ta truy cập metadata dễ dàng bằng cách truyền vào hai đối số: một tham chiếu decorator và một **context** (mục tiêu decorator) để truy xuất metadata từ đó. Trong ví dụ này, **decorator** được chỉ định là `Roles` (tham chiếu lại file `roles.decorator.ts` ở trên). Context được cung cấp bởi lời gọi `context.getHandler()`, dẫn đến việc trích xuất metadata cho route handler hiện đang được xử lý. Hãy nhớ rằng, `getHandler()` cho chúng ta một **tham chiếu** đến hàm route handler.

Ngoài ra, chúng ta có thể tổ chức controller của mình bằng cách áp dụng metadata ở cấp độ controller, áp dụng cho tất cả các routes trong class controller.

```typescript
@@filename(cats.controller)
@Roles(['admin'])
@Controller('cats')
export class CatsController {}
@@switch
@Roles(['admin'])
@Controller('cats')
export class CatsController {}
```

Trong trường hợp này, để trích xuất metadata của controller, chúng ta truyền `context.getClass()` làm đối số thứ hai (để cung cấp class controller làm context cho việc trích xuất metadata) thay vì `context.getHandler()`:

```typescript
@@filename(roles.guard)
const roles = this.reflector.get(Roles, context.getClass());
```

Đưa ra khả năng cung cấp metadata ở nhiều cấp độ, bạn có thể cần trích xuất và hợp nhất metadata từ nhiều ngữ cảnh. Class `Reflector` cung cấp hai phương thức utility được sử dụng để giúp việc này. Các phương thức này trích xuất metadata **cả** controller và method cùng một lúc, và kết hợp chúng theo các cách khác nhau.

Hãy xem xét kịch bản sau, nơi bạn đã cung cấp metadata `Roles` ở cả hai cấp độ.

```typescript
@@filename(cats.controller)
@Roles(['user'])
@Controller('cats')
export class CatsController {
  @Post()
  @Roles(['admin'])
  async create(@Body() createCatDto: CreateCatDto) {
    this.catsService.create(createCatDto);
  }
}
@@switch
@Roles(['user'])
@Controller('cats')
export class CatsController {}
  @Post()
  @Roles(['admin'])
  @Bind(Body())
  async create(createCatDto) {
    this.catsService.create(createCatDto);
  }
}
```

Nếu ý định của bạn là chỉ định `'user'` làm role mặc định, và ghi đè nó một cách có chọn lọc cho một số phương thức nhất định, bạn có thể sẽ sử dụng phương thức `getAllAndOverride()`.

```typescript
const roles = this.reflector.getAllAndOverride(Roles, [context.getHandler(), context.getClass()]);
```

Một guard với code này, chạy trong ngữ cảnh của phương thức `create()`, với metadata ở trên, sẽ dẫn đến `roles` chứa `['admin']`.

Để lấy metadata cho cả hai và hợp nhất nó (phương thức này hợp nhất cả mảng và objects), sử dụng phương thức `getAllAndMerge()`:

```typescript
const roles = this.reflector.getAllAndMerge(Roles, [context.getHandler(), context.getClass()]);
```

Điều này sẽ dẫn đến `roles` chứa `['user', 'admin']`.

Đối với cả hai phương thức hợp nhất này, bạn truyền metadata key làm đối số đầu tiên, và một mảng các contexts mục tiêu metadata (tức là các lời gọi đến các phương thức `getHandler()` và/hoặc `getClass()`) làm đối số thứ hai.

#### Low-level approach

Như đã đề cập trước đó, thay vì sử dụng `Reflector#createDecorator`, bạn cũng có thể sử dụng decorator tích hợp sẵn `@SetMetadata()` để đính kèm metadata vào một handler.

```typescript
@@filename(cats.controller)
@Post()
@SetMetadata('roles', ['admin'])
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
@@switch
@Post()
@SetMetadata('roles', ['admin'])
@Bind(Body())
async create(createCatDto) {
  this.catsService.create(createCatDto);
}
```

> info **Hint** Decorator `@SetMetadata()` được import từ package `@nestjs/common`.

Với cấu trúc ở trên, chúng tôi đã đính kèm metadata `roles` (`roles` là một metadata key và `['admin']` là giá trị liên kết) vào phương thức `create()`. Mặc dù điều này hoạt động, đó không phải là thực hành tốt để sử dụng `@SetMetadata()` trực tiếp trong các routes của bạn. Thay vào đó, bạn có thể tạo các decorator của riêng bạn, như được hiển thị bên dưới:

```typescript
@@filename(roles.decorator)
import { SetMetadata } from '@nestjs/common';

export const Roles = (...roles: string[]) => SetMetadata('roles', roles);
@@switch
import { SetMetadata } from '@nestjs/common';

export const Roles = (...roles) => SetMetadata('roles', roles);
```

Cách tiếp cận này sạch hơn và dễ đọc hơn, và somewhat giống cách tiếp cận `Reflector#createDecorator`. Sự khác biệt là với `@SetMetadata` bạn có nhiều quyền kiểm soát hơn over metadata key và value, và cũng có thể tạo các decorator nhận nhiều hơn một đối số.

Bây giờ chúng ta có một decorator `@Roles()` tùy chỉnh, chúng ta có thể sử dụng nó để decorate phương thức `create()`.

```typescript
@@filename(cats.controller)
@Post()
@Roles('admin')
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
@@switch
@Post()
@Roles('admin')
@Bind(Body())
async create(createCatDto) {
  this.catsService.create(createCatDto);
}
```

Để truy xuất role(s) của route (custom metadata), chúng tôi sẽ sử dụng class helper `Reflector` một lần nữa:

```typescript
@@filename(roles.guard)
@Injectable()
export class RolesGuard {
  constructor(private reflector: Reflector) {}
}
@@switch
@Injectable()
@Dependencies(Reflector)
export class CatsService {
  constructor(reflector) {
    this.reflector = reflector;
  }
}
```

> info **Hint** Class `Reflector` được import từ package `@nestjs/core`.

Bây giờ, để đọc metadata của handler, sử dụng phương thức `get()`.

```typescript
const roles = this.reflector.get<string[]>('roles', context.getHandler());
```

Ở đây thay vì truyền một tham chiếu decorator, chúng ta truyền metadata **key** làm đối số đầu tiên (trong trường hợp của chúng ta là `'roles'`). Mọi thứ khác vẫn giữ nguyên như trong ví dụ `Reflector#createDecorator`.