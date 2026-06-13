### Raw body

Một trong các trường hợp sử dụng phổ biến nhất để có quyền truy cập vào body yêu cầu thô là thực hiện xác minh chữ ký webhook. Thường để thực hiện xác minh chữ ký webhook, body yêu cầu chưa được serialize là cần thiết để tính toán một HMAC hash.

> warning **Cảnh báo** Tính năng này chỉ có thể được sử dụng nếu middleware body parser toàn cầu tích hợp sẵn được bật, tức là, bạn không được truyền `bodyParser: false` khi tạo ứng dụng.

#### Sử dụng với Express

Trước tiên bật tùy chọn khi tạo ứng dụng Nest Express của bạn:

```typescript
import { NestFactory } from '@nestjs/core';
import type { NestExpressApplication } from '@nestjs/platform-express';
import { AppModule } from './app.module';

// trong hàm "bootstrap"
const app = await NestFactory.create<NestExpressApplication>(AppModule, {
  rawBody: true,
});
await app.listen(process.env.PORT ?? 3000);
```

Để truy cập body yêu cầu thô trong một controller, một interface tiện lợi `RawBodyRequest` được cung cấp để expose một trường `rawBody` trên yêu cầu: sử dụng kiểu interface `RawBodyRequest`:

```typescript
import { Controller, Post, RawBodyRequest, Req } from '@nestjs/common';
import { Request } from 'express';

@Controller('cats')
class CatsController {
  @Post()
  create(@Req() req: RawBodyRequest<Request>) {
    const raw = req.rawBody; // trả về một `Buffer`.
  }
}
```

#### Đăng ký một parser khác

Theo mặc định, chỉ có các parser `json` và `urlencoded` được đăng ký. Nếu bạn muốn đăng ký một parser khác ngay lập tức, bạn sẽ cần làm điều đó một cách rõ ràng.

Ví dụ, để đăng ký một parser `text`, bạn có thể sử dụng mã sau:

```typescript
app.useBodyParser('text');
```

> warning **Cảnh báo** Đảm bảo rằng bạn đang cung cấp kiểu ứng dụng chính xác cho lệnh gọi `NestFactory.create`. Đối với các ứng dụng Express, kiểu chính xác là `NestExpressApplication`. Nếu không phương thức `.useBodyParser` sẽ không được tìm thấy.

#### Giới hạn kích thước body parser

Nếu ứng dụng của bạn cần phân tích một body lớn hơn mặc định `100kb` của Express, sử dụng sau:

```typescript
app.useBodyParser('json', { limit: '10mb' });
```

Phương thức `.useBodyParser` sẽ tôn trọng tùy chọn `rawBody` được truyền trong tùy chọn ứng dụng.

#### Sử dụng với Fastify

Trước tiên bật tùy chọn khi tạo ứng dụng Nest Fastify của bạn:

```typescript
import { NestFactory } from '@nestjs/core';
import {
  FastifyAdapter,
  NestFastifyApplication,
} from '@nestjs/platform-fastify';
import { AppModule } from './app.module';

// trong hàm "bootstrap"
const app = await NestFactory.create<NestFastifyApplication>(
  AppModule,
  new FastifyAdapter(),
  {
    rawBody: true,
  },
);
await app.listen(process.env.PORT ?? 3000);
```

Để truy cập body yêu cầu thô trong một controller, một interface tiện lợi `RawBodyRequest` được cung cấp để expose một trường `rawBody` trên yêu cầu: sử dụng kiểu interface `RawBodyRequest`:

```typescript
import { Controller, Post, RawBodyRequest, Req } from '@nestjs/common';
import { FastifyRequest } from 'fastify';

@Controller('cats')
class CatsController {
  @Post()
  create(@Req() req: RawBodyRequest<FastifyRequest>) {
    const raw = req.rawBody; // trả về một `Buffer`.
  }
}
```

#### Đăng ký một parser khác

Theo mặc định, chỉ có các parser `application/json` và `application/x-www-form-urlencoded` được đăng ký. Nếu bạn muốn đăng ký một parser khác ngay lập tức, bạn sẽ cần làm điều đó một cách rõ ràng.

Ví dụ, để đăng ký một parser `text/plain`, bạn có thể sử dụng mã sau:

```typescript
app.useBodyParser('text/plain');
```

> warning **Cảnh báo** Đảm bảo rằng bạn đang cung cấp kiểu ứng dụng chính xác cho lệnh gọi `NestFactory.create`. Đối với các ứng dụng Fastify, kiểu chính xác là `NestFastifyApplication`. Nếu không phương thức `.useBodyParser` sẽ không được tìm thấy.

#### Giới hạn kích thước body parser

Nếu ứng dụng của bạn cần phân tích một body lớn hơn mặc định 1MiB của Fastify, sử dụng sau:

```typescript
const bodyLimit = 10_485_760; // 10MiB
app.useBodyParser('application/json', { bodyLimit });
```

Phương thức `.useBodyParser` sẽ tôn trọng tùy chọn `rawBody` được truyền trong tùy chọn ứng dụng.