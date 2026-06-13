### Compression

Compression có thể giảm đáng kể kích thước của phần thân phản hồi, do đó tăng tốc độ của một ứng dụng web.

Đối với các website **high-traffic** trong production, được khuyến nghị mạnh mẽ để offload compression từ ứng dụng server - thường là trong reverse proxy (ví dụ: Nginx). Trong trường hợp đó, bạn không nên sử dụng compression middleware.

#### Sử dụng với Express (mặc định)

Sử dụng package middleware [compression](https://github.com/expressjs/compression) để bật gzip compression.

Đầu tiên cài đặt package cần thiết:

```bash
$ npm i --save compression
$ npm i --save-dev @types/compression
```

Sau khi cài đặt hoàn tất, áp dụng compression middleware làm middleware toàn cục.

```typescript
import * as compression from 'compression';
// somewhere in your initialization file
app.use(compression());
```

#### Sử dụng với Fastify

Nếu sử dụng `FastifyAdapter`, bạn sẽ muốn sử dụng [fastify-compress](https://github.com/fastify/fastify-compress):

```bash
$ npm i --save @fastify/compress
```

Sau khi cài đặt hoàn tất, áp dụng middleware `@fastify/compress` làm middleware toàn cục.

> warning **Cảnh báo** Vui lòng đảm bảo rằng bạn sử dụng type `NestFastifyApplication` khi tạo ứng dụng. Nếu không, bạn không thể sử dụng `register` để áp dụng compression-middleware.

```typescript
import { FastifyAdapter, NestFastifyApplication } from '@nestjs/platform-fastify';

import compression from '@fastify/compress';

// inside bootstrap()
const app = await NestFactory.create<NestFastifyApplication>(AppModule, new FastifyAdapter());
await app.register(compression);
```

Theo mặc định, `@fastify/compress` sẽ sử dụng Brotli compression (trên Node >= 11.7.0) khi các trình duyệt chỉ ra hỗ trợ cho encoding. Mặc dù Brotli có thể khá hiệu quả về tỷ lệ compression, nó cũng có thể khá chậm. Theo mặc định, Brotli đặt chất lượng compression tối đa là 11, mặc dù nó có thể được điều chỉnh để giảm thời gian compression thay vì chất lượng compression bằng cách điều chỉnh `BROTLI_PARAM_QUALITY` giữa 0 tối thiểu và 11 tối đa. Điều này sẽ cần tinh chỉnh để tối ưu hóa hiệu suất không gian/thời gian. Một ví dụ với chất lượng 4:

```typescript
import { constants } from 'node:zlib';
// somewhere in your initialization file
await app.register(compression, { brotliOptions: { params: { [constants.BROTLI_PARAM_QUALITY]: 4 } } });
```

Để đơn giản hóa, bạn có thể muốn cho `fastify-compress` chỉ sử dụng deflate và gzip để compress phản hồi; bạn sẽ kết thúc với các phản hồi có thể lớn hơn nhưng chúng sẽ được giao nhanh hơn nhiều.

Để chỉ định các encoding, cung cấp đối số thứ hai cho `app.register`:

```typescript
await app.register(compression, { encodings: ['gzip', 'deflate'] });
```

Câu lệnh trên cho `fastify-compress` chỉ sử dụng các encoding gzip và deflate, ưu tiên gzip nếu client hỗ trợ cả hai.