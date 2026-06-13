### Hiệu suất (Fastify)

Theo mặc định, Nest sử dụng framework [Express](https://expressjs.com/). Như đã đề cập trước đó, Nest cũng cung cấp khả năng tương thích với các thư viện khác như, ví dụ, [Fastify](https://github.com/fastify/fastify). Nest đạt được sự độc lập framework này bằng cách implement một bộ điều hợp framework (framework adapter) có chức năng chính là proxy middleware và các handler sang các implementation cụ thể cho thư viện tương ứng.

> info **Gợi ý** Lưu ý rằng để một bộ điều hợp framework được implement, thư viện đích phải cung cấp xử lý pipeline request/response tương tự như được tìm thấy trong Express.

[Fastify](https://github.com/fastify/fastify) cung cấp một framework thay thế tốt cho Nest vì nó giải quyết các vấn đề thiết kế theo cách tương tự như Express. Tuy nhiên, fastify nhanh hơn Express **nhiều**, đạt được kết quả benchmark tốt hơn gần hai lần. Một câu hỏi hợp lý là tại sao Nest sử dụng Express làm nhà cung cấp HTTP mặc định? Lý do là Express được sử dụng rộng rãi, được biết đến rộng rãi, và có một bộ lớn các middleware tương thích, có sẵn cho người dùng Nest out-of-the-box.

Nhưng vì Nest cung cấp sự độc lập framework, bạn có thể dễ dàng di chuyển giữa chúng. Fastify có thể là một lựa chọn tốt hơn khi bạn đặt giá trị cao vào hiệu suất rất nhanh. Để sử dụng Fastify, chỉ cần chọn `FastifyAdapter` tích hợp sẵn như hiển thị trong chương này.

#### Cài đặt

Đầu tiên, chúng ta cần cài đặt package cần thiết:

```bash
$ npm i --save @nestjs/platform-fastify
```

#### Bộ điều hợp

Khi nền tảng Fastify được cài đặt, chúng ta có thể sử dụng `FastifyAdapter`.

```typescript
@@filename(main)
import { NestFactory } from '@nestjs/core';
import {
  FastifyAdapter,
  NestFastifyApplication,
} from '@nestjs/platform-fastify';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create<NestFastifyApplication>(
    AppModule,
    new FastifyAdapter()
  );
  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```

Theo mặc định, Fastify chỉ lắng nghe trên interface `localhost 127.0.0.1` ([đọc thêm](https://www.fastify.io/docs/latest/Guides/Getting-Started/#your-first-server)). Nếu bạn muốn chấp nhận các kết nối trên các host khác, bạn nên chỉ định `'0.0.0.0'` trong lời gọi `listen()`:

```typescript
async function bootstrap() {
  const app = await NestFactory.create<NestFastifyApplication>(
    AppModule,
    new FastifyAdapter(),
  );
  await app.listen(3000, '0.0.0.0');
}
```

#### Các package cụ thể cho nền tảng

Hãy nhớ rằng khi bạn sử dụng `FastifyAdapter`, Nest sử dụng Fastify làm **nhà cung cấp HTTP**. Điều này có nghĩa là mỗi recipe phụ thuộc vào Express có thể không còn hoạt động. Bạn nên sử dụng các package tương đương của Fastify thay thế.

#### Phản hồi chuyển hướng

Fastify xử lý các phản hồi chuyển hướng hơi khác so với Express. Để thực hiện chuyển hướng đúng với Fastify, trả về cả mã trạng thái và URL, như sau:

```typescript
@Get()
index(@Res() res) {
  res.status(302).redirect('/login');
}
```

#### Tùy chọn Fastify

Bạn có thể truyền các tùy chọn vào constructor Fastify thông qua constructor `FastifyAdapter`. Ví dụ:

```typescript
new FastifyAdapter({ logger: true });
```

#### Middleware

Các hàm middleware truy xuất các đối tượng `req` và `res` thô thay vì các wrapper của Fastify. Đây là cách package `middie` hoạt động (được sử dụng dưới lớp vỏ) và `fastify` - kiểm tra [trang này](https://www.fastify.io/docs/latest/Reference/Middleware/) để biết thêm thông tin,

```typescript
@@filename(logger.middleware)
import { Injectable, NestMiddleware } from '@nestjs/common';
import { FastifyRequest, FastifyReply } from 'fastify';

@Injectable()
export class LoggerMiddleware implements NestMiddleware {
  use(req: FastifyRequest['raw'], res: FastifyReply['raw'], next: () => void) {
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

#### Cấu hình Route

Bạn có thể sử dụng tính năng [cấu hình route](https://fastify.dev/docs/latest/Reference/Routes/#config) của Fastify với decorator `@RouteConfig()`.

```typescript
@RouteConfig({ output: 'hello world' })
@Get()
index(@Req() req) {
  return req.routeConfig.output;
}
```

#### Ràng buộc Route

Kể từ v10.3.0, `@nestjs/platform-fastify` hỗ trợ tính năng [ràng buộc route](https://fastify.dev/docs/latest/Reference/Routes/#constraints) của Fastify với decorator `@RouteConstraints`.

```typescript
@RouteConstraints({ version: '1.2.x' })
newFeature() {
  return 'This works only for version >= 1.2.x';
}
```

> info **Gợi ý** `@RouteConfig()` và `@RouteConstraints` được nhập từ `@nestjs/platform-fastify`.

#### Ví dụ

Một ví dụ hoạt động có sẵn [tại đây](https://github.com/nestjs/nest/tree/master/sample/10-fastify).