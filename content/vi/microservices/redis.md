### Redis

Transporter [Redis](https://redis.io/) triển khai paradigm nhắn tin publish/subscribe và tận dụng tính năng [Pub/Sub](https://redis.io/topics/pubsub) của Redis. Các tin nhắn được xuất bản được phân loại trong các kênh, mà không biết những subscribers nào (nếu có) cuối cùng sẽ nhận tin nhắn. Mỗi microservice có thể subscribe vào bất kỳ số lượng kênh nào. Ngoài ra, nhiều hơn một kênh có thể được subscribe cùng một lúc. Các tin nhắn được trao đổi qua các kênh là **fire-and-forget**, nghĩa là nếu một tin nhắn được xuất bản và không có subscribers quan tâm đến nó, tin nhắn bị xóa và không thể được phục hồi. Do đó, bạn không có sự đảm bảo rằng các tin nhắn hoặc sự kiện sẽ được xử lý bởi ít nhất một dịch vụ. Một tin nhắn đơn có thể được subscribe vào (và nhận) bởi nhiều subscribers.

<figure><img class="illustrative-image" src="/assets/Redis_1.png" /></figure>

#### Cài đặt

Để bắt đầu xây dựng microservices dựa trên Redis, trước tiên hãy cài đặt gói cần thiết:

```bash
$ npm i --save ioredis
```

#### Tổng quan

Để sử dụng transporter Redis, truyền đối tượng options sau đến phương thức `createMicroservice()`:

```typescript
@@filename(main)
const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  transport: Transport.REDIS,
  options: {
    host: 'localhost',
    port: 6379,
  },
});
@@switch
const app = await NestFactory.createMicroservice(AppModule, {
  transport: Transport.REDIS,
  options: {
    host: 'localhost',
    port: 6379,
  },
});
```

> info **Gợi ý** Enum `Transport` được import từ gói `@nestjs/microservices`.

#### Tùy chọn

Thuộc tính `options` là cụ thể cho transporter được chọn. Transporter <strong>Redis</strong> expose các thuộc tính được mô tả dưới đây.

<table>
  <tr>
    <td><code>host</code></td>
    <td>Url kết nối</td>
  </tr>
  <tr>
    <td><code>port</code></td>
    <td>Port kết nối</td>
  </tr>
  <tr>
    <td><code>retryAttempts</code></td>
    <td>Số lần thử lại tin nhắn (mặc định: <code>0</code>)</td>
  </tr>
  <tr>
    <td><code>retryDelay</code></td>
    <td>Độ trễ giữa các lần thử lại tin nhắn (ms) (mặc định: <code>0</code>)</td>
  </tr>
  <tr>
    <td><code>wildcards</code></td>
    <td>Kích hoạt subscriptions wildcard Redis, hướng dẫn transporter sử dụng <code>psubscribe</code>/<code>pmessage</code> bên dưới. (mặc định: <code>false</code>)</td>
  </tr>
</table>

Tất cả các thuộc tính được hỗ trợ bởi client [ioredis](https://redis.github.io/ioredis/index.html#RedisOptions) chính thức cũng được hỗ trợ bởi transporter này.

#### Client

Giống như các transporter microservice khác, bạn có <a href="https://docs.nestjs.com/microservices/basics#client">nhiều tùy chọn</a> để tạo một thể hiện Redis `ClientProxy`.

Một phương pháp để tạo một thể hiện là sử dụng `ClientsModule`. Để tạo một thể hiện client với `ClientsModule`, import nó và sử dụng phương thức `register()` để truyền một đối tượng options với các thuộc tính giống nhau được hiển thị ở trên trong phương thức `createMicroservice()`, cũng như một thuộc tính `name` để được sử dụng như injection token. Đọc thêm về `ClientsModule` <a href="https://docs.nestjs.com/microservices/basics#client">ở đây</a>.

```typescript
@Module({
  imports: [
    ClientsModule.register([
      {
        name: 'MATH_SERVICE',
        transport: Transport.REDIS,
        options: {
          host: 'localhost',
          port: 6379,
        }
      },
    ]),
  ]
  ...
})
```

Các tùy chọn khác để tạo một client (hoặc `ClientProxyFactory` hoặc `@Client()`) cũng có thể được sử dụng. Bạn có thể đọc về chúng <a href="https://docs.nestjs.com/microservices/basics#client">ở đây</a>.

#### Context

Trong các kịch bản phức tạp hơn, bạn có thể cần truy cập thông tin bổ sung về yêu cầu đến. Khi sử dụng transporter Redis, bạn có thể truy cập đối tượng `RedisContext`.

```typescript
@@filename()
@MessagePattern('notifications')
getNotifications(@Payload() data: number[], @Ctx() context: RedisContext) {
  console.log(`Channel: ${context.getChannel()}`);
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('notifications')
getNotifications(data, context) {
  console.log(`Channel: ${context.getChannel()}`);
}
```

> info **Gợi ý** `@Payload()`, `@Ctx()` và `RedisContext` được import từ gói `@nestjs/microservices`.

#### Wildcards

Để kích hoạt hỗ trợ wildcards, đặt tùy chọn `wildcards` thành `true`. Điều này hướng dẫn transporter sử dụng `psubscribe` và `pmessage` bên dưới.

```typescript
const app = await NestFactory.createMicroservice(AppModule, {
  transport: Transport.REDIS,
  options: {
    // Các tùy chọn khác
    wildcards: true,
  },
});
```

Đảm bảo truyền tùy chọn `wildcards` khi tạo một thể hiện client cũng vậy.

Với tùy chọn này được kích hoạt, bạn có thể sử dụng wildcards trong các pattern tin nhắn và sự kiện của mình. Ví dụ, để subscribe vào tất cả các kênh bắt đầu bằng `notifications`, bạn có thể sử dụng pattern sau:

```typescript
@EventPattern('notifications.*')
```

#### Cập nhật trạng thái thể hiện

Để nhận cập nhật thời gian thực về kết nối và trạng thái của thể hiện driver cơ bản, bạn có thể subscribe vào stream `status`. Stream này cung cấp các cập nhật trạng thái cụ thể cho driver được chọn. Đối với driver Redis, stream `status` emit các sự kiện `connected`, `disconnected`, và `reconnecting`.

```typescript
this.client.status.subscribe((status: RedisStatus) => {
  console.log(status);
});
```

> info **Gợi ý** Kiểu `RedisStatus` được import từ gói `@nestjs/microservices`.

Tương tự, bạn có thể subscribe vào stream `status` của server để nhận thông báo về trạng thái của server.

```typescript
const server = app.connectMicroservice<MicroserviceOptions>(...);
server.status.subscribe((status: RedisStatus) => {
  console.log(status);
});
```

#### Lắng nghe các sự kiện Redis

Trong một số trường hợp, bạn có thể muốn lắng nghe các sự kiện nội bộ được emit bởi microservice. Ví dụ, bạn có thể lắng nghe sự kiện `error` để kích hoạt các thao tác bổ sung khi xảy ra lỗi. Để làm điều này, sử dụng phương thức `on()`, như được hiển thị dưới đây:

```typescript
this.client.on('error', (err) => {
  console.error(err);
});
```

Tương tự, bạn có thể lắng nghe các sự kiện nội bộ của server:

```typescript
server.on<RedisEvents>('error', (err) => {
  console.error(err);
});
```

> info **Gợi ý** Kiểu `RedisEvents` được import từ gói `@nestjs/microservices`.

#### Truy cập driver cơ bản

Đối với các trường hợp sử dụng nâng cao hơn, bạn có thể cần truy cập thể hiện driver cơ bản. Điều này có thể hữu ích cho các kịch bản như đóng thủ công kết nối hoặc sử dụng các phương thức cụ thể của driver. Tuy nhiên, hãy nhớ rằng đối với hầu hết các trường hợp, bạn **không nên cần** truy cập trực tiếp driver.

Để làm điều này, bạn có thể sử dụng phương thức `unwrap()`, trả về thể hiện driver cơ bản. Tham số kiểu generic nên chỉ định loại thể hiện driver bạn mong đợi.

```typescript
const [pub, sub] =
  this.client.unwrap<[import('ioredis').Redis, import('ioredis').Redis]>();
```

Tương tự, bạn có thể truy cập thể hiện driver cơ bản của server:

```typescript
const [pub, sub] =
  server.unwrap<[import('ioredis').Redis, import('ioredis').Redis]>();
```

Lưu ý rằng, trái ngược với các transporter khác, transporter Redis trả về một tuple của hai thể hiện `ioredis`: cái đầu tiên được sử dụng để xuất bản tin nhắn, và cái thứ hai được sử dụng để subscribe vào tin nhắn.