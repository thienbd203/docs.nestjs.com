### Middleware

Middleware là một function được gọi **trước** route handler. Middleware functions có quyền truy cập vào các đối tượng request và response, và function middleware `next()` trong chu kỳ request-response của ứng dụng. Function middleware **next** thường được biểu thị bằng một biến tên là `next`.

<figure><img class="illustrative-image" src="/assets/Middlewares_1.png" /></figure>

Nest middleware, theo mặc định, tương đương với express middleware. Mô tả sau từ tài liệu express chính thức mô tả các khả năng của middleware:

<blockquote class="external">
  Middleware functions có thể thực hiện các nhiệm vụ sau:
  <ul>
    <li>thực thi bất kỳ code nào.</li>
    <li>thực hiện các thay đổi đối với các đối tượng request và response.</li>
    <li>kết thúc chu kỳ request-response.</li>
    <li>gọi function middleware tiếp theo trong stack.</li>
    <li>nếu function middleware hiện tại không kết thúc chu kỳ request-response, nó phải gọi <code>next()</code> để
      chuyển điều khiển đến function middleware tiếp theo. Nếu không, yêu cầu sẽ bị treo.</li>
  </ul>
</blockquote>

Bạn implement custom Nest middleware trong một function, hoặc trong một class với decorator `@Injectable()`. Class nên implement interface `NestMiddleware`, trong khi function không có bất kỳ yêu cầu đặc biệt nào. Hãy bắt đầu bằng cách implement một tính năng middleware đơn giản sử dụng method class.

> warning **Cảnh báo** `Express` và `fastify` xử lý middleware khác nhau và cung cấp các method signatures khác nhau, đọc thêm ở đây.

```typescript
@@filename(logger.middleware)
import { Injectable, NestMiddleware } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';

@Injectable()
export class LoggerMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    console.log('Request...');
    next();
  }
}
@@switch
import { Injectable } from '@nestjs/common';

@Injectable()
export class LoggerMiddleware {
  use(req, res, next) {
    console.log('Request...');
    next();
  }
}
```

#### Dependency injection

Nest middleware hỗ trợ đầy đủ Dependency Injection. Giống như với providers và controllers, chúng có thể **inject dependencies** có sẵn trong cùng module. Như thường lệ, điều này được thực hiện thông qua `constructor`.

#### Applying middleware

Không có chỗ cho middleware trong decorator `@Module()`. Thay vào đó, chúng ta thiết lập chúng sử dụng method `configure()` của class module. Các module bao gồm middleware phải implement interface `NestModule`. Hãy thiết lập `LoggerMiddleware` ở mức `AppModule`.

```typescript
@@filename(app.module)
import { Module, NestModule, MiddlewareConsumer } from '@nestjs/common';
import { LoggerMiddleware } from './common/middleware/logger.middleware';
import { CatsModule } from './cats/cats.module';

@Module({
  imports: [CatsModule],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes('cats');
  }
}
@@switch
import { Module } from '@nestjs/common';
import { LoggerMiddleware } from './common/middleware/logger.middleware';
import { CatsModule } from './cats/cats.module';

@Module({
  imports: [CatsModule],
})
export class AppModule {
  configure(consumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes('cats');
  }
}
```

Trong ví dụ trên chúng ta đã thiết lập `LoggerMiddleware` cho các route handlers `/cats` được định nghĩa trước đó bên trong `CatsController`. Chúng ta cũng có thể hạn chế thêm một middleware đến một method yêu cầu cụ thể bằng cách truyền một đối tượng chứa route `path` và request `method` đến method `forRoutes()` khi cấu hình middleware. Trong ví dụ dưới đây, lưu ý rằng chúng ta import enum `RequestMethod` để tham chiếu loại method yêu cầu mong muốn.

```typescript
@@filename(app.module)
import { Module, NestModule, RequestMethod, MiddlewareConsumer } from '@nestjs/common';
import { LoggerMiddleware } from './common/middleware/logger.middleware';
import { CatsModule } from './cats/cats.module';

@Module({
  imports: [CatsModule],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes({ path: 'cats', method: RequestMethod.GET });
  }
}
@@switch
import { Module, RequestMethod } from '@nestjs/common';
import { LoggerMiddleware } from './common/middleware/logger.middleware';
import { CatsModule } from './cats/cats.module';

@Module({
  imports: [CatsModule],
})
export class AppModule {
  configure(consumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes({ path: 'cats', method: RequestMethod.GET });
  }
}
```

> info **Gợi ý** Method `configure()` có thể được làm asynchronous sử dụng `async/await` (ví dụ, bạn có thể `await` hoàn thành của một operation asynchronous bên trong body method `configure()`).

> warning **Cảnh báo** Khi sử dụng adapter `express`, ứng dụng NestJS sẽ đăng ký `json` và `urlencoded` từ package `body-parser` theo mặc định. Điều này có nghĩa là nếu bạn muốn tùy chỉnh middleware đó qua `MiddlewareConsumer`, bạn cần tắt middleware global bằng cách đặt flag `bodyParser` thành `false` khi tạo ứng dụng với `NestFactory.create()`.

#### Route wildcards

Các route dựa trên pattern cũng được hỗ trợ trong NestJS middleware. Ví dụ, wildcard được đặt tên (`*splat`) có thể được sử dụng như một wildcard để khớp bất kỳ kết hợp ký tự nào trong một route. Trong ví dụ sau, middleware sẽ được thực thi cho bất kỳ route nào bắt đầu bằng `abcd/`, bất kể số lượng ký tự theo sau.

```typescript
forRoutes({
  path: 'abcd/*splat',
  method: RequestMethod.ALL,
});
```

> info **Gợi ý** `splat` chỉ đơn giản là tên của tham số wildcard và không có ý nghĩa đặc biệt. Bạn có thể đặt tên nó bất cứ gì bạn thích, ví dụ, `*wildcard`.

Đường dẫn route `'abcd/*'` sẽ khớp `abcd/1`, `abcd/123`, `abcd/abc`, v.v. Dấu gạch ngang (`-`) và dấu chấm (`.`) được hiểu theo nghĩa đen bởi các đường dẫn dựa trên chuỗi. Tuy nhiên, `abcd/` không có ký tự bổ sung sẽ không khớp route. Để làm điều này, bạn cần bọc wildcard trong dấu ngoặc nhọn để làm cho nó tùy chọn:

```typescript
forRoutes({
  path: 'abcd/{*splat}',
  method: RequestMethod.ALL,
});
```

#### Middleware consumer

`MiddlewareConsumer` là một class helper. Nó cung cấp một số methods built-in để quản lý middleware. Tất cả chúng có thể đơn giản được **chained** trong fluent style. Method `forRoutes()` có thể nhận một string đơn lẻ, nhiều strings, một đối tượng `RouteInfo`, một controller class và thậm chí nhiều controller classes. Trong hầu hết các trường hợp bạn có thể sẽ chỉ truyền một danh sách **controllers** được phân tách bằng dấu phẩy. Dưới đây là một ví dụ với một controller đơn lẻ:

```typescript
@@filename(app.module)
import { Module, NestModule, MiddlewareConsumer } from '@nestjs/common';
import { LoggerMiddleware } from './common/middleware/logger.middleware';
import { CatsModule } from './cats/cats.module';
import { CatsController } from './cats/cats.controller';

@Module({
  imports: [CatsModule],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes(CatsController);
  }
}
@@switch
import { Module } from '@nestjs/common';
import { LoggerMiddleware } from './common/middleware/logger.middleware';
import { CatsModule } from './cats/cats.module';
import { CatsController } from './cats/cats.controller';

@Module({
  imports: [CatsModule],
})
export class AppModule {
  configure(consumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes(CatsController);
  }
}
```

> info **Gợi ý** Method `apply()` có thể nhận một middleware đơn lẻ, hoặc nhiều đối số để chỉ định multiple middlewares.

#### Excluding routes

Đôi khi, chúng ta có thể muốn **exclude** một số routes nhất định không được áp dụng middleware. Điều này có thể đạt được dễ dàng sử dụng method `exclude()`. Method `exclude()` chấp nhận một string đơn lẻ, nhiều strings, hoặc một đối tượng `RouteInfo` để xác định các routes bị loại trừ.

Đây là một ví dụ về cách sử dụng nó:

```typescript
consumer
  .apply(LoggerMiddleware)
  .exclude(
    { path: 'cats', method: RequestMethod.GET },
    { path: 'cats', method: RequestMethod.POST },
    'cats/{*splat}',
  )
  .forRoutes(CatsController);
```

> info **Gợi ý** Method `exclude()` hỗ trợ các tham số wildcard sử dụng package path-to-regexp.

Với ví dụ trên, `LoggerMiddleware` sẽ được bound đến tất cả các routes được định nghĩa bên trong `CatsController` **trừ** ba routes được truyền đến method `exclude()`.

Cách tiếp cận này cung cấp tính linh hoạt trong việc áp dụng hoặc loại trừ middleware dựa trên các routes cụ thể hoặc patterns route.

#### Functional middleware

Class `LoggerMiddleware` mà chúng ta đã sử dụng khá đơn giản. Nó không có thành viên, không có methods bổ sung, và không có dependencies. Tại sao chúng ta không thể định nghĩa nó trong một function đơn giản thay vì một class? Trong thực tế, chúng ta có thể. Loại middleware này được gọi là **functional middleware**. Hãy biến đổi logger middleware từ class-based thành functional middleware để minh họa sự khác biệt:

```typescript
@@filename(logger.middleware)
import { Request, Response, NextFunction } from 'express';

export function logger(req: Request, res: Response, next: NextFunction) {
  console.log(`Request...`);
  next();
}
@@switch
export function logger(req, res, next) {
  console.log(`Request...`);
  next();
}
```

Và sử dụng nó bên trong `AppModule`:

```typescript
@@filename(app.module)
consumer
  .apply(logger)
  .forRoutes(CatsController);
```

> info **Gợi ý** Hãy xem xét sử dụng alternative **functional middleware** đơn giản hơn bất cứ khi nào middleware của bạn không cần bất kỳ dependencies nào.

#### Multiple middleware

Như đã đề cập ở trên, để bind nhiều middleware được thực thi tuần tự, chỉ cần cung cấp một danh sách được phân tách bằng dấu phẩy bên trong method `apply()`:

```typescript
consumer.apply(cors(), helmet(), logger).forRoutes(CatsController);
```

#### Global middleware

Nếu chúng ta muốn bind middleware đến mọi route được đăng ký cùng một lúc, chúng ta có thể sử dụng method `use()` được cung cấp bởi instance `INestApplication`:

```typescript
@@filename(main)
const app = await NestFactory.create(AppModule);
app.use(logger);
await app.listen(process.env.PORT ?? 3000);
```

> info **Gợi ý** Truy cập container DI trong một global middleware là không thể. Bạn có thể sử dụng functional middleware thay thế khi sử dụng `app.use()`. Ngoài ra, bạn có thể sử dụng một class middleware và tiêu thụ nó với `.forRoutes('*')` bên trong `AppModule` (hoặc bất kỳ module nào khác).