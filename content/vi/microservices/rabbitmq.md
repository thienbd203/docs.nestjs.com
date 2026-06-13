### RabbitMQ

[RabbitMQ](https://www.rabbitmq.com/) là một message broker open source và nhẹ hỗ trợ nhiều giao thức nhắn tin. Nó có thể được triển khai trong các cấu hình phân tán và liên kết để đáp ứng các yêu cầu quy mô cao, tính sẵn sàng cao. Ngoài ra, đó là message broker được triển khai rộng rãi nhất, được sử dụng trên toàn thế giới tại các startup nhỏ và các doanh nghiệp lớn.

#### Cài đặt

Để bắt đầu xây dựng microservices dựa trên RabbitMQ, trước tiên hãy cài đặt các gói cần thiết:

```bash
$ npm i --save amqplib amqp-connection-manager
```

#### Tổng quan

Để sử dụng transporter RabbitMQ, truyền đối tượng options sau đến phương thức `createMicroservice()`:

```typescript
@@filename(main)
const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  transport: Transport.RMQ,
  options: {
    urls: ['amqp://localhost:5672'],
    queue: 'cats_queue',
    queueOptions: {
      durable: false
    },
  },
});
@@switch
const app = await NestFactory.createMicroservice(AppModule, {
  transport: Transport.RMQ,
  options: {
    urls: ['amqp://localhost:5672'],
    queue: 'cats_queue',
    queueOptions: {
      durable: false
    },
  },
});
```

> info **Gợi ý** Enum `Transport` được import từ gói `@nestjs/microservices`.

#### Tùy chọn

Thuộc tính `options` là cụ thể cho transporter được chọn. Transporter <strong>RabbitMQ</strong> expose các thuộc tính được mô tả dưới đây.

<table>
  <tr>
    <td><code>urls</code></td>
    <td>Một mảng các URL kết nối để thử theo thứ tự</td>
  </tr>
  <tr>
    <td><code>queue</code></td>
    <td>Tên hàng đợi mà server của bạn sẽ lắng nghe</td>
  </tr>
  <tr>
    <td><code>prefetchCount</code></td>
    <td>Đặt số lượng prefetch cho kênh</td>
  </tr>
  <tr>
    <td><code>isGlobalPrefetchCount</code></td>
    <td>Kích hoạt prefetching theo kênh</td>
  </tr>
  <tr>
    <td><code>noAck</code></td>
    <td>Nếu <code>false</code>, chế độ acknowledgment thủ công được kích hoạt</td>
  </tr>
  <tr>
    <td><code>consumerTag</code></td>
    <td>Một tên mà server sẽ sử dụng để phân biệt các giao hàng tin nhắn cho consumer; không được đang được sử dụng trên kênh. Thường dễ dàng hơn để bỏ qua cái này, trong trường hợp đó server sẽ tạo một tên ngẫu nhiên và cung cấp nó trong phản hồi. Consumer Tag Identifier (đọc thêm <a href="https://amqp-node.github.io/amqplib/channel_api.html#channel_consume" rel="nofollow" target="_blank">ở đây</a>)</td>
  </tr>
  <tr>
    <td><code>queueOptions</code></td>
    <td>Các tùy chọn hàng đợi bổ sung (đọc thêm <a href="https://amqp-node.github.io/amqplib/channel_api.html#channel_assertQueue" rel="nofollow" target="_blank">ở đây</a>)</td>
  </tr>
  <tr>
    <td><code>socketOptions</code></td>
    <td>Các tùy chọn socket bổ sung (đọc thêm <a href="https://amqp-node.github.io/amqplib/channel_api.html#connect" rel="nofollow" target="_blank">ở đây</a>)</td>
  </tr>
  <tr>
    <td><code>headers</code></td>
    <td>Headers được gửi cùng với mọi tin nhắn</td>
  </tr>
  <tr>
    <td><code>replyQueue</code></td>
    <td>Hàng đợi phản hồi cho producer. Mặc định là <code>amq.rabbitmq.reply-to</code></td>
  </tr>
  <tr>
    <td><code>persistent</code></td>
    <td>Nếu truthy, tin nhắn sẽ sống sót qua các lần khởi động lại broker miễn là nó nằm trong một hàng đợi cũng sống sót qua các lần khởi động lại</td>
  </tr>
  <tr>
    <td><code>noAssert</code></td>
    <td>Khi false, một hàng đợi sẽ không được assert trước khi tiêu thụ</td>
  </tr>
  <tr>
    <td><code>wildcards</code></td>
    <td>Đặt thành true chỉ khi bạn muốn sử dụng Topic Exchange để định tuyến tin nhắn đến các hàng đợi. Kích hoạt điều này sẽ cho phép bạn sử dụng wildcards (*, #) như các pattern tin nhắn và sự kiện</td>
  </tr>
  <tr>
    <td><code>exchange</code></td>
    <td>Tên cho exchange. Mặc định là tên hàng đợi khi "wildcards" được đặt thành true</td>
  </tr>
  <tr>
    <td><code>exchangeType</code></td>
    <td>Loại của exchange. Mặc định là <code>topic</code>. Các giá trị hợp lệ là <code>direct</code>, <code>fanout</code>, <code>topic</code>, và <code>headers</code></td>
  </tr>
  <tr>
    <td><code>routingKey</code></td>
    <td>Routing key bổ sung cho topic exchange</td>
  </tr>
  <tr>
    <td><code>maxConnectionAttempts</code></td>
    <td>Số lần tối đa cố gắng kết nối. Chỉ áp dụng cho cấu hình consumer. -1 === vô hạn</td>
  </tr>
</table>

#### Client

Giống như các transporter microservice khác, bạn có <a href="https://docs.nestjs.com/microservices/basics#client">nhiều tùy chọn</a> để tạo một thể hiện RabbitMQ `ClientProxy`.

Một phương pháp để tạo một thể hiện là sử dụng `ClientsModule`. Để tạo một thể hiện client với `ClientsModule`, import nó và sử dụng phương thức `register()` để truyền một đối tượng options với các thuộc tính giống nhau được hiển thị ở trên trong phương thức `createMicroservice()`, cũng như một thuộc tính `name` để được sử dụng như injection token. Đọc thêm về `ClientsModule` <a href="https://docs.nestjs.com/microservices/basics#client">ở đây</a>.

```typescript
@Module({
  imports: [
    ClientsModule.register([
      {
        name: 'MATH_SERVICE',
        transport: Transport.RMQ,
        options: {
          urls: ['amqp://localhost:5672'],
          queue: 'cats_queue',
          queueOptions: {
            durable: false
          },
        },
      },
    ]),
  ]
  ...
})
```

Các tùy chọn khác để tạo một client (hoặc `ClientProxyFactory` hoặc `@Client()`) cũng có thể được sử dụng. Bạn có thể đọc về chúng <a href="https://docs.nestjs.com/microservices/basics#client">ở đây</a>.

#### Context

Trong các kịch bản phức tạp hơn, bạn có thể cần truy cập thông tin bổ sung về yêu cầu đến. Khi sử dụng transporter RabbitMQ, bạn có thể truy cập đối tượng `RmqContext`.

```typescript
@@filename()
@MessagePattern('notifications')
getNotifications(@Payload() data: number[], @Ctx() context: RmqContext) {
  console.log(`Pattern: ${context.getPattern()}`);
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('notifications')
getNotifications(data, context) {
  console.log(`Pattern: ${context.getPattern()}`);
}
```

> info **Gợi ý** `@Payload()`, `@Ctx()` và `RmqContext` được import từ gói `@nestjs/microservices`.

Để truy cập tin nhắn RabbitMQ gốc (với `properties`, `fields`, và `content`), sử dụng phương thức `getMessage()` của đối tượng `RmqContext`, như sau:

```typescript
@@filename()
@MessagePattern('notifications')
getNotifications(@Payload() data: number[], @Ctx() context: RmqContext) {
  console.log(context.getMessage());
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('notifications')
getNotifications(data, context) {
  console.log(context.getMessage());
}
```

Để truy xuất một tham chiếu đến [channel](https://www.rabbitmq.com/channels.html) RabbitMQ, sử dụng phương thức `getChannelRef` của đối tượng `RmqContext`, như sau:

```typescript
@@filename()
@MessagePattern('notifications')
getNotifications(@Payload() data: number[], @Ctx() context: RmqContext) {
  console.log(context.getChannelRef());
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('notifications')
getNotifications(data, context) {
  console.log(context.getChannelRef());
}
```

#### Acknowledgment tin nhắn

Để đảm bảo một tin nhắn không bao giờ bị mất, RabbitMQ hỗ trợ [message acknowledgements](https://www.rabbitmq.com/confirms.html). Một acknowledgment được gửi lại bởi consumer để nói với RabbitMQ rằng một tin nhắn cụ thể đã được nhận, xử lý và rằng RabbitMQ tự do xóa nó. Nếu một consumer chết (kênh của nó được đóng, kết nối được đóng, hoặc kết nối TCP bị mất) mà không gửi một ack, RabbitMQ sẽ hiểu rằng một tin nhắn không được xử lý đầy đủ và sẽ re-queue nó.

Để kích hoạt chế độ acknowledgment thủ công, đặt thuộc tính `noAck` thành `false`:

```typescript
options: {
  urls: ['amqp://localhost:5672'],
  queue: 'cats_queue',
  noAck: false,
  queueOptions: {
    durable: false
  },
},
```

Khi acknowledgments consumer thủ công được bật, chúng ta phải gửi một acknowledgment thích hợp từ worker để tín hiệu rằng chúng ta đã xong với một nhiệm vụ.

```typescript
@@filename()
@MessagePattern('notifications')
getNotifications(@Payload() data: number[], @Ctx() context: RmqContext) {
  const channel = context.getChannelRef();
  const originalMsg = context.getMessage();

  channel.ack(originalMsg);
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('notifications')
getNotifications(data, context) {
  const channel = context.getChannelRef();
  const originalMsg = context.getMessage();

  channel.ack(originalMsg);
}
```

#### Record builders

Để cấu hình các tùy chọn tin nhắn, bạn có thể sử dụng lớp `RmqRecordBuilder` (lưu ý: điều này có thể thực hiện cho các luồng event-based cũng vậy). Ví dụ, để đặt các thuộc tính `headers` và `priority`, sử dụng phương thức `setOptions`, như sau:

```typescript
const message = ':cat:';
const record = new RmqRecordBuilder(message)
  .setOptions({
    headers: {
      ['x-version']: '1.0.0',
    },
    priority: 3,
  })
  .build();

this.client.send('replace-emoji', record).subscribe(...);
```

> info **Gợi ý** Lớp `RmqRecordBuilder` được xuất từ gói `@nestjs/microservices`.

Và bạn có thể đọc các giá trị này ở phía server cũng vậy, bằng cách truy cập `RmqContext`, như sau:

```typescript
@@filename()
@MessagePattern('replace-emoji')
replaceEmoji(@Payload() data: string, @Ctx() context: RmqContext): string {
  const { properties: { headers } } = context.getMessage();
  return headers['x-version'] === '1.0.0' ? '🐱' : '🐈';
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('replace-emoji')
replaceEmoji(data, context) {
  const { properties: { headers } } = context.getMessage();
  return headers['x-version'] === '1.0.0' ? '🐱' : '🐈';
}
```

#### Cập nhật trạng thái thể hiện

Để nhận cập nhật thời gian thực về kết nối và trạng thái của thể hiện driver cơ bản, bạn có thể subscribe vào stream `status`. Stream này cung cấp các cập nhật trạng thái cụ thể cho driver được chọn. Đối với driver RMQ, stream `status` emit các sự kiện `connected` và `disconnected`.

```typescript
this.client.status.subscribe((status: RmqStatus) => {
  console.log(status);
});
```

> info **Gợi ý** Kiểu `RmqStatus` được import từ gói `@nestjs/microservices`.

Tương tự, bạn có thể subscribe vào stream `status` của server để nhận thông báo về trạng thái của server.

```typescript
const server = app.connectMicroservice<MicroserviceOptions>(...);
server.status.subscribe((status: RmqStatus) => {
  console.log(status);
});
```

#### Lắng nghe các sự kiện RabbitMQ

Trong một số trường hợp, bạn có thể muốn lắng nghe các sự kiện nội bộ được emit bởi microservice. Ví dụ, bạn có thể lắng nghe sự kiện `error` để kích hoạt các thao tác bổ sung khi xảy ra lỗi. Để làm điều này, sử dụng phương thức `on()`, như được hiển thị dưới đây:

```typescript
this.client.on('error', (err) => {
  console.error(err);
});
```

Tương tự, bạn có thể lắng nghe các sự kiện nội bộ của server:

```typescript
server.on<RmqEvents>('error', (err) => {
  console.error(err);
});
```

> info **Gợi ý** Kiểu `RmqEvents` được import từ gói `@nestjs/microservices`.

#### Truy cập driver cơ bản

Đối với các trường hợp sử dụng nâng cao hơn, bạn có thể cần truy cập thể hiện driver cơ bản. Điều này có thể hữu ích cho các kịch bản như đóng thủ công kết nối hoặc sử dụng các phương thức cụ thể của driver. Tuy nhiên, hãy nhớ rằng đối với hầu hết các trường hợp, bạn **không nên cần** truy cập trực tiếp driver.

Để làm điều này, bạn có thể sử dụng phương thức `unwrap()`, trả về thể hiện driver cơ bản. Tham số kiểu generic nên chỉ định loại thể hiện driver bạn mong đợi.

```typescript
const managerRef =
  this.client.unwrap<import('amqp-connection-manager').AmqpConnectionManager>();
```

Tương tự, bạn có thể truy cập thể hiện driver cơ bản của server:

```typescript
const managerRef =
  server.unwrap<import('amqp-connection-manager').AmqpConnectionManager>();
```

#### Wildcards

RabbitMQ hỗ trợ sử dụng wildcards trong các routing key để cho phép định tuyến tin nhắn linh hoạt. Wildcard `#` khớp với không hoặc nhiều từ, trong khi wildcard `*` khớp chính xác một từ.

Ví dụ, routing key `cats.#` khớp với `cats`, `cats.meow`, và `cats.meow.purr`. Routing key `cats.*` khớp với `cats.meow` nhưng không khớp với `cats.meow.purr`.

Để kích hoạt hỗ trợ wildcard trong microservice RabbitMQ của bạn, đặt tùy chọn cấu hình `wildcards` thành `true` trong đối tượng options:

```typescript
const app = await NestFactory.createMicroservice<MicroserviceOptions>(
  AppModule,
  {
    transport: Transport.RMQ,
    options: {
      urls: ['amqp://localhost:5672'],
      queue: 'cats_queue',
      wildcards: true,
    },
  },
);
```

Với cấu hình này, bạn có thể sử dụng wildcards trong các routing key của mình khi subscribe vào các sự kiện/tin nhắn. Ví dụ, để lắng nghe các tin nhắn với routing key `cats.#`, bạn có thể sử dụng mã sau:

```typescript
@MessagePattern('cats.#')
getCats(@Payload() data: { message: string }, @Ctx() context: RmqContext) {
  console.log(`Received message with routing key: ${context.getPattern()}`);

  return {
    message: 'Hello from the cats service!',
  }
}
```

Để gửi một tin nhắn với một routing key cụ thể, bạn có thể sử dụng phương thức `send()` của thể hiện `ClientProxy`:

```typescript
this.client.send('cats.meow', { message: 'Meow!' }).subscribe((response) => {
  console.log(response);
});
```