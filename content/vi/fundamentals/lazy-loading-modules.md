### Lazy loading modules

Theo mặc định, modules được eagerly loaded, điều này có nghĩa là ngay khi ứng dụng tải, tất cả các modules cũng được tải, cho dù chúng có cần thiết lập tức hay không. Mặc dù điều này ổn cho hầu hết các ứng dụng, nó có thể trở thành một bottleneck cho các ứng dụng/workers chạy trong **serverless environment**, nơi startup latency ("cold start") là rất quan trọng.

Lazy loading có thể giúp giảm bootstrap time bằng cách chỉ tải các modules cần thiết cho lời gọi serverless function cụ thể. Ngoài ra, bạn cũng có thể tải các modules khác asynchronously khi serverless function đã "warm" để tăng tốc bootstrap time cho các cuộc gọi tiếp theo ngay cả (đăng ký deferred modules).

> info **Gợi ý** Nếu bạn quen với framework **Angular**, bạn có thể đã thấy thuật ngữ "lazy-loading modules" trước đó. Hãy lưu ý rằng kỹ thuật này **khác nhau về mặt chức năng** trong Nest và hãy nghĩ về đây như một tính năng hoàn toàn khác chia sẻ các quy ước đặt tên tương tự.

> warning **Cảnh báo** Lưu ý rằng các methods lifecycle hooks không được invoke trong các lazy loaded modules và services.

#### Getting started

Để tải modules theo yêu cầu, Nest cung cấp class `LazyModuleLoader` có thể được inject vào một class theo cách bình thường:

```typescript
@@filename(cats.service)
@Injectable()
export class CatsService {
  constructor(private lazyModuleLoader: LazyModuleLoader) {}
}
@@switch
@Injectable()
@Dependencies(LazyModuleLoader)
export class CatsService {
  constructor(lazyModuleLoader) {
    this.lazyModuleLoader = lazyModuleLoader;
  }
}
```

> info **Gợi ý** Class `LazyModuleLoader` được import từ package `@nestjs/core`.

Ngoài ra, bạn có thể obtain một tham chiếu đến provider `LazyModuleLoader` từ bên trong file bootstrap ứng dụng của bạn (`main.ts`), như sau:

```typescript
// "app" đại diện cho một instance ứng dụng Nest
const lazyModuleLoader = app.get(LazyModuleLoader);
```

Với điều này ở chỗ, bây giờ bạn có thể tải bất kỳ module nào sử dụng cấu trúc sau:

```typescript
const { LazyModule } = await import('./lazy.module');
const moduleRef = await this.lazyModuleLoader.load(() => LazyModule);
```

> info **Gợi ý** Các modules "Lazy loaded" được **cached** upon lời gọi method `LazyModuleLoader#load` đầu tiên. Điều đó có nghĩa là, mỗi lần thử load `LazyModule` tiếp theo sẽ **rất nhanh** và sẽ trả về một instance được cache, thay vì tải module lại.

> ```bash
> Load "LazyModule" attempt: 1
> time: 2.379ms
> Load "LazyModule" attempt: 2
> time: 0.294ms
> Load "LazyModule" attempt: 3
> time: 0.303ms
> ```

Ngoài ra, các modules "lazy loaded" chia sẻ cùng một modules graph như các eagerly loaded trên bootstrap ứng dụng cũng như bất kỳ lazy modules nào được đăng ký sau trong ứng dụng của bạn.

Trong đó `lazy.module.ts` là một file TypeScript export một **Nest module thường** (không cần thay đổi bổ sung).

Method `LazyModuleLoader#load` trả về module reference (của `LazyModule`) cho phép bạn điều hướng danh sách providers nội bộ và obtain một tham chiếu đến bất kỳ provider nào sử dụng injection token của nó như một lookup key.

Ví dụ, hãy nói rằng chúng ta có một `LazyModule` với định nghĩa sau:

```typescript
@Module({
  providers: [LazyService],
  exports: [LazyService],
})
export class LazyModule {}
```

> info **Gợi ý** Lazy loaded modules không thể được đăng ký như **global modules** vì điều đó đơn giản là không có ý nghĩa (vì chúng được đăng ký lazily, on-demand khi tất cả các modules được đăng ký tĩnh đã được instantiated). Tương tự, các **global enhancers** được đăng ký (guards/interceptors/v.v.) **sẽ không hoạt động** đúng.

Với điều này, chúng ta có thể obtain một tham chiếu đến provider `LazyService`, như sau:

```typescript
const { LazyModule } = await import('./lazy.module');
const moduleRef = await this.lazyModuleLoader.load(() => LazyModule);

const { LazyService } = await import('./lazy.service');
const lazyService = moduleRef.get(LazyService);
```

> warning **Cảnh báo** Nếu bạn sử dụng **Webpack**, hãy đảm bảo cập nhật file `tsconfig.json` của bạn - đặt `compilerOptions.module` thành `"esnext"` và thêm thuộc tính `compilerOptions.moduleResolution` với `"node"` như một giá trị:

> ```json
> {
>   "compilerOptions": {
>     "module": "esnext",
>     "moduleResolution": "node",
>     ...
>   }
> }
> ```

Với các tùy chọn này được thiết lập, bạn sẽ có thể tận dụng tính năng code splitting.

#### Lazy loading controllers, gateways, và resolvers

Vì controllers (hoặc resolvers trong các ứng dụng GraphQL) trong Nest đại diện cho các bộ routes/paths/topics (hoặc queries/mutations), bạn **không thể lazy load chúng** sử dụng class `LazyModuleLoader`.

> error **Cảnh báo** Controllers, resolvers, và gateways được đăng ký bên trong lazy loaded modules sẽ không hoạt động như mong đợi. Tương tự, bạn không thể đăng ký các functions middleware (bằng cách implement interface `MiddlewareConsumer`) on-demand.

Ví dụ, hãy nói rằng bạn đang xây dựng một REST API (ứng dụng HTTP) với driver Fastify dưới mui (sử dụng package `@nestjs/platform-fastify`). Fastify không cho phép bạn đăng ký routes sau khi ứng dụng đã sẵn sàng/successfully lắng nghe các tin nhắn. Điều này có nghĩa là ngay cả khi chúng ta phân tích các route mappings được đăng ký trong controllers của module, tất cả các lazy loaded routes sẽ không thể truy cập được vì không có cách để đăng ký chúng tại runtime.

Tương tự, một số chiến lược transport chúng tôi cung cấp như một phần của package `@nestjs/microservices` (bao gồm Kafka, gRPC, hoặc RabbitMQ) yêu cầu subscribe/lắng nghe các topics/channels cụ thể trước khi kết nối được thiết lập. Một khi ứng dụng của bạn bắt đầu lắng nghe các tin nhắn, framework sẽ không thể subscribe/lắng nghe các topics mới.

Cuối cùng, package `@nestjs/graphql` với cách tiếp cận code first được kích hoạt động tự động tạo GraphQL schema on-the-fly dựa trên metadata. Điều đó có nghĩa là, nó yêu cầu tất cả các classes được tải trước đó. Nếu không, sẽ không thể tạo ra schema phù hợp, hợp lệ.

#### Common use-cases

Phổ biến nhất, bạn sẽ thấy lazy loaded modules trong các tình huống khi worker/cron job/lambda & serverless function/webhook của bạn phải kích hoạt các services khác nhau (logic khác nhau) dựa trên các đối số đầu vào (route path/date/query parameters, v.v.). Mặt khác, lazy loading modules có thể không có quá nhiều ý nghĩa cho các ứng dụng monolithic, nơi startup time khá không liên quan.