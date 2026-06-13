### CSRF Protection

Cross-site request forgery (CSRF hoặc XSRF) là một loại tấn công trong đó các lệnh **không được ủy quyền** được gửi từ một user đáng tin cậy đến một ứng dụng web. Để giúp ngăn chặn điều này, bạn có thể sử dụng package [csrf-csrf](https://github.com/Psifi-Solutions/csrf-csrf).

#### Use with Express (default)

Bắt đầu bằng cách cài đặt package cần thiết:

```bash
$ npm i csrf-csrf
```

> warning **Cảnh báo** Như đã lưu ý trong [tài liệu csrf-csrf](https://github.com/Psifi-Solutions/csrf-csrf?tab=readme-ov-file#getting-started), middleware này yêu cầu session middleware hoặc `cookie-parser` được khởi tạo trước đó. Vui lòng tham khảo tài liệu để biết thêm chi tiết.

Sau khi cài đặt hoàn tất, đăng ký middleware `csrf-csrf` như một global middleware.

```typescript
import { doubleCsrf } from 'csrf-csrf';
// ...
// somewhere in your initialization file
const {
  invalidCsrfTokenError, // This is provided purely for convenience if you plan on creating your own middleware.
  generateToken, // Use this in your routes to generate and provide a CSRF hash, along with a token cookie and token.
  validateRequest, // Also a convenience if you plan on making your own middleware.
  doubleCsrfProtection, // This is the default CSRF protection middleware.
} = doubleCsrf(doubleCsrfOptions);
app.use(doubleCsrfProtection);
```

#### Use with Fastify

Bắt đầu bằng cách cài đặt package cần thiết:

```bash
$ npm i --save @fastify/csrf-protection
```

Sau khi cài đặt hoàn tất, đăng ký plugin `@fastify/csrf-protection`, như sau:

```typescript
import fastifyCsrf from '@fastify/csrf-protection';
// ...
// somewhere in your initialization file after registering some storage plugin
await app.register(fastifyCsrf);
```

> warning **Cảnh báo** Như đã giải thích trong tài liệu `@fastify/csrf-protection` [ở đây](https://github.com/fastify/csrf-protection#usage), plugin này yêu cầu một storage plugin được khởi tạo trước. Vui lòng xem tài liệu đó để biết thêm hướng dẫn.