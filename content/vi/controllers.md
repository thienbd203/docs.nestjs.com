### Controllers

Controllers chịu trách nhiệm xử lý các **yêu cầu** (requests) gửi đến và gửi **phản hồi** (responses) lại cho client.

<figure><img class="illustrative-image" src="/assets/Controllers_1.png" /></figure>

Mục đích của controller là xử lý các yêu cầu cụ thể cho ứng dụng. Cơ chế **routing** xác định controller nào sẽ xử lý mỗi yêu cầu. Thường thì một controller có nhiều routes, và mỗi route có thể thực hiện một hành động khác nhau.

Để tạo một controller cơ bản, chúng ta sử dụng các class và **decorators**. Decorators liên kết các class với metadata cần thiết, cho phép Nest tạo một routing map kết nối các yêu cầu với các controller tương ứng của chúng.

> info **Gợi ý** Để nhanh chóng tạo một CRUD controller với built-in validation, bạn có thể sử dụng CRUD generator của CLI: `nest g resource [name]`.

#### Routing

Trong ví dụ sau, chúng ta sẽ sử dụng decorator `@Controller()`, decorator này **bắt buộc** để định nghĩa một controller cơ bản. Chúng ta sẽ chỉ định một tiền tố đường dẫn route tùy chọn là `cats`. Sử dụng tiền tố đường dẫn trong decorator `@Controller()` giúp chúng ta nhóm các route liên quan lại với nhau và giảm mã lặp lại. Ví dụ, nếu chúng ta muốn nhóm các route quản lý tương tác với thực thể cat dưới đường dẫn `/cats`, chúng ta có thể chỉ định tiền tố đường dẫn `cats` trong decorator `@Controller()`. Bằng cách này, chúng ta không cần lặp lại phần đường dẫn đó cho mỗi route trong file.

```typescript
@@filename(cats.controller)
import { Controller, Get } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Get()
  findAll(): string {
    return 'This action returns all cats';
  }
}
@@switch
import { Controller, Get } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Get()
  findAll() {
    return 'This action returns all cats';
  }
}
```

> info **Gợi ý** Để tạo một controller sử dụng CLI, chỉ cần thực thi lệnh `$ nest g controller [name]`.

Decorator method yêu cầu HTTP `@Get()` được đặt trước method `findAll()` báo cho Nest tạo một handler cho một endpoint cụ thể cho các yêu cầu HTTP. Endpoint này được định nghĩa bởi method yêu cầu HTTP (GET trong trường hợp này) và đường dẫn route. Vậy đường dẫn route là gì? Đường dẫn route cho một handler được xác định bằng cách kết hợp tiền tố (tùy chọn) được khai báo cho controller với bất kỳ đường dẫn nào được chỉ định trong decorator của method. Vì chúng ta đã đặt một tiền tố (`cats`) cho mỗi route và chưa thêm bất kỳ đường dẫn cụ thể nào trong decorator method, Nest sẽ map các yêu cầu `GET /cats` đến handler này.

Như đã đề cập, đường dẫn route bao gồm cả tiền tố đường dẫn controller tùy chọn **và** bất kỳ chuỗi đường dẫn nào được chỉ định trong decorator của method. Ví dụ, nếu tiền tố controller là `cats` và decorator method là `@Get('breed')`, route kết quả sẽ là `GET /cats/breed`.

Trong ví dụ trên của chúng ta, khi một yêu cầu GET được thực hiện đến endpoint này, Nest route yêu cầu đến method `findAll()` do người dùng định nghĩa. Lưu ý rằng tên method chúng ta chọn ở đây hoàn toàn tùy ý. Trong khi chúng ta phải khai báo một method để bind route đến, Nest không gắn bất kỳ ý nghĩa cụ thể nào cho tên method.

Method này sẽ trả về mã trạng thái 200 cùng với phản hồi liên quan, trong trường hợp này chỉ là một chuỗi. Tại sao điều này xảy ra? Để giải thích, trước tiên chúng ta cần giới thiệu khái niệm rằng Nest sử dụng hai tùy chọn **khác nhau** để thao tác với các phản hồi:

<table>
  <tr>
    <td>Standard (khuyến nghị)</td>
    <td>
      Sử dụng method built-in này, khi một handler yêu cầu trả về một đối tượng hoặc mảng JavaScript, nó sẽ <strong>tự động</strong>
      được serialize sang JSON. Khi nó trả về một kiểu nguyên thủy JavaScript (ví dụ: <code>string</code>, <code>number</code>, <code>boolean</code>), tuy nhiên, Nest sẽ chỉ gửi giá trị mà không cố gắng serialize nó. Điều này làm cho việc xử lý phản hồi đơn giản: chỉ cần trả về giá trị, và Nest lo phần còn lại.
      <br />
      <br /> Hơn nữa, <strong>mã trạng thái</strong> của phản hồi luôn là 200 theo mặc định, ngoại trừ các yêu cầu
      POST sử dụng 201. Chúng ta có thể dễ dàng thay đổi hành vi này bằng cách thêm decorator <code>@HttpCode(...)</code>
      ở mức handler (xem <a href='controllers#status-code'>Mã trạng thái</a>).
    </td>
  </tr>
  <tr>
    <td>Library-specific</td>
    <td>
      Chúng ta có thể sử dụng đối tượng phản hồi library-specific (ví dụ: Express) <a href="https://expressjs.com/en/api.html#res" rel="nofollow" target="_blank">response object</a>, có thể được inject bằng decorator <code>@Res()</code> trong signature handler method (ví dụ, <code>findAll(@Res() response)</code>).  Với cách tiếp cận này, bạn có khả năng sử dụng các methods xử lý phản hồi native được expose bởi đối tượng đó.  Ví dụ, với Express, bạn có thể xây dựng các phản hồi sử dụng code như <code>response.status(200).send()</code>.
    </td>
  </tr>
</table>

> warning **Cảnh báo** Nest phát hiện khi handler sử dụng `@Res()` hoặc `@Next()`, cho biết bạn đã chọn tùy chọn library-specific. Nếu cả hai cách tiếp cận được sử dụng cùng lúc, cách tiếp cận Standard bị **tự động vô hiệu hóa** cho route đơn này và sẽ không còn hoạt động như mong đợi. Để sử dụng cả hai cách tiếp cận cùng lúc (ví dụ, bằng cách inject đối tượng phản hồi chỉ để đặt cookies/headers nhưng vẫn để phần còn lại cho framework), bạn phải đặt tùy chọn `passthrough` thành `true` trong decorator `@Res({ passthrough: true })`.

<app-banner-devtools></app-banner-devtools>

#### Request object

Các handler thường cần truy cập chi tiết **yêu cầu** của client. Nest cung cấp truy cập đến [request object](https://expressjs.com/en/api.html#req) từ nền tảng cơ bản (Express theo mặc định). Bạn có thể truy cập request object bằng cách chỉ thị Nest inject nó sử dụng decorator `@Req()` trong signature của handler.

```typescript
@@filename(cats.controller)
import { Controller, Get, Req } from '@nestjs/common';
import type { Request } from 'express';

@Controller('cats')
export class CatsController {
  @Get()
  findAll(@Req() request: Request): string {
    return 'This action returns all cats';
  }
}
@@switch
import { Controller, Bind, Get, Req } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Get()
  @Bind(Req())
  findAll(request) {
    return 'This action returns all cats';
  }
}
```

> info **Gợi ý** Để tận dụng typings của `express` (như trong ví dụ parameter `request: Request` ở trên), hãy đảm bảo cài đặt package `@types/express`.

Request object đại diện cho yêu cầu HTTP và chứa các thuộc tính cho query string, parameters, HTTP headers, và body (đọc thêm ở đây). Trong hầu hết các trường hợp, bạn không cần truy cập thủ công các thuộc tính này. Thay vào đó, bạn có thể sử dụng các decorators chuyên dụng như `@Body()` hoặc `@Query()`, có sẵn out-of-the-box. Dưới đây là danh sách các decorators được cung cấp và các đối tượng platform-specific tương ứng mà chúng đại diện.

<table>
  <tbody>
    <tr>
      <td><code>@Request(), @Req()</code></td>
      <td><code>req</code></td></tr>
    <tr>
      <td><code>@Response(), @Res()</code><span class="table-code-asterisk">*</span></td>
      <td><code>res</code></td>
    </tr>
    <tr>
      <td><code>@Next()</code></td>
      <td><code>next</code></td>
    </tr>
    <tr>
      <td><code>@Session()</code></td>
      <td><code>req.session</code></td>
    </tr>
    <tr>
      <td><code>@Param(key?: string)</code></td>
      <td><code>req.params</code> / <code>req.params[key]</code></td>
    </tr>
    <tr>
      <td><code>@Body(key?: string)</code></td>
      <td><code>req.body</code> / <code>req.body[key]</code></td>
    </tr>
    <tr>
      <td><code>@Query(key?: string)</code></td>
      <td><code>req.query</code> / <code>req.query[key]</code></td>
    </tr>
    <tr>
      <td><code>@Headers(name?: string)</code></td>
      <td><code>req.headers</code> / <code>req.headers[name]</code></td>
    </tr>
    <tr>
      <td><code>@Ip()</code></td>
      <td><code>req.ip</code></td>
    </tr>
    <tr>
      <td><code>@HostParam()</code></td>
      <td><code>req.hosts</code></td>
    </tr>
  </tbody>
</table>

<sup>\* </sup>Để tương thích với typings trên các nền tảng HTTP cơ bản (ví dụ, Express và Fastify), Nest cung cấp decorators `@Res()` và `@Response()`. `@Res()` chỉ là một alias cho `@Response()`. Cả hai đều expose trực tiếp interface đối tượng phản hồi native platform cơ bản. Khi sử dụng chúng, bạn cũng nên import typings cho thư viện cơ bản (ví dụ, `@types/express`) để tận dụng đầy đủ. Lưu ý rằng khi bạn inject `@Res()` hoặc `@Response()` trong một handler method, bạn đặt Nest vào **Library-specific mode** cho handler đó, và bạn trở thành người chịu trách nhiệm quản lý phản hồi. Khi làm như vậy, bạn phải phát hành một số loại phản hồi bằng cách thực hiện một cuộc gọi trên đối tượng phản hồi (ví dụ, `res.json(...)` hoặc `res.send(...)`), hoặc HTTP server sẽ treo.

> info **Gợi ý** Để tìm hiểu cách tạo decorators tùy chỉnh của riêng bạn, hãy truy cập chương này.

#### Resources

Trước đó, chúng ta định nghĩa một endpoint để lấy tài nguyên cats (route **GET**). Chúng ta thường cũng muốn cung cấp một endpoint tạo ra các bản ghi mới. Để làm điều này, hãy tạo handler **POST**:

```typescript
@@filename(cats.controller)
import { Controller, Get, Post } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Post()
  create(): string {
    return 'This action adds a new cat';
  }

  @Get()
  findAll(): string {
    return 'This action returns all cats';
  }
}
@@switch
import { Controller, Get, Post } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Post()
  create() {
    return 'This action adds a new cat';
  }

  @Get()
  findAll() {
    return 'This action returns all cats';
  }
}
```

Đơn giản vậy thôi. Nest cung cấp decorators cho tất cả các method HTTP tiêu chuẩn: `@Get()`, `@Post()`, `@Put()`, `@Delete()`, `@Patch()`, `@Options()`, và `@Head()`. Ngoài ra, `@All()` định nghĩa một endpoint xử lý tất cả chúng.

#### Route wildcards

Các route dựa trên pattern cũng được hỗ trợ trong NestJS. Ví dụ, dấu hoa thị (`*`) có thể được sử dụng như một wildcard để khớp bất kỳ kết hợp ký tự nào trong một route ở cuối đường dẫn. Trong ví dụ sau, method `findAll()` sẽ được thực thi cho bất kỳ route nào bắt đầu bằng `abcd/`, bất kể số lượng ký tự theo sau.

```typescript
@Get('abcd/*')
findAll() {
  return 'This route uses a wildcard';
}
```

Đường dẫn route `'abcd/*'` sẽ khớp `abcd/`, `abcd/123`, `abcd/abc`, v.v. Dấu gạch ngang (`-`) và dấu chấm (`.`) được hiểu theo nghĩa đen bởi các đường dẫn dựa trên chuỗi.

Cách tiếp cận này hoạt động trên cả Express và Fastify. Tuy nhiên, với bản phát hành mới nhất của Express (v5), hệ thống routing đã trở nên nghiêm ngặt hơn. Trong Express thuần, bạn phải sử dụng một wildcard được đặt tên để làm cho route hoạt động—ví dụ, `abcd/*splat`, trong đó `splat` chỉ là tên của tham số wildcard và không có ý nghĩa đặc biệt. Bạn có thể đặt tên nó bất cứ gì bạn thích. Điều đó nói rằng, vì Nest cung cấp một lớp tương thích cho Express, bạn vẫn có thể sử dụng dấu hoa thị (`*`) như một wildcard.

Khi đến dấu hoa thị được sử dụng ở **giữa một route**, Express yêu cầu wildcards được đặt tên (ví dụ, `ab*splatcd`), trong khi Fastify không hỗ trợ chúng hoàn toàn.

#### Status code

Như đã đề cập, mã trạng thái mặc định cho các phản hồi luôn là **200**, ngoại trừ các yêu cầu POST, mặc định là **201**. Bạn có thể dễ dàng thay đổi hành vi này bằng cách sử dụng decorator `@HttpCode(...)` ở mức handler.

```typescript
@Post()
@HttpCode(204)
create() {
  return 'This action adds a new cat';
}
```

> info **Gợi ý** Import `HttpCode` từ package `@nestjs/common`.

Thường thì mã trạng thái của bạn không tĩnh mà phụ thuộc vào các yếu tố khác nhau. Trong trường hợp đó, bạn có thể sử dụng đối tượng **phản hồi** library-specific (inject sử dụng `@Res()`) (hoặc, trong trường hợp lỗi, throw một exception).

#### Response headers

Để chỉ định một phản hồi header tùy chỉnh, bạn có thể sử dụng decorator `@Header()` hoặc đối tượng phản hồi library-specific (và gọi `res.header()` trực tiếp).

```typescript
@Post()
@Header('Cache-Control', 'no-store')
create() {
  return 'This action adds a new cat';
}
```

> info **Gợi ý** Import `Header` từ package `@nestjs/common`.

#### Redirection

Để chuyển hướng một phản hồi đến một URL cụ thể, bạn có thể sử dụng decorator `@Redirect()` hoặc đối tượng phản hồi library-specific (và gọi `res.redirect()` trực tiếp).

`@Redirect()` nhận hai đối số, `url` và `statusCode`, cả hai đều tùy chọn. Giá trị mặc định của `statusCode` là `302` (`Found`) nếu bị bỏ qua.

```typescript
@Get()
@Redirect('https://nestjs.com', 301)
```

> info **Gợi ý** Đôi khi bạn có thể muốn xác định mã trạng thái HTTP hoặc URL chuyển hướng động. Làm điều này bằng cách trả về một đối tượng theo interface `HttpRedirectResponse` (từ `@nestjs/common`).

Các giá trị trả về sẽ ghi đè bất kỳ đối số nào được truyền đến decorator `@Redirect()`. Ví dụ:

```typescript
@Get('docs')
@Redirect('https://docs.nestjs.com', 302)
getDocs(@Query('version') version) {
  if (version && version === '5') {
    return { url: 'https://docs.nestjs.com/v5/' };
  }
}
```

#### Route parameters

Các route với đường dẫn tĩnh sẽ không hoạt động khi bạn cần chấp nhận **dữ liệu động** như một phần của yêu cầu (ví dụ, `GET /cats/1` để lấy con mèo với id `1`). Để định nghĩa các route với các tham số, bạn có thể thêm các **tokens** tham số route trong đường dẫn route để bắt các giá trị động từ URL. Token tham số route trong ví dụ decorator `@Get()` dưới đây minh họa cách tiếp cận này. Các tham số route này sau đó có thể được truy cập sử dụng decorator `@Param()`, nên được thêm vào signature method.

> info **Gợi ý** Các route với tham số nên được khai báo sau bất kỳ đường dẫn tĩnh nào. Điều này ngăn các đường dẫn được tham số hóa chặn traffic dành cho các đường dẫn tĩnh.

```typescript
@@filename()
@Get(':id')
findOne(@Param() params: any): string {
  console.log(params.id);
  return `This action returns a #${params.id} cat`;
}
@@switch
@Get(':id')
@Bind(Param())
findOne(params) {
  console.log(params.id);
  return `This action returns a #${params.id} cat`;
}
```

Decorator `@Param()` được sử dụng để decorate một tham số method (trong ví dụ trên, `params`), làm cho các tham số **route** có thể truy cập được như các thuộc tính của tham số method được decorate đó bên trong method. Như được thể hiện trong code, bạn có thể truy cập tham số `id` bằng cách tham chiếu `params.id`. Ngoài ra, bạn có thể truyền một token tham số cụ thể đến decorator và tham chiếu trực tiếp tham số route theo tên bên trong method body.

> info **Gợi ý** Import `Param` từ package `@nestjs/common`.

```typescript
@@filename()
@Get(':id')
findOne(@Param('id') id: string): string {
  return `This action returns a #${id} cat`;
}
@@switch
@Get(':id')
@Bind(Param('id'))
findOne(id) {
  return `This action returns a #${id} cat`;
}
```

#### Sub-domain routing

Decorator `@Controller` có thể nhận một tùy chọn `host` để yêu cầu rằng HTTP host của các yêu cầu gửi đến khớp với một giá trị cụ thể.

```typescript
@Controller({ host: 'admin.example.com' })
export class AdminController {
  @Get()
  index(): string {
    return 'Admin page';
  }
}
```

> warning **Cảnh báo** Vì **Fastify** không hỗ trợ nested routers, nếu bạn đang sử dụng sub-domain routing, nên sử dụng Express adapter mặc định thay thế.

Tương tự như một đường dẫn route `path`, tùy chọn `host` có thể sử dụng tokens để bắt giá trị động ở vị trí đó trong tên host. Token tham số host trong ví dụ decorator `@Controller()` dưới đây minh họa cách sử dụng này. Các tham số host được khai báo theo cách này có thể được truy cập sử dụng decorator `@HostParam()`, nên được thêm vào signature method.

```typescript
@Controller({ host: ':account.example.com' })
export class AccountController {
  @Get()
  getInfo(@HostParam('account') account: string) {
    return account;
  }
}
```

#### State sharing

Đối với các nhà phát triển đến từ các ngôn ngữ lập trình khác, có thể đáng ngạc nhiên khi biết rằng trong Nest, gần như mọi thứ được chia sẻ trên các yêu cầu gửi đến. Điều này bao gồm các tài nguyên như pool kết nối database, các service singleton với trạng thái toàn cầu, và nhiều hơn nữa. Điều quan trọng là phải hiểu rằng Node.js không sử dụng Mô hình Multi-Threaded Stateless request/response, trong đó mỗi yêu cầu được xử lý bởi một thread riêng. Kết quả là, sử dụng các instance singleton trong Nest hoàn toàn **an toàn** cho các ứng dụng của chúng ta.

Điều đó nói rằng, có các trường hợp edge cụ thể mà có thể cần có các lifetime dựa trên yêu cầu cho controllers. Ví dụ bao gồm caching theo yêu cầu trong các ứng dụng GraphQL, theo dõi yêu cầu, hoặc triển khai multi-tenancy. Bạn có thể tìm hiểu thêm về việc kiểm soát các phạm vi injection ở đây.

#### Asynchronicity

Chúng tôi yêu thích JavaScript hiện đại, đặc biệt là sự nhấn mạnh của nó vào việc xử lý dữ liệu **bất đồng bộ**. Đó là lý do Nest hỗ trợ đầy đủ các hàm `async`. Mỗi hàm `async` phải trả về một `Promise`, cho phép bạn trả về một giá trị trì hoãn mà Nest có thể giải quyết tự động. Đây là một ví dụ:

```typescript
@@filename(cats.controller)
@Get()
async findAll(): Promise<any[]> {
  return [];
}
@@switch
@Get()
async findAll() {
  return [];
}
```

Code này hoàn toàn hợp lệ. Nhưng Nest đi xa hơn bằng cách cho phép các handler route trả về các luồng observable RxJS cũng được. Nest sẽ xử lý subscription nội bộ và giải quyết giá trị phát ra cuối cùng khi luồng hoàn thành.

```typescript
@@filename(cats.controller)
@Get()
findAll(): Observable<any[]> {
  return of([]);
}
@@switch
@Get()
findAll() {
  return of([]);
}
```

Cả hai cách tiếp cận đều hợp lệ, và bạn có thể chọn cái phù hợp nhất với nhu cầu của bạn.

#### Request payloads

Trong ví dụ trước của chúng ta, handler route POST không chấp nhận bất kỳ tham số client nào. Hãy sửa điều đó bằng cách thêm decorator `@Body()`.

Trước khi tiếp tục (nếu bạn đang sử dụng TypeScript), chúng ta cần định nghĩa schema **DTO** (Data Transfer Object). DTO là một đối tượng chỉ định cách dữ liệu nên được gửi qua mạng. Chúng ta có thể định nghĩa schema DTO sử dụng các interfaces **TypeScript** hoặc các class đơn giản. Tuy nhiên, chúng tôi khuyến nghị sử dụng **classes** ở đây. Tại sao? Classes là một phần của tiêu chuẩn JavaScript ES6, vì vậy chúng vẫn nguyên vẹn như các thực thể thực trong JavaScript đã biên dịch. Ngược lại, các interfaces TypeScript bị xóa trong quá trình transpilation, có nghĩa là Nest không thể tham chiếu chúng tại runtime. Điều này quan trọng vì các tính năng như **Pipes** dựa vào việc có quyền truy cập metatype của các biến tại runtime, điều này chỉ có thể với classes.

Hãy tạo class `CreateCatDto`:

```typescript
@@filename(create-cat.dto)
export class CreateCatDto {
  name: string;
  age: number;
  breed: string;
}
```

Nó chỉ có ba thuộc tính cơ bản. Sau đó chúng ta có thể sử dụng DTO mới tạo bên trong `CatsController`:

```typescript
@@filename(cats.controller)
@Post()
async create(@Body() createCatDto: CreateCatDto) {
  return 'This action adds a new cat';
}
@@switch
@Post()
@Bind(Body())
async create(createCatDto) {
  return 'This action adds a new cat';
}
```

> info **Gợi ý** `ValidationPipe` của chúng ta có thể lọc ra các thuộc tính không nên được nhận bởi handler method. Trong trường hợp này, chúng ta có thể whitelist các thuộc tính có thể chấp nhận, và bất kỳ thuộc tính nào không được bao gồm trong whitelist bị tự động tách khỏi đối tượng kết quả. Trong ví dụ `CreateCatDto`, whitelist của chúng ta là các thuộc tính `name`, `age`, và `breed`. Tìm hiểu thêm ở đây.

#### Query parameters

Khi xử lý các tham số query trong các route của bạn, bạn có thể sử dụng decorator `@Query()` để trích xuất chúng từ các yêu cầu gửi đến. Hãy xem cách này hoạt động trong thực tế.

Xem xét một route nơi chúng ta muốn lọc danh sách mèo dựa trên các tham số query như `age` và `breed`. Đầu tiên, định nghĩa các tham số query trong `CatsController`:

```typescript
@@filename(cats.controller)
@Get()
async findAll(@Query('age') age: number, @Query('breed') breed: string) {
  return `This action returns all cats filtered by age: ${age} and breed: ${breed}`;
}
```

Trong ví dụ này, decorator `@Query()` được sử dụng để trích xuất các giá trị của `age` và `breed` từ query string. Ví dụ, một yêu cầu đến:

```plaintext
GET /cats?age=2&breed=Persian
```

sẽ dẫn đến `age` là `2` và `breed` là `Persian`.

Nếu ứng dụng của bạn yêu cầu xử lý các tham số query phức tạp hơn, chẳng hạn như các đối tượng lồng nhau hoặc mảng:

```plaintext
?filter[where][name]=John&filter[where][age]=30
?item[]=1&item[]=2
```

bạn sẽ cần cấu hình HTTP adapter của bạn (Express hoặc Fastify) để sử dụng một query parser phù hợp. Trong Express, bạn có thể sử dụng parser `extended`, cho phép các đối tượng query phong phú:

```typescript
const app = await NestFactory.create<NestExpressApplication>(AppModule);
app.set('query parser', 'extended');
```

Trong Fastify, bạn có thể sử dụng tùy chọn `querystringParser`:

```typescript
const app = await NestFactory.create<NestFastifyApplication>(
  AppModule,
  new FastifyAdapter({
    querystringParser: (str) => qs.parse(str),
  }),
);
```

> info **Gợi ý** `qs` là một querystring parser hỗ trợ nesting và mảng. Bạn có thể cài đặt nó sử dụng `npm install qs`.

#### Handling errors

Có một chương riêng về xử lý lỗi (tức là làm việc với exceptions) ở đây.

#### Full resource sample

Dưới đây là một ví dụ minh họa việc sử dụng một số decorators có sẵn để tạo một controller cơ bản. Controller này cung cấp một số methods để truy cập và thao tác dữ liệu nội bộ.

```typescript
@@filename(cats.controller)
import { Controller, Get, Query, Post, Body, Put, Param, Delete } from '@nestjs/common';
import { CreateCatDto, UpdateCatDto, ListAllEntities } from './dto';

@Controller('cats')
export class CatsController {
  @Post()
  create(@Body() createCatDto: CreateCatDto) {
    return 'This action adds a new cat';
  }

  @Get()
  findAll(@Query() query: ListAllEntities) {
    return `This action returns all cats (limit: ${query.limit} items)`;
  }

  @Get(':id')
  findOne(@Param('id') id: string) {
    return `This action returns a #${id} cat`;
  }

  @Put(':id')
  update(@Param('id') id: string, @Body() updateCatDto: UpdateCatDto) {
    return `This action updates a #${id} cat`;
  }

  @Delete(':id')
  remove(@Param('id') id: string) {
    return `This action removes a #${id} cat`;
  }
}
@@switch
import { Controller, Get, Query, Post, Body, Put, Param, Delete, Bind } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Post()
  @Bind(Body())
  create(createCatDto) {
    return 'This action adds a new cat';
  }

  @Get()
  @Bind(Query())
  findAll(query) {
    console.log(query);
    return `This action returns all cats (limit: ${query.limit} items)`;
  }

  @Get(':id')
  @Bind(Param('id'))
  findOne(id) {
    return `This action returns a #${id} cat`;
  }

  @Put(':id')
  @Bind(Param('id'), Body())
  update(id, updateCatDto) {
    return `This action updates a #${id} cat`;
  }

  @Delete(':id')
  @Bind(Param('id'))
  remove(id) {
    return `This action removes a #${id} cat`;
  }
}
```

> info **Gợi ý** Nest CLI cung cấp một generator (schematic) tự động tạo **tất cả mã boilerplate**, giúp bạn không phải làm điều này thủ công và cải thiện trải nghiệm nhà phát triển tổng thể. Tìm hiểu thêm về tính năng này ở đây.

#### Getting up and running

Ngay cả với `CatsController` được định nghĩa đầy đủ, Nest vẫn chưa biết về nó và sẽ không tự động tạo một instance của class.

Controllers phải luôn là một phần của một module, đó là lý do chúng ta bao gồm mảng `controllers` trong decorator `@Module()`. Vì chúng ta chưa định nghĩa bất kỳ module nào khác ngoài module gốc `AppModule`, chúng ta sẽ sử dụng nó để đăng ký `CatsController`:

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { CatsController } from './cats/cats.controller';

@Module({
  controllers: [CatsController],
})
export class AppModule {}
```

Chúng ta đã gắn metadata vào class module sử dụng decorator `@Module()`, và bây giờ Nest có thể dễ dàng xác định controllers nào cần được mount.

#### Library-specific approach

Cho đến nay, chúng ta đã bao gồm cách tiêu chuẩn của Nest để thao tác với các phản hồi. Một cách tiếp cận khác là sử dụng một đối tượng phản hồi library-specific. Để inject một đối tượng phản hồi cụ thể, chúng ta có thể sử dụng decorator `@Res()`. Để làm nổi bật sự khác biệt, hãy viết lại `CatsController` như sau:

```typescript
@@filename()
import { Controller, Get, Post, Res, HttpStatus } from '@nestjs/common';
import { Response } from 'express';

@Controller('cats')
export class CatsController {
  @Post()
  create(@Res() res: Response) {
    res.status(HttpStatus.CREATED).send();
  }

  @Get()
  findAll(@Res() res: Response) {
     res.status(HttpStatus.OK).json([]);
  }
}
@@switch
import { Controller, Get, Post, Bind, Res, Body, HttpStatus } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Post()
  @Bind(Res(), Body())
  create(res, createCatDto) {
    res.status(HttpStatus.CREATED).send();
  }

  @Get()
  @Bind(Res())
  findAll(res) {
     res.status(HttpStatus.OK).json([]);
  }
}
```

Trong khi cách tiếp cận này hoạt động và cung cấp tính linh hoạt hơn bằng cách cung cấp kiểm soát hoàn toàn đối tượng phản hồi (như thao tác header và truy cập các tính năng library-specific), nó nên được sử dụng một cách thận trọng. Nói chung, method này ít rõ ràng hơn và đi kèm với một số nhược điểm. Nhược điểm chính là code của bạn trở thành phụ thuộc nền tảng, vì các thư viện cơ bản khác nhau có thể có các API khác nhau cho đối tượng phản hồi. Ngoài ra, nó có thể làm cho việc kiểm thử trở nên khó khăn hơn, vì bạn sẽ cần mock đối tượng phản hồi, cùng với những thứ khác.

Hơn nữa, bằng cách sử dụng cách tiếp cận này, bạn mất tương thích với các tính năng của Nest dựa trên xử lý phản hồi tiêu chuẩn, chẳng hạn như Interceptors và decorators `@HttpCode()` / `@Header()`. Để giải quyết điều này, bạn có thể kích hoạt tùy chọn `passthrough` như sau:

```typescript
@@filename()
@Get()
findAll(@Res({ passthrough: true }) res: Response) {
  res.status(HttpStatus.OK);
  return [];
}
@@switch
@Get()
@Bind(Res({ passthrough: true }))
findAll(res) {
  res.status(HttpStatus.OK);
  return [];
}
```

Với cách tiếp cận này, bạn có thể tương tác với đối tượng phản hồi native (ví dụ, đặt cookies hoặc headers dựa trên các điều kiện cụ thể), trong khi vẫn cho phép framework xử lý phần còn lại.