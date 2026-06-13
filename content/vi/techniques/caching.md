### Caching

Caching là một kỹ thuật mạnh mẽ và đơn giản để nâng cao hiệu suất của ứng dụng. Bằng cách hoạt động như một lớp lưu trữ tạm thời, nó cho phép truy cập nhanh vào dữ liệu được sử dụng thường xuyên, giảm nhu cầu fetch hoặc tính toán lại cùng một thông tin nhiều lần. Điều này dẫn đến thời gian phản hồi nhanh hơn và hiệu quả tổng thể được cải thiện.

#### Cài đặt

Để bắt đầu với caching trong Nest, bạn cần cài đặt package `@nestjs/cache-manager` cùng với package `cache-manager`.

```bash
$ npm install @nestjs/cache-manager cache-manager
```

Theo mặc định, mọi thứ được lưu trữ trong bộ nhớ; Vì `cache-manager` sử dụng [Keyv](https://keyv.org/docs/) dưới lớp vỏ, bạn có thể dễ dàng chuyển sang giải pháp lưu trữ nâng cao hơn, chẳng hạn như Redis, bằng cách cài đặt package phù hợp. Chúng tôi sẽ đề cập chi tiết hơn về điều này sau.

#### Cache trong bộ nhớ

Để bật caching trong ứng dụng của bạn, nhập `CacheModule` và cấu hình nó bằng phương thức `register()`:

```typescript
import { Module } from '@nestjs/common';
import { CacheModule } from '@nestjs/cache-manager';
import { AppController } from './app.controller';

@Module({
  imports: [CacheModule.register()],
  controllers: [AppController],
})
export class AppModule {}
```

Thiết lập này khởi tạo caching trong bộ nhớ với các cài đặt mặc định, cho phép bạn bắt đầu cache dữ liệu ngay lập tức.

#### Tương tác với Cache store

Để tương tác với instance cache manager, inject nó vào class của bạn bằng token `CACHE_MANAGER`, như sau:

```typescript
constructor(@Inject(CACHE_MANAGER) private cacheManager: Cache) {}
```

> info **Gợi ý** Class `Cache` và token `CACHE_MANAGER` đều được nhập từ package `@nestjs/cache-manager`.

Phương thức `get` trên instance `Cache` (từ package `cache-manager`) được sử dụng để truy xuất các mục từ cache. Nếu mục không tồn tại trong cache, `undefined` sẽ được trả về (trong `cache-manager` v6 và các phiên bản trước đó, `null` được trả về thay thế). Cả hai đều được coi là falsy khi di chuyển.

```typescript
const value = await this.cacheManager.get('key');
```

Để thêm một mục vào cache, sử dụng phương thức `set`:

```typescript
await this.cacheManager.set('key', 'value');
```

> warning **Lưu ý** Cache lưu trữ trong bộ nhớ chỉ có thể lưu trữ các giá trị của các loại được hỗ trợ bởi [structured clone algorithm](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Structured_clone_algorithm#javascript_types).

Bạn có thể chỉ định thủ công một TTL (thời gian hết hạn tính bằng mili giây) cho key cụ thể này, như sau:

```typescript
await this.cacheManager.set('key', 'value', 1000);
```

Trong đó `1000` là TTL tính bằng mili giây - trong trường hợp này, mục cache sẽ hết hạn sau một giây.

Để tắt hết hạn của cache, đặt thuộc tính cấu hình `ttl` thành `0`:

```typescript
await this.cacheManager.set('key', 'value', 0);
```

Để xóa một mục khỏi cache, sử dụng phương thức `del`:

```typescript
await this.cacheManager.del('key');
```

Để xóa toàn bộ cache, sử dụng phương thức `clear`:

```typescript
await this.cacheManager.clear();
```

#### Tự động cache phản hồi

> warning **Cảnh báo** Trong ứng dụng [GraphQL](/graphql/quick-start), các interceptor được thực thi riêng biệt cho từng field resolver. Do đó, `CacheModule` (sử dụng interceptors để cache phản hồi) sẽ không hoạt động chính xác.

Để bật tự động cache phản hồi, chỉ cần gắn `CacheInterceptor` vào nơi bạn muốn cache dữ liệu.

```typescript
@Controller()
@UseInterceptors(CacheInterceptor)
export class AppController {
  @Get()
  findAll(): string[] {
    return [];
  }
}
```

> warning**Cảnh báo** Chỉ các endpoint `GET` được cache. Ngoài ra, các route HTTP server inject đối tượng response gốc (`@Res()`) không thể sử dụng Cache Interceptor. Xem
> <a href="https://docs.nestjs.com/interceptors#response-mapping">response mapping</a> để biết thêm chi tiết.

Để giảm lượng boilerplate cần thiết, bạn có thể bind `CacheInterceptor` vào tất cả các endpoint toàn cục:

```typescript
import { Module } from '@nestjs/common';
import { CacheModule, CacheInterceptor } from '@nestjs/cache-manager';
import { AppController } from './app.controller';
import { APP_INTERCEPTOR } from '@nestjs/core';

@Module({
  imports: [CacheModule.register()],
  controllers: [AppController],
  providers: [
    {
      provide: APP_INTERCEPTOR,
      useClass: CacheInterceptor,
    },
  ],
})
export class AppModule {}
```

#### Time-to-live (TTL)

Giá trị mặc định cho `ttl` là `0`, có nghĩa là cache sẽ không bao giờ hết hạn. Để chỉ định [TTL](https://en.wikipedia.org/wiki/Time_to_live) tùy chỉnh, bạn có thể cung cấp tùy chọn `ttl` trong phương thức `register()`, như minh họa dưới đây:

```typescript
CacheModule.register({
  ttl: 5000, // milliseconds
});
```

#### Sử dụng module toàn cục

Khi bạn muốn sử dụng `CacheModule` trong các module khác, bạn sẽ cần nhập nó (như tiêu chuẩn với bất kỳ module Nest nào). Ngoài ra, khai báo nó là [global module](https://docs.nestjs.com/modules#global-modules) bằng cách đặt thuộc tính `isGlobal` của đối tượng tùy chọn thành `true`, như hiển thị bên dưới. Trong trường hợp đó, bạn sẽ không cần nhập `CacheModule` trong các module khác sau khi nó đã được tải trong module gốc (ví dụ, `AppModule`).

```typescript
CacheModule.register({
  isGlobal: true,
});
```

#### Ghi đè cache toàn cục

Trong khi cache toàn cục được bật, các mục cache được lưu trữ dưới một `CacheKey` được tự động tạo dựa trên đường dẫn route. Bạn có thể ghi đè một số cài đặt cache (`@CacheKey()` và `@CacheTTL()`) trên cơ sở từng phương thức, cho phép các chiến lược cache tùy chỉnh cho từng phương thức controller riêng lẻ. Điều này có thể phù hợp nhất khi sử dụng [các cache store khác nhau.](https://docs.nestjs.com/techniques/caching#different-stores)

Bạn có thể áp dụng decorator `@CacheTTL()` trên cơ sở từng controller để đặt TTL cache cho toàn bộ controller. Trong trường hợp cả hai cài đặt TTL cache ở cấp controller và cấp phương thức được định nghĩa, cài đặt TTL cache được chỉ định ở cấp phương thức sẽ được ưu tiên hơn các cài đặt ở cấp controller.

```typescript
@Controller()
@CacheTTL(50)
export class AppController {
  @CacheKey('custom_key')
  @CacheTTL(20)
  findAll(): string[] {
    return [];
  }
}
```

> info **Gợi ý** Các decorator `@CacheKey()` và `@CacheTTL()` được nhập từ package `@nestjs/cache-manager`.

Decorator `@CacheKey()` có thể được sử dụng với hoặc không có decorator `@CacheTTL()` tương ứng và ngược lại. Bạn có thể chọn chỉ ghi đè `@CacheKey()` hoặc chỉ ghi đè `@CacheTTL()`. Các cài đặt không được ghi đè bằng decorator sẽ sử dụng các giá trị mặc định như đã đăng ký toàn cục (xem [Tùy chỉnh caching](https://docs.nestjs.com/techniques/caching#customize-caching)).

#### WebSockets và Microservices

Bạn cũng có thể áp dụng `CacheInterceptor` cho WebSocket subscribers cũng như các pattern của Microservice (bất kể phương thức transport đang được sử dụng).

```typescript
@@filename()
@CacheKey('events')
@UseInterceptors(CacheInterceptor)
@SubscribeMessage('events')
handleEvent(client: Client, data: string[]): Observable<string[]> {
  return [];
}
@@switch
@CacheKey('events')
@UseInterceptors(CacheInterceptor)
@SubscribeMessage('events')
handleEvent(client, data) {
  return [];
}
```

Tuy nhiên, decorator bổ sung `@CacheKey()` là bắt buộc để chỉ định một key được sử dụng để sau đó lưu trữ và truy xuất dữ liệu cache. Ngoài ra, lưu ý rằng bạn **không nên cache mọi thứ**. Các hành động thực hiện một số hoạt động kinh doanh thay vì chỉ truy vấn dữ liệu không bao giờ nên được cache.

Ngoài ra, bạn có thể chỉ định thời gian hết hạn cache (TTL) bằng cách sử dụng decorator `@CacheTTL()`, điều này sẽ ghi đè giá trị TTL mặc định toàn cục.

```typescript
@@filename()
@CacheTTL(10)
@UseInterceptors(CacheInterceptor)
@SubscribeMessage('events')
handleEvent(client: Client, data: string[]): Observable<string[]> {
  return [];
}
@@switch
@CacheTTL(10)
@UseInterceptors(CacheInterceptor)
@SubscribeMessage('events')
handleEvent(client, data) {
  return [];
}
```

> info **Gợi ý** Decorator `@CacheTTL()` có thể được sử dụng với hoặc không có decorator `@CacheKey()` tương ứng.

#### Điều chỉnh tracking

Theo mặc định, Nest sử dụng URL request (trong ứng dụng HTTP) hoặc cache key (trong ứng dụng websockets và microservices, được đặt thông qua decorator `@CacheKey()`) để liên kết các bản ghi cache với các endpoint của bạn. Tuy nhiên, đôi khi bạn có thể muốn thiết lập tracking dựa trên các yếu tố khác, ví dụ, sử dụng HTTP headers (ví dụ: `Authorization` để xác định đúng các endpoint `profile`).

Để thực hiện điều đó, tạo một subclass của `CacheInterceptor` và ghi đè phương thức `trackBy()`.

```typescript
@Injectable()
class HttpCacheInterceptor extends CacheInterceptor {
  trackBy(context: ExecutionContext): string | undefined {
    return 'key';
  }
}
```

#### Sử dụng Cache store thay thế

Chuyển sang cache store khác rất đơn giản. Đầu tiên, cài đặt package phù hợp. Ví dụ, để sử dụng Redis, cài đặt package `@keyv/redis`:

```bash
$ npm install @keyv/redis
```

Với điều này, bạn có thể đăng ký `CacheModule` với nhiều store như hiển thị dưới đây:

```typescript
import { Module } from '@nestjs/common';
import { CacheModule } from '@nestjs/cache-manager';
import { AppController } from './app.controller';
import KeyvRedis from '@keyv/redis';
import { Keyv } from 'keyv';
import { KeyvCacheableMemory } from 'cacheable';

@Module({
  imports: [
    CacheModule.registerAsync({
      useFactory: async () => {
        return {
          stores: [
            new Keyv({
              store: new KeyvCacheableMemory({ ttl: 60000, lruSize: 5000 }),
            }),
            new KeyvRedis('redis://localhost:6379'),
          ],
        };
      },
    }),
  ],
  controllers: [AppController],
})
export class AppModule {}
```

Trong ví dụ này, chúng tôi đã đăng ký hai store: `CacheableMemory` và `KeyvRedis`. Store `CacheableMemory` là một store trong bộ nhớ đơn giản, được tạo thông qua bộ điều hợp lưu trữ `KeyvCacheableMemory`, trong khi `KeyvRedis` là một store Redis. Mảng `stores` được sử dụng để chỉ định các store bạn muốn sử dụng. Store đầu tiên trong mảng là store mặc định, và phần còn lại là các store fallback.

Xem [tài liệu Keyv](https://keyv.org/docs/) để biết thêm thông tin về các store có sẵn.

#### Cấu hình async

Bạn có thể muốn truyền tùy chọn module một cách async thay vì truyền tĩnh tại thời điểm biên dịch. Trong trường hợp này, sử dụng phương thức `registerAsync()`, cung cấp một số cách để xử lý cấu hình async.

Một cách tiếp cận là sử dụng factory function:

```typescript
CacheModule.registerAsync({
  useFactory: () => ({
    ttl: 5,
  }),
});
```

Factory của chúng tôi hoạt động giống như tất cả các factory module async khác (nó có thể là `async` và có thể inject dependencies thông qua `inject`).

```typescript
CacheModule.registerAsync({
  imports: [ConfigModule],
  useFactory: async (configService: ConfigService) => ({
    ttl: configService.get('CACHE_TTL'),
  }),
  inject: [ConfigService],
});
```

Ngoài ra, bạn có thể sử dụng phương thức `useClass`:

```typescript
CacheModule.registerAsync({
  useClass: CacheConfigService,
});
```

Cấu trúc trên sẽ khởi tạo `CacheConfigService` bên trong `CacheModule` và sử dụng nó để lấy đối tượng tùy chọn. `CacheConfigService` phải implement interface `CacheOptionsFactory` để cung cấp các tùy chọn cấu hình:

```typescript
@Injectable()
class CacheConfigService implements CacheOptionsFactory {
  createCacheOptions(): CacheModuleOptions {
    return {
      ttl: 5,
    };
  }
}
```

Nếu bạn muốn sử dụng một provider cấu hình hiện có được nhập từ một module khác, sử dụng cú pháp `useExisting`:

```typescript
CacheModule.registerAsync({
  imports: [ConfigModule],
  useExisting: ConfigService,
});
```

Điều này hoạt động giống như `useClass` với một sự khác biệt quan trọng - `CacheModule` sẽ tìm kiếm các module đã nhập để tái sử dụng bất kỳ `ConfigService` nào đã được tạo, thay vì khởi tạo riêng của nó.

> info **Gợi ý** `CacheModule#register`, `CacheModule#registerAsync` và `CacheOptionsFactory` có một generic (tham số kiểu) tùy chọn để thu hẹp các tùy chọn cấu hình cụ thể cho store, làm cho nó type safe.

Bạn cũng có thể truyền các `extraProviders` được gọi vào phương thức `registerAsync()`. Các providers này sẽ được hợp nhất với các providers của module.

```typescript
CacheModule.registerAsync({
  imports: [ConfigModule],
  useClass: ConfigService,
  extraProviders: [MyAdditionalProvider],
});
```

Điều này hữu ích khi bạn muốn cung cấp các dependencies bổ sung cho factory function hoặc constructor của class.

#### Ví dụ

Một ví dụ hoạt động có sẵn [tại đây](https://github.com/nestjs/nest/tree/master/sample/20-cache).