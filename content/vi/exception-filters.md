### Exception filters

Nest đi kèm với một **exceptions layer** built-in chịu trách nhiệm xử lý tất cả các exceptions unhandled trên toàn ứng dụng. Khi một exception không được xử lý bởi code ứng dụng của bạn, nó bị bắt bởi layer này, sau đó tự động gửi một response thân thiện với người dùng phù hợp.

<figure>
  <img class="illustrative-image" src="/assets/Filter_1.png" />
</figure>

Out of the box, hành động này được thực hiện bởi một **global exception filter** built-in, xử lý các exceptions của loại `HttpException` (và các subclasses của nó). Khi một exception là **unrecognized** (không phải `HttpException` cũng không phải một class kế thừa từ `HttpException`), exception filter built-in tạo ra response JSON mặc định sau:

```json
{
  "statusCode": 500,
  "message": "Internal server error"
}
```

> info **Gợi ý** Global exception filter hỗ trợ một phần thư viện `http-errors`. Về cơ bản, bất kỳ exception được throw nào chứa các thuộc tính `statusCode` và `message` sẽ được điền đúng cách và gửi lại như một response (thay vì `InternalServerErrorException` mặc định cho các exceptions unrecognized).

#### Throwing standard exceptions

Nest cung cấp một class `HttpException` built-in, exposed từ package `@nestjs/common`. Đối với các ứng dụng HTTP REST/GraphQL API điển hình, best practice là gửi các đối tượng response HTTP tiêu chuẩn khi một số điều kiện lỗi xảy ra.

Ví dụ, trong `CatsController`, chúng ta có một method `findAll()` (một route handler `GET`). Hãy giả định rằng route handler này throw một exception vì một lý do nào đó. Để minh họa điều này, chúng ta sẽ hard-code nó như sau:

```typescript
@@filename(cats.controller)
@Get()
async findAll() {
  throw new HttpException('Forbidden', HttpStatus.FORBIDDEN);
}
```

> info **Gợi ý** Chúng ta đã sử dụng `HttpStatus` ở đây. Đây là một helper enum được import từ package `@nestjs/common`.

Khi client gọi endpoint này, response trông như sau:

```json
{
  "statusCode": 403,
  "message": "Forbidden"
}
```

Constructor `HttpException` nhận hai đối số bắt buộc xác định response:

- Đối số `response` định nghĩa body response JSON. Nó có thể là một `string` hoặc một `object` như được mô tả dưới đây.
- Đối số `status` định nghĩa HTTP status code.

Theo mặc định, body response JSON chứa hai thuộc tính:

- `statusCode`: mặc định là HTTP status code được cung cấp trong đối số `status`
- `message`: một mô tả ngắn gọn về lỗi HTTP dựa trên `status`

Để ghi đè chỉ phần message của body response JSON, cung cấp một string trong đối số `response`. Để ghi đè toàn bộ body response JSON, truyền một object trong đối số `response`. Nest sẽ serialize object và trả về nó như body response JSON.

Đối số constructor thứ hai - `status` - nên là một HTTP status code hợp lệ.
Best practice là sử dụng enum `HttpStatus` được import từ `@nestjs/common`.

Có một đối số constructor thứ ba (tùy chọn) - `options` - có thể được sử dụng để cung cấp một error cause. Đối tượng `cause` này không được serialize vào đối tượng response, nhưng nó có thể hữu ích cho mục đích logging, cung cấp thông tin giá trị về lỗi nội bộ gây ra `HttpException` được throw.

Đây là một ví dụ ghi đè toàn bộ body response và cung cấp một error cause:

```typescript
@@filename(cats.controller)
@Get()
async findAll() {
  try {
    await this.service.findAll()
  } catch (error) {
    throw new HttpException({
      status: HttpStatus.FORBIDDEN,
      error: 'This is a custom message',
    }, HttpStatus.FORBIDDEN, {
      cause: error
    });
  }
}
```

Sử dụng ở trên, đây là cách response sẽ trông:

```json
{
  "status": 403,
  "error": "This is a custom message"
}
```

#### Exceptions logging

Theo mặc định, exception filter không log các exceptions built-in như `HttpException` (và bất kỳ exceptions nào kế thừa từ nó). Khi các exceptions này được throw, chúng sẽ không xuất hiện trong console, vì chúng được coi như một phần của luồng ứng dụng bình thường. Cùng hành vi áp dụng cho các exceptions built-in khác như `WsException` và `RpcException`.

Các exceptions này đều kế thừa từ class cơ sở `IntrinsicException`, được export từ package `@nestjs/common`. Class này giúp phân biệt giữa các exceptions là một phần của hoạt động ứng dụng bình thường và những cái không phải.

Nếu bạn muốn log các exceptions này, bạn có thể tạo một custom exception filter. Chúng ta sẽ giải thích cách làm điều này trong phần tiếp theo.

#### Custom exceptions

Trong nhiều trường hợp, bạn sẽ không cần viết custom exceptions, và có thể sử dụng exception HTTP built-in của Nest, như được mô tả trong phần tiếp theo. Nếu bạn cần tạo customized exceptions, good practice là tạo **exceptions hierarchy** của riêng bạn, trong đó custom exceptions của bạn kế thừa từ class cơ sở `HttpException`. Với cách tiếp cận này, Nest sẽ nhận ra exceptions của bạn, và tự động lo việc các error responses. Hãy implement một custom exception như vậy:

```typescript
@@filename(forbidden.exception)
export class ForbiddenException extends HttpException {
  constructor() {
    super('Forbidden', HttpStatus.FORBIDDEN);
  }
}
```

Vì `ForbiddenException` extends class cơ sở `HttpException`, nó sẽ hoạt động seamlessly với exception handler built-in, và do đó chúng ta có thể sử dụng nó bên trong method `findAll()`.

```typescript
@@filename(cats.controller)
@Get()
async findAll() {
  throw new ForbiddenException();
}
```

#### Built-in HTTP exceptions

Nest cung cấp một tập hợp các exceptions tiêu chuẩn kế thừa từ `HttpException` cơ sở. Chúng được expose từ package `@nestjs/common`, và đại diện cho nhiều HTTP exceptions phổ biến nhất:

- `BadRequestException`
- `UnauthorizedException`
- `NotFoundException`
- `ForbiddenException`
- `NotAcceptableException`
- `RequestTimeoutException`
- `ConflictException`
- `GoneException`
- `HttpVersionNotSupportedException`
- `PayloadTooLargeException`
- `UnsupportedMediaTypeException`
- `UnprocessableEntityException`
- `InternalServerErrorException`
- `NotImplementedException`
- `ImATeapotException`
- `MethodNotAllowedException`
- `BadGatewayException`
- `ServiceUnavailableException`
- `GatewayTimeoutException`
- `PreconditionFailedException`

Tất cả các exceptions built-in cũng có thể cung cấp cả error `cause` và error description sử dụng parameter `options`:

```typescript
throw new BadRequestException('Something bad happened', {
  cause: new Error(),
  description: 'Some error description',
});
```

Sử dụng ở trên, đây là cách response sẽ trông:

```json
{
  "message": "Something bad happened",
  "error": "Some error description",
  "statusCode": 400
}
```

#### Exception filters

Trong khi exception filter cơ sở (built-in) có thể tự động xử lý nhiều trường hợp cho bạn, bạn có thể muốn **full control** over exceptions layer. Ví dụ, bạn có thể muốn thêm logging hoặc sử dụng một JSON schema khác dựa trên một số yếu tố động. **Exception filters** được thiết kế chính xác cho mục đích này. Chúng cho phép bạn kiểm soát luồng điều khiển chính xác và nội dung của response được gửi lại cho client.

Hãy tạo một exception filter chịu trách nhiệm bắt các exceptions là một instance của class `HttpException`, và implementing custom response logic cho chúng. Để làm điều này, chúng ta sẽ cần truy cập các đối tượng `Request` và `Response` của nền tảng cơ bản. Chúng ta sẽ truy cập đối tượng `Request` để chúng ta có thể pull ra `url` gốc và bao gồm nó trong thông tin logging. Chúng ta sẽ sử dụng đối tượng `Response` để kiểm soát trực tiếp response được gửi, sử dụng method `response.json()`.

```typescript
@@filename(http-exception.filter)
import { ExceptionFilter, Catch, ArgumentsHost, HttpException } from '@nestjs/common';
import { Request, Response } from 'express';

@Catch(HttpException)
export class HttpExceptionFilter implements ExceptionFilter {
  catch(exception: HttpException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();
    const status = exception.getStatus();

    response
      .status(status)
      .json({
        statusCode: status,
        timestamp: new Date().toISOString(),
        path: request.url,
      });
  }
}
@@switch
import { Catch, HttpException } from '@nestjs/common';

@Catch(HttpException)
export class HttpExceptionFilter {
  catch(exception, host) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse();
    const request = ctx.getRequest();
    const status = exception.getStatus();

    response
      .status(status)
      .json({
        statusCode: status,
        timestamp: new Date().toISOString(),
        path: request.url,
      });
  }
}
```

> info **Gợi ý** Tất cả exception filters nên implement generic interface `ExceptionFilter<T>`. Điều này yêu cầu bạn cung cấp method `catch(exception: T, host: ArgumentsHost)` với signature được chỉ định của nó. `T` chỉ định loại của exception.

> warning **Cảnh báo** Nếu bạn đang sử dụng `@nestjs/platform-fastify` bạn có thể sử dụng `response.send()` thay vì `response.json()`. Đừng quên import các types đúng từ `fastify`.

Decorator `@Catch(HttpException)` bind metadata cần thiết đến exception filter, cho biết Nest rằng filter cụ thể này đang tìm kiếm các exceptions của loại `HttpException` và không có gì khác. Decorator `@Catch()` có thể nhận một parameter đơn lẻ, hoặc một danh sách được phân tách bằng dấu phẩy. Điều này cho phép bạn thiết lập filter cho một số loại exceptions cùng một lúc.

#### Arguments host

Hãy xem các tham số của method `catch()`. Tham số `exception` là đối tượng exception hiện tại đang được xử lý. Tham số `host` là một đối tượng `ArgumentsHost`. `ArgumentsHost` là một đối tượng utility mạnh mẽ mà chúng ta sẽ kiểm tra thêm trong execution context chapter*. Trong mẫu code này, chúng ta sử dụng nó để obtain một tham chiếu đến các đối tượng `Request` và `Response` đang được truyền đến handler yêu cầu gốc (trong controller nơi exception bắt nguồn). Trong mẫu code này, chúng ta đã sử dụng một số helper methods trên `ArgumentsHost` để lấy các đối tượng `Request` và `Response` mong muốn. Tìm hiểu thêm về `ArgumentsHost` ở đây.

*Lý do cho mức độ abstraction này là `ArgumentsHost` hoạt động trong tất cả các contexts (ví dụ, context HTTP server chúng ta đang làm việc ngay bây giờ, nhưng cũng Microservices và WebSockets). Trong execution context chapter chúng ta sẽ thấy cách chúng ta có thể truy cập các underlying arguments phù hợp cho **bất kỳ** execution context nào với sức mạnh của `ArgumentsHost` và các helper functions của nó. Điều này sẽ cho phép chúng ta viết các exception filters generic hoạt động trên tất cả các contexts.

<app-banner-courses></app-banner-courses>

#### Binding filters

Hãy tie filter `HttpExceptionFilter` mới của chúng ta đến method `create()` của `CatsController`.

```typescript
@@filename(cats.controller)
@Post()
@UseFilters(new HttpExceptionFilter())
async create(@Body() createCatDto: CreateCatDto) {
  throw new ForbiddenException();
}
@@switch
@Post()
@UseFilters(new HttpExceptionFilter())
@Bind(Body())
async create(createCatDto) {
  throw new ForbiddenException();
}
```

> info **Gợi ý** Decorator `@UseFilters()` được import từ package `@nestjs/common`.

Chúng ta đã sử dụng decorator `@UseFilters()` ở đây. Tương tự như decorator `@Catch()`, nó có thể nhận một filter instance đơn lẻ, hoặc một danh sách được phân tách bằng dấu phẩy của các filter instances. Ở đây, chúng ta đã tạo instance của `HttpExceptionFilter` in place. Ngoài ra, bạn có thể truyền class (thay vì instance), để lại trách nhiệm instantiation cho framework, và cho phép **dependency injection**.

```typescript
@@filename(cats.controller)
@Post()
@UseFilters(HttpExceptionFilter)
async create(@Body() createCatDto: CreateCatDto) {
  throw new ForbiddenException();
}
@@switch
@Post()
@UseFilters(HttpExceptionFilter)
@Bind(Body())
async create(createCatDto) {
  throw new ForbiddenException();
}
```

> info **Gợi ý** Ưu tiên applying filters sử dụng classes thay vì instances khi có thể. Nó giảm **memory usage** vì Nest có thể dễ dàng tái sử dụng các instances của cùng class trên toàn bộ module của bạn.

Trong ví dụ trên, `HttpExceptionFilter` chỉ được áp dụng cho route handler `create()` đơn lẻ, làm cho nó method-scoped. Exception filters có thể được scoped ở các cấp độ khác nhau: method-scoped của controller/resolver/gateway, controller-scoped, hoặc global-scoped.
Ví dụ, để thiết lập một filter như controller-scoped, bạn sẽ làm như sau:

```typescript
@@filename(cats.controller)
@Controller()
@UseFilters(new HttpExceptionFilter())
export class CatsController {}
```

Cấu trúc này thiết lập `HttpExceptionFilter` cho mọi route handler được định nghĩa bên trong `CatsController`.

Để tạo một filter global-scoped, bạn sẽ làm như sau:

```typescript
@@filename(main)
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalFilters(new HttpExceptionFilter());
  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```

> warning **Cảnh báo** Method `useGlobalFilters()` không thiết lập filters cho gateways hoặc hybrid applications.

Global-scoped filters được sử dụng trên toàn bộ ứng dụng, cho mọi controller và mọi route handler. Về mặt dependency injection, global filters được đăng ký từ bên ngoài bất kỳ module nào (với `useGlobalFilters()` như trong ví dụ trên) không thể inject dependencies vì điều này được thực hiện bên ngoài context của bất kỳ module nào. Để giải quyết vấn đề này, bạn có thể đăng ký một filter global-scoped **trực tiếp từ bất kỳ module nào** sử dụng cấu trúc sau:

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { APP_FILTER } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_FILTER,
      useClass: HttpExceptionFilter,
    },
  ],
})
export class AppModule {}
```

> info **Gợi ý** Khi sử dụng cách tiếp cận này để thực hiện dependency injection cho filter, lưu ý rằng bất kể module nơi cấu trúc này được sử dụng, filter thực tế là global. Nên làm điều này ở đâu? Chọn module nơi filter (`HttpExceptionFilter` trong ví dụ trên) được định nghĩa. Ngoài ra, `useClass` không phải là cách duy nhất để xử lý đăng ký custom provider. Tìm hiểu thêm ở đây.

Bạn có thể thêm nhiều filters với kỹ thuật này khi cần; chỉ cần thêm mỗi cái vào mảng providers.

#### Catch everything

Để bắt **mọi** exception unhandled (bất kể loại exception), để lại danh sách parameter của decorator `@Catch()` trống, ví dụ, `@Catch()`.

Trong ví dụ dưới đây chúng ta có một code là platform-agnostic vì nó sử dụng HTTP adapter để deliver response, và không sử dụng bất kỳ đối tượng platform-specific nào (`Request` và `Response`) trực tiếp:

```typescript
import {
  ExceptionFilter,
  Catch,
  ArgumentsHost,
  HttpException,
  HttpStatus,
} from '@nestjs/common';
import { HttpAdapterHost } from '@nestjs/core';

@Catch()
export class CatchEverythingFilter implements ExceptionFilter {
  constructor(private readonly httpAdapterHost: HttpAdapterHost) {}

  catch(exception: unknown, host: ArgumentsHost): void {
    // In certain situations `httpAdapter` might not be available in the
    // constructor method, thus we should resolve it here.
    const { httpAdapter } = this.httpAdapterHost;

    const ctx = host.switchToHttp();

    const httpStatus =
      exception instanceof HttpException
        ? exception.getStatus()
        : HttpStatus.INTERNAL_SERVER_ERROR;

    const responseBody = {
      statusCode: httpStatus,
      timestamp: new Date().toISOString(),
      path: httpAdapter.getRequestUrl(ctx.getRequest()),
    };

    httpAdapter.reply(ctx.getResponse(), responseBody, httpStatus);
  }
}
```

> warning **Cảnh báo** Khi kết hợp một exception filter bắt mọi thứ với một filter được bound đến một loại cụ thể, filter "Catch anything" nên được khai báo đầu tiên để cho phép filter cụ thể xử lý đúng loại được bound.

#### Inheritance

Thông thường, bạn sẽ tạo các exception filters được tùy chỉnh đầy đủ crafted để fulfill các yêu cầu ứng dụng của bạn. Tuy nhiên, có thể có use-cases khi bạn muốn chỉ đơn giản extend **global exception filter** mặc định built-in, và ghi đè hành vi dựa trên một số yếu tố nhất định.

Để delegate exception processing đến filter cơ sở, bạn cần extend `BaseExceptionFilter` và gọi method `catch()` được kế thừa.

```typescript
@@filename(all-exceptions.filter)
import { Catch, ArgumentsHost } from '@nestjs/common';
import { BaseExceptionFilter } from '@nestjs/core';

@Catch()
export class AllExceptionsFilter extends BaseExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    super.catch(exception, host);
  }
}
@@switch
import { Catch } from '@nestjs/common';
import { BaseExceptionFilter } from '@nestjs/core';

@Catch()
export class AllExceptionsFilter extends BaseExceptionFilter {
  catch(exception, host) {
    super.catch(exception, host);
  }
}
```

> warning **Cảnh báo** Method-scoped và Controller-scoped filters extend `BaseExceptionFilter` không nên được instantiated với `new`. Thay vào đó, hãy để framework instantiate chúng tự động.

Global filters **có thể** extend filter cơ sở. Điều này có thể được thực hiện theo một trong hai cách.

Method đầu tiên là inject tham chiếu `HttpAdapter` khi instantiating custom global filter:

```typescript
async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  const { httpAdapter } = app.get(HttpAdapterHost);
  app.useGlobalFilters(new AllExceptionsFilter(httpAdapter));

  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```

Method thứ hai là sử dụng token `APP_FILTER` như được hiển thị ở đây.