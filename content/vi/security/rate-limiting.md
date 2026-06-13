### Rate Limiting

Một kỹ thuật phổ biến để bảo vệ ứng dụng khỏi các tấn công brute-force là **rate-limiting**. Để bắt đầu, bạn sẽ cần cài đặt package `@nestjs/throttler`.

```bash
$ npm i --save @nestjs/throttler
```

Sau khi cài đặt hoàn tất, `ThrottlerModule` có thể được cấu hình như bất kỳ package Nest nào khác với các phương thức `forRoot` hoặc `forRootAsync`.

```typescript
@@filename(app.module)
@Module({
  imports: [
     ThrottlerModule.forRoot({
      throttlers: [
        {
          ttl: 60000,
          limit: 10,
        },
      ],
    }),
  ],
})
export class AppModule {}
```

Điều trên sẽ đặt các tùy chọn toàn cầu cho `ttl`, thời gian sống tính bằng mili-giây, và `limit`, số lượng yêu cầu tối đa trong ttl, cho các route của ứng dụng của bạn được bảo vệ.

Sau khi module đã được import, bạn sau đó có thể chọn cách bạn muốn bind `ThrottlerGuard`. Bất kỳ loại binding nào được đề cập trong phần [guards](https://docs.nestjs.com/guards) đều ổn. Nếu bạn muốn bind guard toàn cầu, ví dụ, bạn có thể làm như vậy bằng cách thêm provider này vào bất kỳ module nào:

```typescript
{
  provide: APP_GUARD,
  useClass: ThrottlerGuard
}
```

#### Multiple Throttler Definitions

Có thể có những lúc bạn muốn thiết lập nhiều định nghĩa throttling, như không quá 3 cuộc gọi trong một giây, 20 cuộc gọi trong 10 giây, và 100 cuộc gọi trong một phút. Để làm điều đó, bạn có thể thiết lập các định nghĩa của mình trong mảng với các tùy chọn được đặt tên, sau đó có thể được tham chiếu trong các decorator `@SkipThrottle()` và `@Throttle()` để thay đổi các tùy chọn một lần nữa.

```typescript
@@filename(app.module)
@Module({
  imports: [
    ThrottlerModule.forRoot([
      {
        name: 'short',
        ttl: 1000,
        limit: 3,
      },
      {
        name: 'medium',
        ttl: 10000,
        limit: 20
      },
      {
        name: 'long',
        ttl: 60000,
        limit: 100
      }
    ]),
  ],
})
export class AppModule {}
```

#### Customization

Có thể có lúc bạn muốn bind guard đến một controller hoặc toàn cầu, nhưng muốn vô hiệu hóa rate limiting cho một hoặc nhiều endpoints của mình. Đối với điều đó, bạn có thể sử dụng decorator `@SkipThrottle()`, để phủ nhận throttler cho một class hoàn chỉnh hoặc một route đơn lẻ. Decorator `@SkipThrottle()` cũng có thể nhận vào một đối tượng với các khóa chuỗi và giá trị boolean cho trường hợp bạn muốn loại bỏ _hầu hết_ một controller, nhưng không phải mọi route, và cấu hình nó cho mỗi bộ throttler nếu bạn có nhiều hơn một. Nếu bạn không truyền một đối tượng, mặc định là sử dụng `{{ '{' }} default: true {{ '}' }}`

```typescript
@SkipThrottle()
@Controller('users')
export class UsersController {}
```

Decorator `@SkipThrottle()` này có thể được sử dụng để bỏ qua một route hoặc một class hoặc để phủ nhận việc bỏ qua một route trong một class bị bỏ qua.

```typescript
@SkipThrottle()
@Controller('users')
export class UsersController {
  // Rate limiting is applied to this route.
  @SkipThrottle({ default: false })
  dontSkip() {
    return 'List users work with Rate limiting.';
  }
  // This route will skip rate limiting.
  doSkip() {
    return 'List users work without Rate limiting.';
  }
}
```

Cũng có decorator `@Throttle()` có thể được sử dụng để ghi đè `limit` và `ttl` được đặt trong module toàn cầu, để cung cấp các tùy chọn bảo mật chặt chẽ hơn hoặc lỏng lẻo hơn. Decorator này có thể được sử dụng trên một class hoặc một function. Với phiên bản 5 trở đi, decorator nhận vào một đối tượng với chuỗi liên quan đến tên của bộ throttler, và một đối tượng với các khóa limit và ttl và giá trị số nguyên, tương tự như các tùy chọn được truyền đến module gốc. Nếu bạn không có tên được đặt trong các tùy chọn gốc của mình, sử dụng chuỗi `default`. Bạn phải cấu hình nó như sau:

```typescript
// Override default configuration for Rate limiting and duration.
@Throttle({ default: { limit: 3, ttl: 60000 } })
@Get()
findAll() {
  return "List users works with custom rate limiting.";
}
```

#### Proxies

Nếu ứng dụng của bạn đang chạy phía sau một proxy server, điều cần thiết là cấu hình HTTP adapter để tin tưởng proxy. Bạn có thể tham khảo các tùy chọn HTTP adapter cụ thể cho [Express](http://expressjs.com/en/guide/behind-proxies.html) và [Fastify](https://www.fastify.io/docs/latest/Reference/Server/#trustproxy) để bật cài đặt `trust proxy`.

Dưới đây là một ví dụ chứng minh cách bật `trust proxy` cho Express adapter:

```typescript
@@filename(main)
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { NestExpressApplication } from '@nestjs/platform-express';

async function bootstrap() {
  const app = await NestFactory.create<NestExpressApplication>(AppModule);
  app.set('trust proxy', 'loopback'); // Trust requests from the loopback address
  await app.listen(3000);
}

bootstrap();
@@switch
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { NestExpressApplication } from '@nestjs/platform-express';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.set('trust proxy', 'loopback'); // Trust requests from the loopback address
  await app.listen(3000);
}

bootstrap();
```

Bật `trust proxy` cho phép bạn truy xuất địa chỉ IP gốc từ header `X-Forwarded-For`. Bạn cũng có thể tùy chỉnh hành vi của ứng dụng của mình bằng cách ghi đè phương thức `getTracker()` để trích xuất địa chỉ IP từ header này thay vì dựa vào `req.ip`. Ví dụ sau chứng minh cách đạt được điều này cho cả Express và Fastify:

```typescript
@@filename(throttler-behind-proxy.guard)
import { ThrottlerGuard } from '@nestjs/throttler';
import { Injectable } from '@nestjs/common';

@Injectable()
export class ThrottlerBehindProxyGuard extends ThrottlerGuard {
  protected async getTracker(req: Record<string, any>): Promise<string> {
    return req.ips.length ? req.ips[0] : req.ip; // individualize IP extraction to meet your own needs
  }
}
```

> info **Gợi ý** Bạn có thể tìm thấy API của đối tượng Request `req` cho express [ở đây](https://expressjs.com/en/api.html#req.ips) và cho fastify [ở đây](https://www.fastify.io/docs/latest/Reference/Request/).

#### Websockets

Module này có thể hoạt động với websockets, nhưng nó yêu cầu một số mở rộng class. Bạn có thể mở rộng `ThrottlerGuard` và ghi đè phương thức `handleRequest` như sau:

```typescript
@Injectable()
export class WsThrottlerGuard extends ThrottlerGuard {
  async handleRequest(requestProps: ThrottlerRequest): Promise<boolean> {
    const {
      context,
      limit,
      ttl,
      throttler,
      blockDuration,
      getTracker,
      generateKey,
    } = requestProps;

    const client = context.switchToWs().getClient();
    const tracker = client._socket.remoteAddress;
    const key = generateKey(context, tracker, throttler.name);
    const { totalHits, timeToExpire, isBlocked, timeToBlockExpire } =
      await this.storageService.increment(
        key,
        ttl,
        limit,
        blockDuration,
        throttler.name,
      );

    const getThrottlerSuffix = (name: string) =>
      name === 'default' ? '' : `-${name}`;

    // Throw an error when the user reached their limit.
    if (isBlocked) {
      await this.throwThrottlingException(context, {
        limit,
        ttl,
        key,
        tracker,
        totalHits,
        timeToExpire,
        isBlocked,
        timeToBlockExpire,
      });
    }

    return true;
  }
}
```

> info **Gợi ý** Nếu bạn đang sử dụng ws, cần thiết phải thay thế `_socket` bằng `conn`

Có một vài điều cần lưu ý khi làm việc với WebSockets:

- Guard không thể được đăng ký với `APP_GUARD` hoặc `app.useGlobalGuards()`
- Khi đạt đến giới hạn, Nest sẽ phát ra một sự kiện `exception`, vì vậy hãy đảm bảo có một listener sẵn sàng cho điều này

> info **Gợi ý** Nếu bạn đang sử dụng package `@nestjs/platform-ws`, bạn có thể sử dụng `client._socket.remoteAddress` thay thế.

> info **Gợi ý** Khi bạn cấu hình [nhiều định nghĩa throttler](/security/rate-limiting#multiple-throttler-definitions), `handleRequest()` chạy một lần cho mỗi bộ throttler. Sử dụng `throttler.name` từ `ThrottlerRequest` khi tạo khóa lưu trữ và khi báo cáo `ThrottlerLimitDetail`, như được trình bày ở trên, để mỗi throttler được đặt tên theo dõi giới hạn của riêng nó.

#### GraphQL

`ThrottlerGuard` cũng có thể được sử dụng để hoạt động với các yêu cầu GraphQL. Một lần nữa, guard có thể được mở rộng, nhưng lần này phương thức `getRequestResponse` sẽ được ghi đè

```typescript
@Injectable()
export class GqlThrottlerGuard extends ThrottlerGuard {
  getRequestResponse(context: ExecutionContext) {
    const gqlCtx = GqlExecutionContext.create(context);
    const ctx = gqlCtx.getContext();
    return { req: ctx.req, res: ctx.res };
  }
}
```

#### Configuration

Các tùy chọn sau đây là hợp lệ cho đối tượng được truyền đến mảng các tùy chọn của `ThrottlerModule`:

<table>
  <tr>
    <td><code>name</code></td>
    <td>tên để theo dõi nội bộ bộ throttler nào đang được sử dụng. Mặc định là <code>default</code> nếu không được truyền</td>
  </tr>
  <tr>
    <td><code>ttl</code></td>
    <td>số mili-giây mà mỗi yêu cầu sẽ kéo dài trong lưu trữ</td>
  </tr>
  <tr>
    <td><code>limit</code></td>
    <td>số lượng yêu cầu tối đa trong giới hạn TTL</td>
  </tr>
  <tr>
    <td><code>blockDuration</code></td>
    <td>số mili-giây mà yêu cầu sẽ bị chặn cho thời gian đó</td>
  </tr>
  <tr>
    <td><code>ignoreUserAgents</code></td>
    <td>mảng các biểu thức chính quy của user-agents để bỏ qua khi liên quan đến việc throttling các yêu cầu</td>
  </tr>
  <tr>
    <td><code>skipIf</code></td>
    <td>một hàm nhận vào <code>ExecutionContext</code> và trả về một <code>boolean</code> để short circuit logic throttler. Giống <code>@SkipThrottle()</code>, nhưng dựa trên yêu cầu</td>
  </tr>
</table>

Nếu bạn cần thiết lập lưu trữ thay thế, hoặc muốn sử dụng một số tùy chọn ở trên theo ý nghĩa toàn cầu hơn, áp dụng cho mỗi bộ throttler, bạn có thể truyền các tùy chọn ở trên thông qua khóa tùy chọn `throttlers` và sử dụng bảng dưới đây

<table>
  <tr>
    <td><code>storage</code></td>
    <td>một dịch vụ lưu trữ tùy chỉnh cho nơi throttling nên được theo dõi. <a href="/security/rate-limiting#storages">Xem ở đây.</a></td>
  </tr>
  <tr>
    <td><code>ignoreUserAgents</code></td>
    <td>mảng các biểu thức chính quy của user-agents để bỏ qua khi liên quan đến việc throttling các yêu cầu</td>
  </tr>
  <tr>
    <td><code>skipIf</code></td>
    <td>một hàm nhận vào <code>ExecutionContext</code> và trả về một <code>boolean</code> để short circuit logic throttler. Giống <code>@SkipThrottle()</code>, nhưng dựa trên yêu cầu</td>
  </tr>
  <tr>
    <td><code>throttlers</code></td>
    <td>mảng các bộ throttler, được định nghĩa bằng cách sử dụng bảng trên</td>
  </tr>
  <tr>
    <td><code>errorMessage</code></td>
    <td>một <code>string</code> HOẶC một hàm nhận vào <code>ExecutionContext</code> và <code>ThrottlerLimitDetail</code> và trả về một <code>string</code> ghi đè thông báo lỗi throttler mặc định</td>
  </tr>
  <tr>
    <td><code>getTracker</code></td>
    <td>một hàm nhận vào <code>Request</code> và trả về một <code>string</code> để ghi đè logic mặc định của phương thức <code>getTracker</code></td>
  </tr>
  <tr>
    <td><code>generateKey</code></td>
    <td>một hàm nhận vào <code>ExecutionContext</code>, chuỗi tracker <code>string</code> và tên throttler như một <code>string</code> và trả về một <code>string</code> để ghi đè khóa cuối cùng sẽ được sử dụng để lưu trữ giá trị rate limit. Điều này ghi đè logic mặc định của phương thức <code>generateKey</code></td>
  </tr>
</table>

#### Async Configuration

Bạn có thể muốn lấy cấu hình rate-limiting của mình một cách không đồng bộ thay vì đồng bộ. Bạn có thể sử dụng phương thức `forRootAsync()`, cho phép dependency injection và các phương thức `async`.

Một cách tiếp cận sẽ là sử dụng một hàm factory:

```typescript
@Module({
  imports: [
    ThrottlerModule.forRootAsync({
      imports: [ConfigModule],
      inject: [ConfigService],
      useFactory: (config: ConfigService) => [
        {
          ttl: config.get('THROTTLE_TTL'),
          limit: config.get('THROTTLE_LIMIT'),
        },
      ],
    }),
  ],
})
export class AppModule {}
```

Bạn cũng có thể sử dụng cú pháp `useClass`:

```typescript
@Module({
  imports: [
    ThrottlerModule.forRootAsync({
      imports: [ConfigModule],
      useClass: ThrottlerConfigService,
    }),
  ],
})
export class AppModule {}
```

Điều này có thể thực hiện được, miễn là `ThrottlerConfigService` triển khai interface `ThrottlerOptionsFactory`.

#### Storages

Lưu trữ tích hợp sẵn là một cache trong bộ nhớ theo dõi các yêu cầu được thực hiện cho đến khi chúng vượt qua TTL được đặt bởi các tùy chọn toàn cầu. Bạn có thể thêm vào tùy chọn lưu trữ của riêng mình vào tùy chọn `storage` của `ThrottlerModule` miễn là class triển khai interface `ThrottlerStorage`.

Đối với các máy chủ phân tán, bạn có thể sử dụng nhà cung cấp lưu trữ cộng đồng cho [Redis](https://github.com/jmcdo29/nest-lab/tree/main/packages/throttler-storage-redis) để có một nguồn sự thật duy nhất.

> info **Lưu ý** `ThrottlerStorage` có thể được nhập từ `@nestjs/throttler`.

#### Time Helpers

Có một vài phương thức helper để làm cho các thời gian dễ đọc hơn nếu bạn thích sử dụng chúng hơn định nghĩa trực tiếp. `@nestjs/throttler` xuất năm helper khác nhau, `seconds`, `minutes`, `hours`, `days`, và `weeks`. Để sử dụng chúng, chỉ cần gọi `seconds(5)` hoặc bất kỳ helper nào khác, và số mili-giây chính xác sẽ được trả về.

#### Migration Guide

Đối với hầu hết mọi người, việc bọc các tùy chọn của bạn trong một mảng sẽ đủ.

Nếu bạn đang sử dụng một lưu trữ tùy chỉnh, bạn nên bọc `ttl` và `limit` của mình trong một
mảng và gán nó cho thuộc tính `throttlers` của đối tượng tùy chọn.

Bất kỳ decorator `@SkipThrottle()` nào có thể được sử dụng để bỏ qua throttling cho các routes hoặc methods cụ thể. Nó chấp nhận một tham số boolean tùy chọn, mặc định là `true`. Điều này hữu ích khi bạn muốn bỏ qua rate limiting trên các endpoints cụ thể.

Bất kỳ decorator `@Throttle()` nào cũng bây giờ nên nhận vào một đối tượng với các khóa chuỗi,
liên quan đến tên của các bối cảnh throttler (một lần nữa, `'default'` nếu không có tên)
và giá trị của các đối tượng có các khóa `limit` và `ttl`.

> Warning **Quan trọng** `ttl` bây giờ tính bằng **mili-giây**. Nếu bạn muốn giữ ttl
> của mình tính bằng giây để dễ đọc, hãy sử dụng helper `seconds` từ package này. Nó chỉ
> nhân ttl với 1000 để làm cho nó tính bằng mili-giây.

Để biết thêm thông tin, hãy xem [Changelog](https://github.com/nestjs/throttler/blob/master/CHANGELOG.md#500)