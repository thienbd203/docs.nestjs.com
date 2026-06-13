### Helmet

[Helmet](https://github.com/helmetjs/helmet) có thể giúp bảo vệ ứng dụng của bạn khỏi một số lỗ hổng web nổi tiếng bằng cách đặt các HTTP headers một cách thích hợp. Nhìn chung, Helmet chỉ là một tập hợp các hàm middleware nhỏ hơn đặt các HTTP headers liên quan đến security (đọc [thêm](https://github.com/helmetjs/helmet#how-it-works)).

> info **Gợi ý** Lưu ý rằng việc áp dụng `helmet` như global hoặc đăng ký nó phải đến trước các cuộc gọi khác đến `app.use()` hoặc các hàm setup có thể gọi `app.use()`. Điều này là do cách nền tảng cơ bản (tức là Express hoặc Fastify) hoạt động, trong đó thứ tự mà middleware/routes được định nghĩa là quan trọng. Nếu bạn sử dụng middleware như `helmet` hoặc `cors` sau khi bạn định nghĩa một route, thì middleware đó sẽ không áp dụng cho route đó, nó sẽ chỉ áp dụng cho các route được định nghĩa sau middleware.

#### Use with Express (default)

Bắt đầu bằng cách cài đặt package cần thiết.

```bash
$ npm i --save helmet
```

Sau khi cài đặt hoàn tất, áp dụng nó như một global middleware.

```typescript
import helmet from 'helmet';
// somewhere in your initialization file
app.use(helmet());
```

> warning **Cảnh báo** Khi sử dụng `helmet`, `@apollo/server` (4.x), và [Apollo Sandbox](https://docs.nestjs.com/graphql/quick-start#apollo-sandbox), có thể có một vấn đề với [CSP](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP) trên Apollo Sandbox. Để giải quyết vấn đề này cấu hình CSP như dưới đây:
>
> ```typescript
> app.use(helmet({
>   crossOriginEmbedderPolicy: false,
>   contentSecurityPolicy: {
>     directives: {
>       imgSrc: [`'self'`, 'data:', 'apollo-server-landing-page.cdn.apollographql.com'],
>       scriptSrc: [`'self'`, `https: 'unsafe-inline'`],
>       manifestSrc: [`'self'`, 'apollo-server-landing-page.cdn.apollographql.com'],
>       frameSrc: [`'self'`, 'sandbox.embed.apollographql.com'],
>     },
>   },
> }));

#### Use with Fastify

Nếu bạn đang sử dụng `FastifyAdapter`, cài đặt package [@fastify/helmet](https://github.com/fastify/fastify-helmet):

```bash
$ npm i --save @fastify/helmet
```

[fastify-helmet](https://github.com/fastify/fastify-helmet) không nên được sử dụng như một middleware, mà như một [Fastify plugin](https://www.fastify.io/docs/latest/Reference/Plugins/), tức là bằng cách sử dụng `app.register()`:

```typescript
import helmet from '@fastify/helmet'
// somewhere in your initialization file
await app.register(helmet)
```

> warning **Cảnh báo** Khi sử dụng `apollo-server-fastify` và `@fastify/helmet`, có thể có một vấn đề với [CSP](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP) trên GraphQL playground, để giải quyết xung đột này, cấu hình CSP như dưới đây:
>
> ```typescript
> await app.register(fastifyHelmet, {
>    contentSecurityPolicy: {
>      directives: {
>        defaultSrc: [`'self'`, 'unpkg.com'],
>        styleSrc: [
>          `'self'`,
>          `'unsafe-inline'`,
>          'cdn.jsdelivr.net',
>          'fonts.googleapis.com',
>          'unpkg.com',
>        ],
>        fontSrc: [`'self'`, 'fonts.gstatic.com', 'data:'],
>        imgSrc: [`'self'`, 'data:', 'cdn.jsdelivr.net'],
>        scriptSrc: [
>          `'self'`,
>          `https: 'unsafe-inline'`,
>          'cdn.jsdelivr.net',
>          `'unsafe-eval'`,
>        ],
>      },
>    },
>  });
>
> // If you are not going to use CSP at all, you can use this:
> await app.register(fastifyHelmet, {
>   contentSecurityPolicy: false,
> });
> ```