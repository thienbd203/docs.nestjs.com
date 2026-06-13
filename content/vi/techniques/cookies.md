### Cookies

Một **HTTP cookie** là một phần dữ liệu nhỏ được lưu trữ bởi trình duyệt của người dùng. Cookies được thiết kế để là một cơ chế đáng tin cậy để các trang web ghi nhớ thông tin có trạng thái. Khi người dùng truy cập lại trang web, cookie được tự động gửi cùng với yêu cầu.

#### Sử dụng với Express (mặc định)

Đầu tiên cài đặt [package cần thiết](https://github.com/expressjs/cookie-parser) (và các loại của nó cho người dùng TypeScript):

```shell
$ npm i cookie-parser
$ npm i -D @types/cookie-parser
```

Sau khi cài đặt hoàn tất, áp dụng middleware `cookie-parser` làm middleware toàn cục (ví dụ, trong file `main.ts` của bạn).

```typescript
import * as cookieParser from 'cookie-parser';
// somewhere in your initialization file
app.use(cookieParser());
```

Bạn có thể truyền một số tùy chọn cho middleware `cookieParser`:

- `secret` một chuỗi hoặc mảng được sử dụng để ký cookies. Điều này là tùy chọn và nếu không được chỉ định, sẽ không phân tích cú pháp các cookie đã ký. Nếu một chuỗi được cung cấp, điều này được sử dụng làm secret. Nếu một mảng được cung cấp, một nỗ lực sẽ được thực hiện để bỏ ký cookie với mỗi secret theo thứ tự.
- `options` một đối tượng được truyền cho `cookie.parse` làm tùy chọn thứ hai. Xem [cookie](https://www.npmjs.org/package/cookie) để biết thêm thông tin.

Middleware sẽ phân tích cú pháp header `Cookie` trên yêu cầu và expose dữ liệu cookie làm thuộc tính `req.cookies` và, nếu một secret được cung cấp, làm thuộc tính `req.signedCookies`. Các thuộc tính này là các cặp name value của tên cookie đến giá trị cookie.

Khi một secret được cung cấp, module này sẽ bỏ ký và xác thực bất kỳ giá trị cookie đã ký nào và chuyển các cặp name value đó từ `req.cookies` vào `req.signedCookies`. Một cookie đã ký là một cookie có giá trị được tiền tố bằng `s:`. Các cookie đã ký không vượt qua xác thực chữ ký sẽ có giá trị `false` thay vì giá trị đã bị can thiệp.

Với điều này, bạn bây giờ có thể đọc cookies từ bên trong các route handler, như sau:

```typescript
@Get()
findAll(@Req() request: Request) {
  console.log(request.cookies); // or "request.cookies['cookieKey']"
  // or console.log(request.signedCookies);
}
```

> info **Gợi ý** Decorator `@Req()` được nhập từ `@nestjs/common`, trong khi `Request` từ package `express`.

Để đính kèm một cookie vào phản hồi outgoing, sử dụng phương thức `Response#cookie()`:

```typescript
@Get()
findAll(@Res({ passthrough: true }) response: Response) {
  response.cookie('key', 'value')
}
```

> warning **Cảnh báo** Nếu bạn muốn để lại logic xử lý phản hồi cho framework, hãy nhớ đặt tùy chọn `passthrough` thành `true`, như hiển thị ở trên. Đọc thêm [tại đây](/controllers#library-specific-approach).

> info **Gợi ý** Decorator `@Res()` được nhập từ `@nestjs/common`, trong khi `Response` từ package `express`.

#### Sử dụng với Fastify

Đầu tiên cài đặt package cần thiết:

```shell
$ npm i @fastify/cookie
```

Sau khi cài đặt hoàn tất, đăng ký plugin `@fastify/cookie`:

```typescript
import fastifyCookie from '@fastify/cookie';

// somewhere in your initialization file
const app = await NestFactory.create<NestFastifyApplication>(AppModule, new FastifyAdapter());
await app.register(fastifyCookie, {
  secret: 'my-secret', // for cookies signature
});
```

Với điều này, bạn bây giờ có thể đọc cookies từ bên trong các route handler, như sau:

```typescript
@Get()
findAll(@Req() request: FastifyRequest) {
  console.log(request.cookies); // or "request.cookies['cookieKey']"
}
```

> info **Gợi ý** Decorator `@Req()` được nhập từ `@nestjs/common`, trong khi `FastifyRequest` từ package `fastify`.

Để đính kèm một cookie vào phản hồi outgoing, sử dụng phương thức `FastifyReply#setCookie()`:

```typescript
@Get()
findAll(@Res({ passthrough: true }) response: FastifyReply) {
  response.setCookie('key', 'value')
}
```

Để đọc thêm về phương thức `FastifyReply#setCookie()`, kiểm tra [trang này](https://github.com/fastify/fastify-cookie#sending).

> warning **Cảnh báo** Nếu bạn muốn để lại logic xử lý phản hồi cho framework, hãy nhớ đặt tùy chọn `passthrough` thành `true`, như hiển thị ở trên. Đọc thêm [tại đây](/controllers#library-specific-approach).

> info **Gợi ý** Decorator `@Res()` được nhập từ `@nestjs/common`, trong khi `FastifyReply` từ package `fastify`.

#### Tạo decorator tùy chỉnh (cross-platform)

Để cung cấp một cách thuận tiện, declarative để truy cập cookies incoming, chúng ta có thể tạo một [custom decorator](/custom-decorators).

```typescript
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const Cookies = createParamDecorator((data: string, ctx: ExecutionContext) => {
  const request = ctx.switchToHttp().getRequest();
  return data ? request.cookies?.[data] : request.cookies;
});
```

Decorator `@Cookies()` sẽ trích xuất tất cả cookies, hoặc một cookie được đặt tên từ đối tượng `req.cookies` và điền tham số được trang trí với giá trị đó.

Với điều này, chúng ta bây giờ có thể sử dụng decorator trong chữ ký route handler, như sau:

```typescript
@Get()
findAll(@Cookies('name') name: string) {}
```